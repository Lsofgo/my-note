[toc]
# 1 使用技巧
## 1.1 常用的软件包
```
rzsz | 提供rz和sz命令；
```

## 1.2 修改主机名
```
vim /etc/HOSTNAME
hostnamectl set-hostname AAA
```

## 1.3 不能使用密码ssh到suse
```
把 no 改为 yes

vim /etc/ssh/sshd_config
        PasswordAuthentication no
        
systemctl restart sshd
```
## 1.4 用户管理
```
useradd -U zabbix

注意：
        如果不添加 -U 参数，suse会把zabbix的用户组分配给 gid=100的user组；
        -U 在创建用户时，创建和用同名的用户组；
        
        如果要生成 /home/zabbix目录，必须加 -m 选项；
        
userdel -r zabbix
        选项：  -r 同时删除 /home/zabbix 和 zabbix的邮箱账户；
```

## 1.5 systemctl 管理的服务的存放位置
```
/usr/lib/systemd/system

类似 service 和 /etc/init.d 的关系；

```


# 2 配置网络
## 2.1 配置IP
> vim /etc/sysconfig/network/ifcfg-eth0

```
BOOTPROTO='static'   #静态IP
BROADCAST='172.16.107.255'   #广播地址
IPADDR='172.16.107.201'   #IP地址
NETMASK='255.255.255.0'   #子网掩码
NETWORK='172.16.107.0'   #网络地址
STARTMODE='auto'    #开机启动网络
```

## 2.2 配置网关
> vim /etc/sysconfig/network/routes

```
default 172.16.107.2
```

## 2.3 配置DNS
> vim /etc/resolv.conf

在文件后追加：
```
nameserver 114.114.114.114
nameserver 8.8.8.8
```


## 2.4 重启网卡
> systemctl restart network <br>
/etc/init.d/network restart <br>
rcnetwork restart


# 3 网卡bond配置


# 4 zypper本地源配置
## 4.1 zypper本地源配置
```
mkdir /zypper

把suse光盘中的文件全部拷贝到 /zypper
    cp -r /run/media/root/SLE-12-SP3-Server-DVD-x86_640473/* /zypper/
    备注：
        /run/media/root/SLE-12-SP3-Server-DVD-x86_640473/ 是suse的自动挂载目录
        手动可以使用mount -o loop 命令挂载

添加 zypper 本地源
    zypper ar /zypper local

清理本地缓存
    zypper clean

刷新软件源
    zypper ref


zypper 命令参数：
    软件源管理：
	repos, lr		列出全部已定义的软件源
	addrepo, ar		添加一个新软件源
	removerepo, rr		移除指定软件源
	renamerepo, nr		重命名指定软件源
	refresh, ref		刷新全部软件源
	clean			清理本地缓存

    服务管理：
	services, ls		列出全部已定义服务
	addservice, as		添加一个新服务
	modifyservice, ms	修改指定服务
	removeservice, rs	移除指定服务
	refresh-services, refs	刷新全部服务

    软件管理：
	install, in		安装软件包
	remove, rm		移除软件包
	verify, ve		校验软件包的依赖关系完整性
	source-install, si	安装源代码包及其编译依赖
	install-new-recommends, inr
				安装已安装软件包推荐的新增软件包

    更新管理：
	update, up		用新版本更新已安装软件包
	list-updates, lu	列出可用更新
	patch			安装所需补丁
	list-patches, lp	列出所需补丁
	dist-upgrade, dup	执行发行版升级
	patch-check, pchk	检查补丁

    查询：
	search, se		搜索匹配一个模式的软件包
	info, if		显示指定软件包的完整信息
	patch-info		显示指定补丁的完整信息
	pattern-info		显示指定软件集的完整信息
	product-info		显示指定产品的完整信息
	patches, pch		列出全部可用补丁
	packages, pa		列出全部可用软件包
	patterns, pt		列出全部可用软件集
	products, pd		列出全部可用产品
	what-provides, wp	列出能够提供指定功能的软件包
```



















