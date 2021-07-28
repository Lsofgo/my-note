[toc]

# 1. Kubernetes

## 1.1 容器编排流程

1. 用户通过Yaml编排文件, 向服务端的Api Server发起请求, 需要建立新容器
2. 服务端Api Server会把新建立容器信息写入到Etcd
3. 服务端Scheduler会根据调度策略, 分配运行此容器的节点
4. 服务端Api Server会箭筒Scheduler哪些新的Pod的变化信息, 并更新状态到Etcd
5. 在服务端确定了哪些节点运行容器以后, 会调用节点的Kubelet服务, 通过Kubelt调用Docker拉取镜像, 启动镜像
6. 服务端Api Server也会箭筒Kubelet服务, 实时更新Pod状态到Etcd

## 1.2 核心组件介绍

Kubernetes 遵循 master-slace architecture(体系结构); Kubernets的组件可以分为管理单个的node组件和控制平面的一部分组件;

Kubernetes Master是集群的主要控制单元, 用户管理其工作负载, 并指导整个系统的通信;

Kubernetes控制平面由各自的进程组成, 每个组件都可以在单个主节点(Master)上运行, 也可以在支持high-availability cluster的多个主节点上运行;

**核心组件**

```
Master节点:
	etcd保存了整个集群的状态与节点网段分配;
						Master状态管理
	apiserver			提供了资源操作的唯一入口, 并提供认证、授权、访问控制、API注册和发现等机制;
						集群管理API、资源配额控制接口
	controller manager	负载维护集群的状态, 比如故障检测、自动扩展、滚动更新等;
						管理Node、pod副本、命名空间
	scheduler			负责资源的调度, 按照预定的调度策略将Pod调度到相应的机器上;
						将Pods分派到具体的Node运行
Node节点:
	kubelet				负责维护容器的生命周期, 同事也负责Volume(CVI)和网络(CNI)的管理;
						管理Pod、镜像、卷
	kube-proxy			负责为Service提供cluster内部的服务发现和负载均衡;
						网络代理、负载均衡、流量转发
```

**其他组件**

```
kube-dns				负责为整个集群提供DNS服务
Ingress Controller		为服务提供外网入口
Heapster				提供资源监控
Dashboard				提供GUI
Federation				提供跨可用区的集群
```



# 2. 部署Kubernetes

## 2.1 环境分配

| Kubernetes角色          | 分布节点 | 主机名                     | 虚拟机配置 | 内部DNS    | VIP            | 节点IP         |
| ----------------------- | -------- | -------------------------- | ---------- | ---------- | -------------- | -------------- |
| kube-apiserver          | master   | master-1/master-2/master-3 | 1C2G100G   | 10.0.0.0/8 | 192.168.0.0/24 | 192.168.0.0/24 |
| kube-controller-manager | master   | master-1/master-2/master-3 |            |            |                |                |
| kube-scheduler          | master   | master-1/master-2/master-3 |            |            |                |                |
| etcd                    | master   | master-1/master-2/master-3 |            |            |                |                |
| kubelet                 | node     | node-1/node-2              | 2C10G100G  |            |                |                |
| kubeproxy               | node     | node-1/node-2              |            |            |                |                |
| docker                  | node     | node-1/node-2              |            |            |                |                |
| flannel                 | node     | node-1/node-2              |            |            |                |                |

> 虚拟机快照说明

```
Clean				安装完成CentOS系统, 无任何组件
Kubernetes Basic	安装完成Kubernetes, 此阶段组件: CoreDNS, Dashboard
Ingress				安装完成Ingress
Moniter				安装完成Promitrus
Log					安装完成日志监控
```

## 2.2 安装部署

### 2.2.1 系统初始化

**初始化工具安装**

```
[root@master-1 ~]# yum install -y net-tools vim wget lrzsz git
```

**关闭防火墙**

**设置时区**

```
[root@master-1 ~]# cp -vf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

**关闭swap**

```
[root@master-1 ~]# swapoff -a

[root@master-1 ~]# sed -i '/ swap / s/^\(.*\)S/#\1/g' /etc/fstab
```

**时间同步**

```
[root@master-1 ~]# yum install -y ntp
```

**修改hosts**

```
cat <<EOF > /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.88.139 master-1
192.168.88.140 node-1
192.168.88.141 node-2
EOF
```

**主机互信**

```
[root@master-1 ~]# ssh-keygen

[root@master-1 ~]# cat .ssh/id_rsa.pub > .ssh/authorized_keys
[root@master-1 ~]# for i in node-1 node-2; do scp .ssh root@$i:/root/; done
```

**内核参数优化**

```
cat <<EOF >> /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
vm.swappiness=0
fs.file-max=52706963
fs.nr_open=52706963
EOF

# 应用内核参数
sysctl -p
```

### 2.2.2 安装keepalived

用于Master集群的统一入口

**公有云:** 可以在所有master节点前加层LB做四层负载

```
[root@master-1 ~]# yum install -y keepalived
```

```
[root@master-1 ~]# cp -v /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.20210517.bk
[root@master-1 ~]# cat <<EOF > /etc/keepalived/keepalived.conf
global_defs {
    router_id LVS_DEVEL
}
vrrp_script CheckMaster {
    script "curl -k https://192.168.88.250:6443"
    interval 3
    timeout 9
    fall 2
    rise 2
}
vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 61
    priority 100
    advert_int 1
    nopreempt
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.88.250/24 dev ens33
    }
    track_script {
        CheckMaster
    }
}
EOF
```

```
[root@master-2 ~]# cat <<EOF > /etc/keepalived/keepalived.conf
global_defs {
    router_id LVS_DEVEL
}
vrrp_script CheckMaster {
    script "curl -k https://192.168.88.250:6443"
    interval 3
    timeout 9
    fall 2
    rise 2
}
vrrp_instance VI_1 {
    state SLAVE
    interface ens33
    virtual_router_id 61
    priority 99
    advert_int 1
    nopreempt
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        192.168.88.250/24 dev ens33
    }
    track_script {
        CheckMaster
    }
}
EOF
```

```
[root@master-1 ~]# systemctl start keepalived
[root@master-1 ~]# systemctl enable keepalived
```

### 2.2.3 配置证书

| 组件           | 使用的证书                                 |
| -------------- | ------------------------------------------ |
| etcd           | ca.pem, server.pem, server-key.pem         |
| flannel        | ca.pem, server.pem, server-key.pem         |
| kube-apiserver | ca.pem, server.pem, server-key.pem         |
| kubelet        | ca.pem, ca-key.pem                         |
| kuber-proxy    | ca.pem, kube-proxy.pem, kube-proxy-key.pem |
| kubectl        | ca.pem, ca-key.pem                         |

**下载自签名证书工具**

```
[root@master-1 ~]# mkdir soft && cd soft
[root@master-1 soft]# wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
[root@master-1 soft]# wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
[root@master-1 soft]# wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64

[root@master-1 soft]# chmod a+x cfssl_linux-amd64 cfssljson_linux-amd64 cfssl-certinfo_linux-amd64

[root@master-1 soft]# mv cfssl_linux-amd64 /usr/local/bin/cfssl
[root@master-1 soft]# mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
[root@master-1 soft]# mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
```

**生成master证书**

```
#!/bin/bash
# certificate.sh
mkdir /root/ssl && cd /root/ssl
cat > ca-config.json <<EOF
{
    "signing": {
        "default": {
            "expiry": "87600h"
        },
        "profiles": {
            "kubernetes": {
                "expiry": "87600h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
EOF


# --------------------------------------------------
# 生成ca.pem ca-key.pem
# --------------------------------------------------
cat > ca-csr.json <<EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca -


# --------------------------------------------------
# 生成 server.pem server-key.pem
# 配置说明:
# 集群节点IP: "192.168.30.130", "192.168.30.131", "192.168.30.132"
# kubernetes服务ip: "10.1.0.1", 一般是 kue-apiserver 指定的 service-cluster-ip-range 网段的第一个IP
# 集群域名: "kubernetes*"
# --------------------------------------------------
cat > server-csr.json <<EOF
{
    "CN": "kubernetes",
    "hosts": [
        "127.0.0.1",
        "master-1",
        "node-1",
        "node-2",
        "192.168.88.139",
        "192.168.88.140",
        "192.168.88.141",
        "10.0.0.1",
        "kubernetes",
        "kubernetes.default",
        "kubernetes.default.svc",
        "kubernetes.default.svc.cluster",
        "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server


# --------------------------------------------------
# 生成admin.pem admin-key.pem
# 用于集群管路员访问管理集群
# --------------------------------------------------
cat > admin-csr.json <<EOF
{
    "CN": "admin",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
            "O": "system:master",
            "OU": "System"
        }
    ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin


# --------------------------------------------------
# 生成kube-proxy.pem kube-proxy-key-key.pem
# 用于集群管路员访问管理集群
# --------------------------------------------------
cat > kube-proxy-csr.json <<EOF
{
    "CN": "system:kube-proxy",
    "hosts": [],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy

# 删除无用的 .json .csr 文件
ls | grep -v pem | xargs -i rm -v {}
```

**分发证书**

```
[root@master-1 etcd-v3.4.13-linux-amd64]# mkdir -pv /opt/kubernetes/{bin,cfg,ssl,logs}
[root@master-1 etcd-v3.4.13-linux-amd64]# echo 'export PATH=$PATH:/opt/kubernetes/bin' >> /etc/profile
[root@master-1 etcd-v3.4.13-linux-amd64]# source /etc/profile

[root@master-1 etcd-v3.4.13-linux-amd64]# cd /root/ssl/
[root@master-1 ssl]# cp -v kube-proxy.pem kube-proxy-key.pem server.pem server-key.pem ca.pem ca-key.pem /opt/kubernetes/ssl/

[root@master-1 ssl]# for i in node-1 node-2; do scp -r /opt/kubernetes $i:/opt/; done
```





### 2.2.4 安装etcd

**etcd下载地址:** https://github.com/etcd-io/etcd/releases/download/v3.4.16/etcd-v3.4.16-linux-amd64.tar.gz

**安装etcd** (所有master)

```
[root@master-1 ~]# tar -xf etcd-v3.4.16-linux-amd64.tar.gz
[root@master-1 ~]# cd etcd-v3.4.16-linux-amd64

[root@master-1 etcd-v3.4.16-linux-amd64]# chmod u+x etcd etcdctl 
[root@master-1 etcd-v3.4.16-linux-amd64]# cp -v etcd etcdctl /opt/kubernetes/bin/
```

**编辑etcd配置文件** (所有master)

```
#!/bin/bash
# etcd.sh

ETCD_NAME=${1:-"master-1"}
ETCD_IP=${2:-"192.168.88.139"}
ETCD_CLUSTER=${3:-"master-1=https://192.168.88.139:2380,master-2=https://192.168.88.140:2380,master-3=https://192.168.88.141:2380"}

WORK_DIR=/opt/kubernetes

cat <<EOF >$WORK_DIR/cfg/etcd.conf
#[Member]
ETCD_NAME="${ETCD_NAME}"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://${ETCD_IP}:2380"
ETCD_LISTEN_CLIENT_URLS="https://${ETCD_IP}:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://${ETCD_IP}:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://${ETCD_IP}:2379"
ETCD_INITIAL_CLUSTER="${ETCD_CLUSTER}"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF

cat <<EOF > /usr/lib/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=${WORK_DIR}/cfg/etcd.conf
ExecStart=${WORK_DIR}/bin/etcd \\
--cert-file=${WORK_DIR}/ssl/server.pem \\
--key-file=${WORK_DIR}/ssl/server-key.pem \\
--peer-cert-file=${WORK_DIR}/ssl/server.pem \\
--peer-key-file=${WORK_DIR}/ssl/server-key.pem \\
--trusted-ca-file=${WORK_DIR}/ssl/ca.pem \\
--peer-trusted-ca-file=${WORK_DIR}/ssl/ca.pem
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable etcd
systemctl restart etcd
```

> etcd配置文件说明

```
ETCD_NAME							节点名称, 如果有多个节点, name每个节点要修改为本节点的名称;
ETCD_DATA_DIR						数据目录;
ETCD_LISTEN_PEER_URLS				集群通信监听地址;
ETCD_LISTEN_CLIENT_URLS				客户端访问监听地址;
ETCD_INITIAL_ADVERTISE_PEER_URLS	集群通告地址;
ETCD_ADVERTISE_CLIENT_URLS			客户端通告地址;
ETCD_INITIAL_CLUSTER				集群节点地址, 如果有多个节点, 那么用逗号分隔;
ETCD_INITIAL_CLUSTER_TOKEN			集群token;
ETCD_INITIAL_CLUSTER_STATE			计入集群的当前状态, new是新集群, existing表示加入已有集群;
```

#### 查看etcd集群状态

````
[root@master-1 ~]# export ETCDCTL_API=2
[root@master-1 ~]# etcdctl \
--ca-file=/opt/kubernetes/ssl/ca.pem \
--cert-file=/opt/kubernetes/ssl/server.pem \
--key-file=/opt/kubernetes/ssl/server-key.pem \
--endpoints="https://192.168.88.139:2379,https://192.168.88.140:2379,https://192.168.88.141:2379" \
cluster-health
````

```
[root@master-1 ~]# ETCDCTL_API=3
[root@master-1 ~]# etcdctl \
--cacert=/opt/kubernetes/ssl/ca.pem \
--cert=/opt/kubernetes/ssl/server.pem \
--key=/opt/kubernetes/ssl/server-key.pem \
--endpoints="https://192.168.88.139:2379,https://192.168.88.140:2379,https://192.168.88.141:2379" \
--write-out=table \
endpoint status 
```

**向etcd中写入集群Pod网段信息**

172.17.0.0/16 为 Kubernetes Pod 的 IP 地址段

网段必须与 kube-controller-manager 的 --cluster-cidr 参数一致

v2 接口有set , v3接口set换为put

flannel使用etcd的v2接口, 需要使用etcd的v2接口写入数据, v3接口写入的读取不到;

```
[root@master-1 ~]# export ETCDCTL_API=2
[root@master-1 ~]# etcdctl \
--ca-file=/opt/kubernetes/ssl/ca.pem \
--cert-file=/opt/kubernetes/ssl/server.pem \
--key-file=/opt/kubernetes/ssl/server-key.pem \
--endpoints="https://192.168.88.134:2379,https://192.168.88.135:2379,https://192.168.88.136:2379" \
set /coreos.com/network/config \
'{"Network": "172.17.0.0/16", "Backend": {"Type": "vxlan"}}'
```

查看写入的信息

```
[root@master-1 ~]# etcdctl \
--ca-file=/opt/kubernetes/ssl/ca.pem \
--cert-file=/opt/kubernetes/ssl/server.pem \
--key-file=/opt/kubernetes/ssl/server-key.pem \
--endpoints="https://192.168.88.134:2379,https://192.168.88.135:2379,https://192.168.88.136:2379" \
get /coreos.com/network/config
```



### 2.2.5 安装Docker

node节点安装

```
[root@localhost ~]# wget https://mirrors.aliyun.com/repo/Centos-7.repo -P /etc/yum.repos.d/
[root@localhost ~]# wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -P /etc/yum.repos.d/

[root@localhost ~]# yum clean all
[root@localhost ~]# yum install -y docker-ce
```

**配置阿里云镜像加速器**

```
[root@localhost ~]# mkdir -p /etc/docker
tee /etc/docker/daemon.json <<-'EOF'
{
"registry-mirrors": ["https://rfj1yucr.mirror.aliyuncs.com"]
}
EOF

[root@localhost ~]# systemctl daemon-reload
[root@localhost ~]# systemctl enable docker
[root@localhost ~]# systemctl restart docker
```

### 2.2.6 安装Flannel

所有节点安装

**Flanned下载地址**: https://github.com/flannel-io/flannel/releases/download/v0.13.0/flannel-v0.13.0-linux-amd64.tar.gz

**flanned要是用etcd v2接口通信**, 在etcd启动命令中增加 --enable-v2

```
[root@master-1 ~]# mkdir flannel
[root@master-1 ~]# mv flannel-v0.13.0-linux-amd64.tar.gz flannel
[root@master-1 ~]# cd flannel/

[root@master-1 flannel]# tar -xf flannel-v0.13.0-linux-amd64.tar.gz 
[root@master-1 flannel]# mv flanneld mk-docker-opts.sh /opt/kubernetes/bin/

[root@master-1 flannel]# cd /opt/kubernetes/bin/

# 分发flanned到所有机器
[root@master-1 bin]# for i in master-2 master-3 node-1 node-2; do scp flanneld mk-docker-opts.sh $i:/opt/kubernetes/bin/; done
```

**配置Flannel**

```
#!/bin/bash
# flanneld.sh

ETCD_ENDPOINTS=${1:-"https://192.168.88.134:2379,https://192.168.88.135:2379,https://192.168.88.136:2379"}

cat <<EOF >/opt/kubernetes/cfg/flanneld.conf
FLANNEL_OPTIONS="--etcd-endpoints=${ETCD_ENDPOINTS} \\
-etcd-cafile=/opt/kubernetes/ssl/ca.pem \\
-etcd-certfile=/opt/kubernetes/ssl/server.pem \\
-etcd-keyfile=/opt/kubernetes/ssl/server-key.pem"
EOF

cat <<EOF >/usr/lib/systemd/system/flanneld.service
[Unit]
Description=Flanneld overlay address etcd agent
After=network-online.target network.target
Before=docker.service

[Service]
Type=notify
EnvironmentFile=/opt/kubernetes/cfg/flanneld.conf
ExecStart=/opt/kubernetes/bin/flanneld --ip-masq \$FLANNEL_OPTIONS
ExecStartPost=/opt/kubernetes/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/subnet.env
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# 配置docker使用flannel子网：
cp /usr/lib/systemd/system/docker.service /tmp/docker.service.20210518.bk

cat <<EOF > /usr/lib/systemd/system/docker.service

[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/run/flannel/subnet.env
ExecStart=/usr/bin/dockerd \$DOCKER_NETWORK_OPTIONS
ExecReload=/bin/kill -s HUP \$MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
EOF
```

> 启动脚本说明

mk-docker-opts.sh 脚本将分配给flannel的Pod子网网段信息写入 /run/flannel/docker文件, 后续docker启动时, 使用这个文件中的环境变量配置 docker0 网桥;

flanneld 使用系统缺省路由所在的接口与其他节点通信, 对于有多个网络接口(如内网和公网的节点), 可以用 -iface 参数指定通信接口, 比如 eth0;

**master启动flannel; node节点重启docker, 启动flannel**

```
systemctl daemon-reload
systemctl enable flanneld
systemctl restart flanneld
systemctl restart docker
```

配置完成, docker0 和 flannel.1 地址同网段

```
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:28:54:56:9d brd ff:ff:ff:ff:ff:ff
    inet 172.17.101.1/24 brd 172.17.101.255 scope global docker0
       valid_lft forever preferred_lft forever
4: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default 
    link/ether aa:f6:ce:c8:75:54 brd ff:ff:ff:ff:ff:ff
    inet 172.17.101.0/32 brd 172.17.101.0 scope global flannel.1
       valid_lft forever preferred_lft forever
    inet6 fe80::a8f6:ceff:fec8:7554/64 scope link 
       valid_lft forever preferred_lft forever
```

### 2.2.7 安装 master 节点

```
[root@master-1 ~]# tar -xf kubernetes-server-linux-amd64.tar.gz
[root@master-1 ~]# cd kubernetes/server/bin

[root@master-1 bin]# cp -v kube-scheduler kube-apiserver kube-controller-manager kubectl /opt/kubernetes/bin/
```

**分发 kubernetes server 文件到其他 master**

```
[root@master-1 bin]# for i in master-2 master-3; do scp /opt/kubernetes/bin/kube* $i:/opt/kubernetes/bin/ ; done
```

**创建TLS Bootstrapping Token**

```
export BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
cat <<EOF > /opt/kubernetes/cfg/token.csv
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF

# 分发到其他 master 节点
for i in master-2 master-3; do scp /opt/kubernetes/cfg/token.csv $i:/opt/kubernetes/cfg; done
```

#### 安装 kube-apiserver

**创建 apiserver 配置文件和启动文件** 所有master节点

```
#!/bin/bash
# apiservice.sh

MASTER_IP=192.168.88.139
ETCD_SERVERS=${2:-"https://192.168.88.139:2379,https://192.168.88.140:2379,https://192.168.88.141:2379"}

cat > /opt/kubernetes/cfg/kube-apiserver.conf << EOF
KUBE_APISERVER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--etcd-servers=${ETCD_SERVERS} \\
--bind-address=${MASTER_IP} \\
--secure-port=6443 \\
--advertise-address=${MASTER_IP} \\
--allow-privileged=true \\
--service-cluster-ip-range=10.0.0.0/24 \\
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \\
--authorization-mode=RBAC,Node \\
--enable-bootstrap-token-auth=true \\
--token-auth-file=/opt/kubernetes/cfg/token.csv \\
--service-node-port-range=30000-32767 \\
--kubelet-client-certificate=/opt/kubernetes/ssl/server.pem \\
--kubelet-client-key=/opt/kubernetes/ssl/server-key.pem \\
--tls-cert-file=/opt/kubernetes/ssl/server.pem \\
--tls-private-key-file=/opt/kubernetes/ssl/server-key.pem \\
--client-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--etcd-cafile=/opt/kubernetes/ssl/ca.pem \\
--etcd-certfile=/opt/kubernetes/ssl/server.pem \\
--etcd-keyfile=/opt/kubernetes/ssl/server-key.pem \\
--audit-log-maxage=30 \\
--audit-log-maxbackup=3 \\
--audit-log-maxsize=100 \\
--audit-log-path=/opt/kubernetes/logs/k8s-audit.log"
EOF

cat <<EOF >/usr/lib/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=-/opt/kubernetes/cfg/kube-apiserver.conf
ExecStart=/opt/kubernetes/bin/kube-apiserver \$KUBE_APISERVER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable kube-apiserver
systemctl restart kube-apiserver
```

> 参数说明

```
--logtostderr					启用日志
--v								日志等级
--etcd-servers					etcd集群地址
--bind-address					监听地址
--secure-port					安全端口
--advetise-address				集群通告地址
--allow-privileged				启用授权
--service-cluster-ip-range		虚拟IP地址段
--enable-admission-plugins		准入控制模块
--authorization-mode			认证授权, 启用RBAC授权
--enable-bootstrap-token-auth	启用 TLS bootstrap 功能
--token-auth-file				tooken文件
--server-node-port-range		Server Node类型默认分配端口范围
```

**授权 kubelet-bootstrap 用户允许请求**

```
[root@master-1 ~]# kubectl create clusterrolebinding kubelet-bootstrap \
--clusterrole=system:node-bootstrapper \
--user=kubelet-bootstrap

clusterrolebinding.rbac.authorization.k8s.io/kubelet-bootstrap created
```

#### 安装 kube-controller-manager

**创建 kube-controller-manager 配置文件和启动文件** 所有master节点

```
cat > /opt/kubernetes/cfg/kube-controller-manager.conf << EOF
KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--leader-elect=true \\
--master=127.0.0.1:8080 \\
--bind-address=127.0.0.1 \\
--allocate-node-cidrs=true \\
--cluster-cidr=10.244.0.0/16 \\
--service-cluster-ip-range=10.0.0.0/24 \\
--cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \\
--cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--root-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--experimental-cluster-signing-duration=87600h0m0s"
EOF

cat > /usr/lib/systemd/system/kube-controller-manager.service << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-controller-manager.conf
ExecStart=/opt/kubernetes/bin/kube-controller-manager\$KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start kube-controller-manager
systemctl enable kube-controller-manager
```

参数说明

```
--master			通过本地非安全本地端口 8080 连接 apiserver。
-–leader-elect		当该组件启动多个时，自动选举（HA）
–-cluster-signing-cert-file
-–cluster-signing-key-file		自动为 kubelet 颁发证书的 CA，与 apiserver 保持一致
```

#### 安装kube-scheduler

````
cat > /opt/kubernetes/cfg/kube-scheduler.conf << EOF
KUBE_SCHEDULER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--leader-elect \
--master=127.0.0.1:8080 \
--bind-address=127.0.0.1"
EOF

cat > /usr/lib/systemd/system/kube-scheduler.service << EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-scheduler.conf
ExecStart=/opt/kubernetes/bin/kube-scheduler \$KUBE_SCHEDULER_OPTS
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start kube-scheduler
systemctl enable kube-scheduler
````

参数说明

```
–-master			通过本地非安全本地端口 8080 连接 apiserver
-–leader-elect		当该组件启动多个时，自动选举（HA）
```

#### 查看集群状态

```
[root@master-1 ~]# kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"}
```



### 2.2.8 安装 node 节点

```
[root@node-1 ~]# tar -xf kubernetes-node-linux-amd64-v1.19.10.tar.gz
[root@node-1 ~]# cd kubernetes/node/bin/

[root@node-1 bin]# cp -v kubelet kube-proxy /opt/kubernetes/bin/
[root@node-1 bin]# scp kubelet kube-proxy node-2:/opt/kubernetes/bin/
```

**生成 bootstrap.kubeconfig 和 kube-proxy.kubeconfig 文件**

master节点

```
APISERVER_IP=${1:-"192.168.88.139"}
KUBE_APISERVER="https://${APISERVER_IP}:6443" # apiserver IP:PORT
TOKEN="5d520bc77968dc9c1fb4796b277a6bee" # 与 token.csv 里保持一致

# 生成 kubelet bootstrap.kubeconfig 配置文件
kubectl config set-cluster kubernetes \
--certificate-authority=/opt/kubernetes/ssl/ca.pem \
--embed-certs=true \
--server=${KUBE_APISERVER} \
--kubeconfig=bootstrap.kubeconfig
kubectl config set-credentials "kubelet-bootstrap" \
--token=${TOKEN} \
--kubeconfig=bootstrap.kubeconfig
kubectl config set-context default \
--cluster=kubernetes \
--user="kubelet-bootstrap" \
--kubeconfig=bootstrap.kubeconfig
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig

# 生成 kubelet kube-proxy.kubeconfig 配置文件
kubectl config set-cluster kubernetes \
    --certificate-authority=/opt/kubernetes/ssl/ca.pem \
    --embed-certs=true \
    --server=${KUBE_APISERVER} \
    --kubeconfig=kube-proxy.kubeconfig

kubectl config set-credentials kube-proxy \
    --client-certificate=/opt/kubernetes/ssl/kube-proxy.pem \
    --client-key=/opt/kubernetes/ssl/kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes \
    --user=kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```



分发到node节点

```
[root@master-1 ~]# for i in node-1 node-2; do scp bootstrap.kubeconfig kube-proxy.kubeconfig $i:/opt/kubernetes/cfg/ ; done
```

#### 安装 kubelet

**生成配置文件**

```
NODE_NAME="node-1"
NODE_ADDRESS=${1:-"192.168.88.140"}
DNS_SERVER_IP=${2:-"10.0.0.2"}

cat > /opt/kubernetes/cfg/kubelet.conf << EOF
KUBELET_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--hostname-override=${NODE_NAME} \\
--network-plugin=cni \\
--kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \\
--bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \\
--config=/opt/kubernetes/cfg/kubelet-config.yml \\
--cert-dir=/opt/kubernetes/ssl \\
--pod-infra-container-image=lizhenliang/pause-amd64:3.0"
EOF

cat > /opt/kubernetes/cfg/kubelet-config.yml << EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: ${NODE_ADDRESS}
port: 10250
readOnlyPort: 10255
cgroupDriver: cgroupfs
clusterDNS:
- ${DNS_SERVER_IP}
clusterDomain: cluster.local
failSwapOn: false
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /opt/kubernetes/ssl/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
maxOpenFiles: 1000000
maxPods: 110
EOF

cat > /usr/lib/systemd/system/kubelet.service << EOF
[Unit]
Description=Kubernetes Kubelet
After=docker.service
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kubelet.conf
ExecStart=/opt/kubernetes/bin/kubelet \$KUBELET_OPTS
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF


systemctl daemon-reload
systemctl start kubelet
systemctl enable kubelet
```

参数说明

```
kubelet.config
    -–hostname-override：显示名称，集群中唯一
    -–network-plugin：启用 CNI
    -–kubeconfig：空路径，会自动生成，后面用于连接 apiserver
    -–bootstrap-kubeconfig：首次启动向 apiserver 申请证书
    -–config：配置参数文件
    -–cert-dir：kubelet 证书生成目录
    -–pod-infra-container-image：管理 Pod 网络容器的镜像

bootstrap.kubeconfig
	certificate-authority-data		/opt/kubernetes/ssl/ca.pem
	token							token.csv 第一段
```

批准 kubelet 证书申请并加入集群

```
# 查看 kubelet 证书请求
[root@master-1 ~]# kubectl get csr
NAME                                                   AGE     SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-HT3FmGf2JZbT95_n67HWWBbYoPwh8whNJfcFrjlQxCg   86s     kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
node-csr-q6W-nU6ytHBsyb-YpcQTUAjp0bXxn5klcKfkFprAuJk   2m16s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending

# 批准申请
[root@master-1 ~]# kubectl certificate approve node-csr-HT3FmGf2JZbT95_n67HWWBbYoPwh8whNJfcFrjlQxCg
certificatesigningrequest.certificates.k8s.io/node-csr-HT3FmGf2JZbT95_n67HWWBbYoPwh8whNJfcFrjlQxCg approved
[root@master-1 ~]# kubectl certificate approve node-csr-q6W-nU6ytHBsyb-YpcQTUAjp0bXxn5klcKfkFprAuJk
certificatesigningrequest.certificates.k8s.io/node-csr-q6W-nU6ytHBsyb-YpcQTUAjp0bXxn5klcKfkFprAuJk approved

# 查看node节点
[root@master-1 ~]# kubectl get nodes
NAME             STATUS     ROLES    AGE   VERSION
192.168.88.140   NotReady   <none>   28s   v1.19.10
192.168.88.141   NotReady   <none>   50s   v1.19.10
```



#### 安装 kube-proxy

```
NODE_NAME="node-1"

cat > /opt/kubernetes/cfg/kube-proxy.conf << EOF
KUBE_PROXY_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--config=/opt/kubernetes/cfg/kube-proxy-config.yml"
EOF

cat > /opt/kubernetes/cfg/kube-proxy-config.yml << EOF
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
metricsBindAddress: 0.0.0.0:10249
clientConnection:
  kubeconfig: /opt/kubernetes/cfg/kube-proxy.kubeconfig
hostnameOverride: ${NODE_NAME}
clusterCIDR: 10.0.0.0/24
mode: ipvs
ipvs:
  scheduler: "rr"
iptables:
  masqueradeAll: true
EOF

cat > /usr/lib/systemd/system/kube-proxy.service << EOF
[Unit]
Description=Kubernetes Proxy
After=network.target
[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-proxy.conf
ExecStart=/opt/kubernetes/bin/kube-proxy \$KUBE_PROXY_OPTS
Restart=on-failure
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target
EOF


systemctl daemon-reload
systemctl start kube-proxy
systemctl enable kube-proxy
```

#### 安装 CNI 网络

下载地址: https://github.com/containernetworking/plugins/releases/download/v0.8.6/cni-plugins-linux-amd64-v0.8.6.tgz

```
[root@node-1 ~]# mkdir -pv /opt/cni/bin
[root@node-1 ~]# tar -xf cni-plugins-linux-amd64-v0.8.6.tgz -C /opt/cni/bin/
```

```
[root@master-1 ~]# wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

[root@master-1 ~]# kubectl apply -f kube-flannel.yml 
# 查看node状态, status 已经是 Ready
[root@master-1 ~]# kubectl get pods -n kube-system
```

#### 授权 apiserver 访问 kubelet

```
cat > apiserver-to-kubelet-rbac.yaml<< EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
      - pods/log
    verbs:
      - "*"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

```
[root@master-1 ~]# kubectl apply -f apiserver-to-kubelet-rbac.yaml
```



### 2.2.9 安装CoreDNS

```
wget https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/coredns/coredns.yaml.base
```









