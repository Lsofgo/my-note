### 安装etcd

**etcd下载地址:** https://github.com/etcd-io/etcd/releases/download/v3.4.13/etcd-v3.4.13-linux-amd64.tar.gz

**安装etcd** (所有master)

```
[root@master-1 ~]# tar -xf etcd-v3.4.13-linux-amd64.tar.gz
[root@master-1 ~]# cd etcd-v3.4.13-linux-amd64

[root@master-1 etcd-v3.4.13-linux-amd64]# mkdir -pv /opt/kubernetes/{bin,cfg,ssl}
[root@master-1 etcd-v3.4.13-linux-amd64]# echo 'export PATH=$PATH:/opt/kubernetes/bin' >> /etc/profile
[root@master-1 etcd-v3.4.13-linux-amd64]# source /etc/profile

[root@master-1 etcd-v3.4.13-linux-amd64]# chmod u+x etcd etcdctl 
[root@master-1 etcd-v3.4.13-linux-amd64]# cp -v etcd etcdctl /opt/kubernetes/bin/
```

**编辑etcd配置文件** (所有master)

```
[root@master-1 etcd-v3.4.13-linux-amd64]# cd /root/ssl/
[root@master-1 ssl]# cp -v server.pem server-key.pem ca.pem /opt/kubernetes/ssl/
```

```
#!/bin/bash
# etcd.sh

ETCD_NAME=${1:-"master-1"}
ETCD_IP=${2:-"192.168.88.134"}
ETCD_CLUSTER=${3:-"master-1=https://192.168.88.134:2380,master-2=https://192.168.88.135:2380,master-3=https://192.168.88.136:2380"}

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
--peer-trusted-ca-file=${WORK_DIR}/ssl/ca.pem \\
--enable-v2
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

> 启动文件说明

```
--enable-v2		启动v2版本接口
```



#### 查看etcd集群状态

````
[root@master-1 ~]# etcdctl \
--cacert=/opt/kubernetes/ssl/ca.pem \
--cert=/opt/kubernetes/ssl/server.pem \
--key=/opt/kubernetes/ssl/server-key.pem \
--endpoints="https://192.168.88.134:2379,https://192.168.88.135:2379,https://192.168.88.136:2379" \
--write-out=table \
endpoint status 
````

**向etcd中写入集群Pod网段信息**

172.17.0.0/16 为 Kubernetes Pod 的 IP 地址段

网段必须与 kube-controller-manager 的 --cluster-cidr 参数一致

```
[root@master-1 ~]# etcdctl \
--cacert=/opt/kubernetes/ssl/ca.pem \
--cert=/opt/kubernetes/ssl/server.pem \
--key=/opt/kubernetes/ssl/server-key.pem \
--endpoints="https://192.168.88.134:2379,https://192.168.88.135:2379,https://192.168.88.136:2379" \
set /coreos.com/network/config \
'{"Network": "172.17.0.0/16", "Backend": {"Type": "vxlan"}}'
```

查看写入的信息

```
[root@master-1 ~]# etcdctl \
--cacert=/opt/kubernetes/ssl/ca.pem \
--cert=/opt/kubernetes/ssl/server.pem \
--key=/opt/kubernetes/ssl/server-key.pem \
--endpoints="https://192.168.88.134:2379,https://192.168.88.135:2379,https://192.168.88.136:2379" \
get /coreos.com/network/config
```

#### 在3.4版本中使用v2接口

```
[root@master-1 ~]# echo 'export ETCDCTL_API=2' > /etc/profile
[root@master-1 ~]# source /etc/profile
```

