### 查看/关闭防火墙

```shell
ufw status		# c
ufw disable		# 关闭防火墙
```

### 配置静态IP

```shell
root@localhost:/etc/netplan# vim 00-installer-config.yaml
network:
  ethernets:
    ens33:     							#配置的网卡的名称
      addresses: [192.168.31.181/24]    #配置的静态ip地址和掩码
      dhcp4: no    						#关闭DHCP，如果需要打开DHCP则写yes
      optional: true
      gateway4: 192.168.31.2    		#网关地址
      nameservers:
         addresses: [114.114.114.114,8.8.8.8]    #DNS服务器地址
  version: 2
  renderer: networkd    #指定后端采用systemd-networkd或者Network Manager，可不填写则默认使用systemd-workd
```

### 初始化

```shell
apt install lrzsz
```

