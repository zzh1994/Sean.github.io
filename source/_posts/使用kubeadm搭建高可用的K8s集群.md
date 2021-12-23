---
title: 使用kubeadm搭建高可用的K8s集群
author: Sean
top: true
date: 2021-12-23 21:09:10
summary:
img:
cover:
coverImg:
tags:
- K8S
- Docker
- 脚本
categories:
- docker
password:
---

## 1. 安装要求

在开始之前，部署Kubernetes集群机器需要满足以下几个条件：

- 一台或多台机器，操作系统 CentOS7.x-86_x64
- 硬件配置：2GB或更多RAM，2个CPU或更多CPU，硬盘30GB或更多
- 可以访问外网，需要拉取镜像，如果服务器不能上网，需要提前下载镜像并导入节点
- 禁止swap分区

## 2. 准备环境

| 角色          | IP            |
| ------------- | ------------- |
| master1       | 100.100.1.221 |
| master2       | 100.100.1.222 |
| node1         | 100.100.1.223 |
| VIP（虚拟ip） | 100.100.1.35  |

```
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭selinux
sed -i 's/enforcing/disabled/' /etc/selinux/config  # 永久
setenforce 0  # 临时

# 关闭swap
swapoff -a  # 临时
sed -ri 's/.*swap.*/#&/' /etc/fstab    # 永久

# 根据规划设置主机名
hostnamectl set-hostname <hostname>

# 在master添加hosts
cat >> /etc/hosts << EOF
100.100.1.35  master.k8s.io   vip
100.100.1.221    master01.k8s.io wz0
100.100.1.222    master02.k8s.io wz1
100.100.1.223    node01.k8s.io   wz2
EOF

# 将桥接的IPv4流量传递到iptables的链
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system  # 生效

# 时间同步
yum install ntpdate -y
ntpdate time.windows.com
```



## 3. 所有master节点部署keepalived

### 3.1 安装相关包和keepalived

```
yum install -y conntrack-tools libseccomp libtool-ltdl

yum install -y keepalived
```

### 3.2配置master节点

master1节点配置

```
cat > /etc/keepalived/keepalived.conf <<EOF 
! Configuration File for keepalived

global_defs {
   router_id k8s
}

vrrp_script check_haproxy {
    script "killall -0 haproxy"
    interval 3
    weight -2
    fall 10
    rise 2
}

vrrp_instance VI_1 {
    state MASTER 
    interface ens224
    virtual_router_id 51
    priority 250
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass ceb1b3ec013d66163d6ab
    }
    virtual_ipaddress {
        100.100.1.35
    }
    track_script {
        check_haproxy
    }

}
EOF
```

master2节点配置

```
cat > /etc/keepalived/keepalived.conf <<EOF 
! Configuration File for keepalived

global_defs {
   router_id k8s
}

vrrp_script check_haproxy {
    script "killall -0 haproxy"
    interval 3
    weight -2
    fall 10
    rise 2
}

vrrp_instance VI_1 {
    state BACKUP 
    interface ens224 
    virtual_router_id 51
    priority 200
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass ceb1b3ec013d66163d6ab
    }
    virtual_ipaddress {
        100.100.1.35
    }
    track_script {
        check_haproxy
    }

}
EOF
```

### 3.3 启动和检查

在两台master节点都执行

```
# 启动keepalived
$ systemctl start keepalived.service
设置开机启动
$ systemctl enable keepalived.service
# 查看启动状态
$ systemctl status keepalived.service
```

启动后查看master1的网卡信息

```
ip a s ens224
```



## 4. 部署haproxy

### 4.1 安装

```
yum install -y haproxy
```

### 4.2 配置

两台master节点的配置均相同，配置中声明了后端代理的两个master节点服务器，指定了haproxy运行的端口为16443等，因此16443端口为集群的入口

```
cat > /etc/haproxy/haproxy.cfg << EOF
#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2
    
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon 
       
    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats
#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------  
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000
#---------------------------------------------------------------------
# kubernetes apiserver frontend which proxys to the backends
#--------------------------------------------------------------------- 
frontend kubernetes-apiserver
    mode                 tcp
    bind                 *:16443
    option               tcplog
    default_backend      kubernetes-apiserver    
#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
backend kubernetes-apiserver
    mode        tcp
    balance     roundrobin
    server      master01.k8s.io   100.100.1.221:16443 check
    server      master02.k8s.io   100.100.1.222:16443 check
#---------------------------------------------------------------------
# collection haproxy statistics message
#---------------------------------------------------------------------
listen stats
    bind                 *:1080
    stats auth           admin:awesomePassword
    stats refresh        5s
    stats realm          HAProxy\ Statistics
    stats uri            /admin?stats
EOF
```

### 4.3 启动和检查

两台master都启动

```
# 设置开机启动
$ systemctl enable haproxy
# 开启haproxy
$ systemctl start haproxy
# 查看启动状态
$ systemctl status haproxy
```

检查端口

```
netstat -lntup|grep haproxy
```



## 5. 所有节点安装Docker/kubeadm/kubelet

### 5.1 安装Docker

参考：https://blog.csdn.net/qq_38332722/article/details/108705821



### 5.2 安装kubeadm，kubelet和kubectl

由于版本更新频繁，这里指定版本号部署：

```
$ yum install -y kubelet-1.18.20 kubeadm-1.18.20 kubectl-1.18.20
$ systemctl enable kubelet
```



## 6. 部署Kubernetes Master

### 6.1 创建kubeadm配置文件

在具有vip的master上操作，这里为master1

```
$ mkdir /usr/local/kubernetes/manifests -p

$ cd /usr/local/kubernetes/manifests/

$ vi kubeadm-config.yaml

apiServer:
  certSANs:
    - wz0
    - wz1
    - master.k8s.io
    - 100.100.1.35
    - 100.100.1.221
    - 100.100.1.222
    - 127.0.0.1
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta1
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: "master.k8s.io:16443"
controllerManager: {}
dns: 
  type: CoreDNS
etcd:
  local:    
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.18.20
networking: 
  dnsDomain: cluster.local  
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.1.0.0/16
scheduler: {}
```



### 6.2 在master1节点执行

```
$ kubeadm init --config kubeadm-config.yaml
```



按照提示配置环境变量，使用kubectl工具：

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
$ kubectl get nodes
$ kubectl get pods -n kube-system
```



**按照提示保存以下内容，一会要使用：**

```bash
kubeadm join master.k8s.io:16443 --token y7tvvv.ur4q2wnkdml8f64b \
    --discovery-token-ca-cert-hash sha256:d393a6bbf0e61126dbd0dd3809988c768fef63c711da0ad31ea4144c889fec6b \
    --control-plane 
```

查看集群状态

```bash
kubectl get cs

kubectl get pods -n kube-system
```



## 7.安装集群网络

从官方地址获取到flannel的yaml，在master1上执行

```bash
mkdir flannel
cd flannel
wget -c https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

无法下载，从网上下载kube-flannel.yml 文件执行。
```



安装flannel网络

```bash
kubectl apply -f kube-flannel.yml 
```

检查

```bash
kubectl get pods -n kube-system
```



## 8、master2节点加入集群

### 8.1 复制密钥及相关文件

从master1复制密钥及相关文件到master2

```bash
# ssh root@100.100.1.222 mkdir -p /etc/kubernetes/pki/etcd

# scp /etc/kubernetes/admin.conf root@100.100.1.223:/etc/kubernetes
   
# scp /etc/kubernetes/pki/{ca.*,sa.*,front-proxy-ca.*} root@100.100.1.223:/etc/kubernetes/pki
   
# 
```

### 8.2 master2加入集群

执行在master1上init后输出的join命令,需要带上参数`--control-plane`表示把master控制节点加入集群

```
kubeadm join master.k8s.io:16443 --token vkfjut.zrxy4fvqo3htwi0i --discovery-token-ca-cert-hash sha256:5a0a50d655377ed9b1e8a784b06192bb0d15e224b29110a007fb60a9b1bde22a --control-plane
```

检查状态

```
kubectl get node

kubectl get pods --all-namespaces
```

## 

## 5. 加入Kubernetes Node

在node1上执行

向集群添加新节点，执行在kubeadm init输出的kubeadm join命令：

```
kubeadm join master.k8s.io:16443 --token ckf7bs.30576l0okocepg8b     --discovery-token-ca-cert-hash sha256:19afac8b11182f61073e254fb57b9f19ab4d798b70501036fc69ebef46094aba
```

**集群网络重新安装，因为添加了新的node节点**

/usr/local/kubernetes/manifests/flannel路径下，kube-flannel.yml

kubectl delete -f   kube-flannel.yml

kubectl apply -f   kube-flannel.yml

检查状态

```
kubectl get node

kubectl get pods --all-namespaces
```

## 

## 7. 测试kubernetes集群

在Kubernetes集群中创建一个pod，验证是否正常运行：

```
$ kubectl create deployment nginx --image=nginx
$ kubectl expose deployment nginx --port=80 --target-port=80 --type=NodePort
$ kubectl get pod,svc
```

访问地址：http://NodeIP:Port  集群任意ip+给定端口
