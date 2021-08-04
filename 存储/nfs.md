### 安装

```shell
yum install -y nfs

mkdir -pv /data/sharedir
echo '/data/sharedir *(rw,no_root_squash,sync)' >> /etc/exports
```

### 启动

```shell
systemctl start nfs-server
systemctl enable nfs-server
```

### 查看共享信息

```shell
exportfs -v
```

### 查看共享目录被挂载信息

```shell
showmount -e
```

