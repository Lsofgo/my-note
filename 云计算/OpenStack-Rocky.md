[toc]

> [参考文档：https://docs.openstack.org/mitaka/zh_CN/install-guide-rdo/](https://docs.openstack.org/mitaka/zh_CN/install-guide-rdo/)
# 1 安装结构说明

# 1.1 版本和硬件配置

- 操作系统：CentOS-7.4.1708-minimal
- OpenStack版本：juno
- 关闭防火墙和SELinux

节点 | CPU | 内存 | 硬盘 | 网卡
---- | --- | ---- | ---- | ----
Controller | 2核 | 1.5GB | 100GB | 1块网卡
Compute | Max | Max | 100GB | 2块网卡
Network | 2核 | 1.5GB | 100GB | 3块网卡
Block | 2核 | 1GB | 100GB | 1块网卡

## 1.2 网络拓扑

- M: Management Network 管理网络 192.168.136.0(NAT)
- I: Instance Tunnels Network 实例网络，虚拟机和虚拟机之间通信网络 192.168.192.0(Host Only)
- E: External Network 外部网络，由Neturn代理到内部网络 192.168.111.0(Host Only)

节点IP说明：

- Controller
  - M: 192.168.136.11
- Network
  - M: 192.168.136.12
  - I: 192.168.192.11
  - E: 192.168.111.12
- Compute
  - M: 192.168.136.13
  - I: 192.168.192.12
- Block
  - M: 192.168.136.14

# 2 安装前准备
## 2.1 VMWare网络配置
添加三个虚拟网络；
分别用于模拟管理网络，实例网络，以及外部网络；
管理网络使用NAT模式，方便软件部署；其余使用Host Only模式；

## 2.2 服务器配置
### 2.2.1 域名解析

配置hosts或者DNS

```
# 追加到所有节点
cat <<EOF >> /etc/hosts
192.168.136.11 controller.alec.com controller
192.168.136.12 neutorn.alec.com neutron
192.168.136.13 compute.alec.com compute
192.168.136.14 block.alec.com block
EOF
```

### 2.2.2 yum源配置
```
# CentOS 源
wget -O /etc/yum.repos.d/Centos-7.repo http://mirrors.aliyun.com/repo/Centos-7.repo

# epel 源
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo

# openstack源
cat <<EOF >> /etc/yum.repos.d/OpenStack-Train.repo
[OpenStack]
name=Queen
baseurl=https://mirrors.aliyun.com/centos/7/cloud/x86_64/openstack-train/
enabled=1
gpgcheck=0
EOF
```

### 2.2.3 NTP时间同步

没有外网的情况，可以把controller配置为本地时间服务器

```
yum install -y ntp

# 替换server配置
vim +21 /etc/ntp.conf
server ntp1.aliyun.com iburst
server ntp2.aliyun.com iburst
server ntp3.aliyun.com iburst
server ntp4.aliyun.com iburst
server ntp5.aliyun.com iburst

systemctl start ntpd
systemctl enable ntpd
```

### 2.2.4 防火墙
```
systemctl stop firewalld
systemctl disable firewalld

setenforce 0
sed -i '/^SELINUX=/ s/enforcing/disabled/i' /etc/selinux/config
```

### 2.2.5 安装OpenStack预备包

```
# 安装 yum-plugin-priorities 包，防止高优先级软件被低优先级软件覆盖
yum install -y yum-plugin-priorities

# 更新操作系统
yum upgrade -y

# 安装 openstack-selinux 自动管理selinux
yum install -y openstack-selinux

rm -vf /etc/yum.repos.d/CentOS-*
```

### 2.2.6 安装Memcached

```
yum install -y memcached python-memcached

systemctl start memcached.service
systemctl enable memcached.service
```

### 2.2.7 安装MariaDB

```
yum install -y mariadb-server MySQL-python

vim /etc/my.cnf.d/mariadb-server.cnf
[mysqld]
......
bind-address = 192.168.136.11
default-storage-engine = innodb
innodb_file_per_table = 1
character_set_server=utf8
collation-server = utf8_general_ci

systemctl start mariadb
systemctl enable mariadb

# 设置密码
MariaDB [(none)]> set password = password('admin');
```

### 2.2.8 安装RabbitMQ

- 功能：协调操作和状态信息服务
- 常用软件：
  - RabbitMQ
  - Opid
  - ZeroMQ
- 安装rabbitmq-server

```
yum install -y rabbitmq-server

systemctl start rabbitmq-server
systemctl enable rabbitmq-server

# 添加rabbitmq用户，默认的guest用户只能通过127.0.0.1访问；
# 第一个alec是账号，第二个alec是密码
rabbitmqctl add_user alec alec
# 给alec账号赋予最高权限
rabbitmqctl set_user_tags alec administrator
# 查看rabbitmq账户
rabbitmqctl list_users
# 添加权限
rabbitmqctl set_permissions -p '/' alec '.*' '.*' '.*'
```



# 3 Keystone - 认证服务

## 3.1 Keystone说明

- Keystone是OpenStack Identity Service的项目名称，是一个负责身份管理与授权的组件；
- 主要功能：
  - 实现用户的身份认证；
  - 基于角色的权限管理；
  - openstack其他组件的访问地址和安全策略管理；
- 主要目的是给整个openstack的各个组件(nova, cinder, glance...)提供一个统一的验证方式；

## 3.2 Kenstone功能

- 用户管理
  - Account 账户
  - Authentication 身份认证
  - Authorization 授权
- 服务目录管理

## 3.3 名词解释

- **User(用户)**

  一个人、系统或服务在OpenStack中的数字表示；已经登录的用户分配令牌以访问资源；用户可以直接分配个特定的租户，就像隶属于每个组

- **Credentials(凭证)**

  用于确认用户身份数据；例如：用户名和密码，用户名和API key，或由认证服务提供的身份验证令牌；

- **Authentication(验证)**

  确认用户身份的过程

- **Token(令牌)**

  一个用于访问OpenStack API和资源的字母数字字符串；一个令牌可以随时撤销，并且持续一段时间有效；

- **Tenant(租户)**

  一个组织或孤立资源的容器；租户个可以组织或隔离认证对象；根据服务运营的要求，一个租户可以映射到客户、账户、组织或项目；

- **Service(服务)**

  OpenStack服务，例如计算服务(Nova)，对象存储服务(swift)，或镜像服务(glance)；它提供了一个或多个端点，供用户访问资源和执行操作；

- **Endpoint(端点)**

  一个用户访问某个服务的可以通过网络进行访问的地址，通常是一个URL地址；

- **Role(角色)**

  定制化的包含特定用户权限和特权的集合；

- **Keystone Client(keystone命令行工具)**

  Keystone的命令行工具；通过该工具可以创建用户、角色、服务和端点等；

##  3.4 部署 Keystone

### 3.4.1 创建Keystone数据库

```
mysql -uroot -padmin

create database keystone;

grant all privileges on keystone.* to 'keystone'@'localhost' identified by 'keystone';
grant all privileges on keystone.* to 'keystone'@'%' identified by 'keystone';

flush privileges;
```

### 3.4.2 在初始配置生成一个随机值作为管理员令牌

```
[root@controller ~]# openssl rand -hex 10;
7b152382d84315c02cf0
```

### 3.4.3 安装配置Keystone

```
yum install -y openstack-keystone python2-openstackclient http mod_wsgi

vim /etc/keystone/keystone.conf
# 添加管理员令牌
admin_token = 7b152382d84315c02cf0
# 修改数据库配置
[database]
connection = mysql+pymysql://keystone:keystone@controller/keystone
#                             用户      密码
[token]
provider = fernet
```

### 3.4.3 初始化keystone数据库

```
su -s /bin/sh -c "keystone-manage db_sync" keystone
```
### 3.4.4 初始化Fernet keys

```
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
```

> 在Queen版本前，keystone需要运行在两个不同的端口上，用来运行identify v2 API，除5000端口外，还需要35357上运行一个单独的管理服务；移除v2 API后，keystone只需要一个5000端口就可以运行所有API

```
keystone-manage bootstrap --bootstrap-password admin \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne
```

> 这一步，会自动生成admin域、项目、用户、角色、服务实体、API端点

### 3.4.5 配置apache托管keystone

```
ln -sv /usr/share/keystone/wsgi-keystone.conf /etc/httpd/conf.d/

vim +95 /etc/httpd/conf/httpd.conf
ServerName controller

systemctl start httpd
systemctl enable httpd
```

### 3.4.6 通过设置环境变量来配置管理账户

```
export OS_USERNAME=admin
export OS_PASSWORD=admin
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
```

### 3.4.7 创建域、项目、用户和角色

```
# 创建 example 域(default域已经存在，这是只是创建域的一个方法)
openstack domain create --description "An Example Domain" example
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | An Example Domain                |
| enabled     | True                             |
| id          | 94f16414c32249b29cbe19dea06009ef |
| name        | example                          |
| tags        | []                               |
+-------------+----------------------------------+
```

> 创建一个的service项目，OpenStack的组件会关联到这个项目中

```
# 在 default 域中创建 service 项目
openstack project create --domain default --description "Service Project" service
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 059e13ceff1f4bfc9b9a0a3466d573bf |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+
```

> 创建一个普通项目、用户、角色

```
# 创建 demo 项目
openstack project create --domain default --description "Demo Project" demo
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | default                          |
| enabled     | True                             |
| id          | ff5f94d5a8714249a42a7b4f6bc1587b |
| is_domain   | False                            |
| name        | demo                             |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+


# 创建 demo 用户
openstack user create --domain default --password-prompt demo
User Password:demo
Repeat User Password:demo
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | bfda144c255847d3a9209ea282764c3b |
| name                | demo                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+

# 创建 user 角色
openstack role create user
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | 5eb3696fba5f47f0b2ed1c1c60c9ff5c |
| name      | user                             |
+-----------+----------------------------------+

# 添加 user 角色到 demo 项目和用户
openstack role add --project demo --user demo user
```

## 3.5 验证服务

> 取消 `OS_AUTH_URL` `OS_URL`

```
unset OS_AUTH_URL OS_PASSWORD
```

> 作为 `admin` 用户，请求认证令牌

```
openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name default --os-user-domain-name default \
  --os-project-name admin --os-username admin token issue
  Password: admin
+------------+--------------------------------------------------------------------------------
| Field      | Value
+------------+--------------------------------------------------------------------------------
| expires    | 2020-07-17T09:08:29+0000
| id         | gAAAAABfEwS6GJWppKNjn0YDWJunrn_hQJmCCiuvCXt3KVgnVBiQefZ6CqXDozGxA0ChYn1XQ-20rP0Fz6O7sqvI_HnfReWcpgQTbgvo0SsawBR6QmUSlyCKp9yOe7fcNU9m4Dt1OEAQ_hN6WNm_1leHJ-pdPmem29lPtMha-DHiMifzuJKNL8k
| project_id | c6f8d8041d5c4f128c4d6c489156b875
| user_id    | 5d7d7304b8b44434a6e942fe94728c4a
```

> 作为``demo`` 用户，请求认证令牌

```
openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name default --os-user-domain-name default \
  --os-project-name demo --os-username demo token issue
Password: demo
+------------+-------------------------------------------------------------
| Field      | Value
+------------+-------------------------------------------------------------
| expires    | 2020-07-17T09:11:52+0000
| id         | gAAAAABfEwT5BOP7z_jXLp2ZfkPXHBLtxfCJIKKSisboBvnWtF9qll4xb26xlERvXwe3nW7AN2rtpoS93i24KWgwUHDhQsfPxb7NbXEbFHeSsy3rpebJ_FSqb0jerzFWo2TR9f6KQgqNldUl8DbjapK7hMdAChIAPcyB6mcR8SiHugC7ven-ONQ
| project_id | ff5f94d5a8714249a42a7b4f6bc1587b
| user_id    | bfda144c255847d3a9209ea282764c3b
```

## 3.6 创建OpenStack 客户端环境脚本

### 3.6.1 创建脚本

> 创建admin-openrc

```
cat <<EOF >> admin-openrc
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=admin
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOF
```

> 创建demo-openrc

```
cat <<EOF > demo-openrc
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=demo
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
EOF
```

### 3.6.2 使用脚本

> 加载``admin-openrc``文件来身份认证服务的环境变量位置和``admin``项目和用户证书

```
source admin-openrc
```

> 请求认证令牌

```
openstack token issue
+------------+--------------------------------------------------------------------------
| Field      | Value
+------------+--------------------------------------------------------------------------
| expires    | 2020-07-17T09:31:14+0000
| id         | gAAAAABfEwVPiOYCR6nRZUPBGDMozhzg9UoLCy4mXOY-ww9CpWSK7Wi8Pw3mrFxhogh9zupK4pa4aERerFsYAq6-P3qxvJhROz1D6RQzWmNOCJPWe6BAe_XYW1oEUN-Q97veTqNetY0kH0UlHCRHutmVYZPW6Lb3n0PNSEtCPUgi5g8iAIF5vJY
| project_id | c6f8d8041d5c4f128c4d6c489156b875
| user_id    | 5d7d7304b8b44434a6e942fe94728c4a
```

# 4 Glance - 镜像服务

## 4.1 Glance说明

### 4.1.1 Glance服务功能

- OpenStack镜像服务(Glance)使用户能够发现、注册并检索虚拟机镜像(.img文件)；
- 它提供了一个 `REST API` 接口，使用户可以查询虚拟机镜像源数据和检索一个实际的镜像文件；
- 不论是简单的文件系统还是 OpenStack 对象存储，你都可以通过镜像服务在不同位置存储虚拟镜像
- 默认情况下，上传的虚拟机镜像存储路径为 `/var/lib/glance/images/`

### 4.1.2 组件说明

- **glance-api**

  一个用来接收镜像、发现、检索和存储的API接口；

- **glance-registry**

  用来存储、处理和检索镜像的元数据；

  元数据包换对象的大小和类型；

  glance-registry是一个OpenStack镜像服务使用的内部服务，不要透露给用户；

- **DataBase**

  用户存储镜像的元数据的大小、类型，支持大多数数据库，一般选择`MySQL`或`SQLite`；
  
- **Storage repository for image files**
  
  镜像文件的存储仓库；
  
  支持包括普通文件系统在内的各种存储类型；
  
  包括对象存储、块设备、HTTP、Amazon S3，但有些存储只支持只读访问；

- **Image Identifiers**
  
  `Image URL`，格式`<Glance Server Location>/images/<ID>`；
  
  全局唯一；
  
- **Image Status**
  
  - `Queued ` 镜像ID已被保留，镜像还没有上传
  - `Saving ` 镜像正在被上传
  - `Active` 镜像可以使用
  - `Killed` 镜像损坏或者不可用
  - `Deleted` 镜像被删除
  
- **Disk Format**
  
  - `Raw` This si unstructured disk image format
  - `Vhd` VMare、XEN、Microsoft、VirtualBox
  - `Vmdk` common format
  - `Vdi` VirtualBox、QEMU emulator
  - `ISO` optical disc
  - `Qcow2` QEMU emulator
  - `Aki` Amazon Kernel Image
  - `Ari` Amazon RamDisk Image
  - `Ami` Amazon Machine Image
  
- **Container Format**
  
  - `Bare`
  - `Ovf`
  - `Aki`
  - `Ami`
  - `Ari`
  

## 4.2 部署 Glance

### 4.2.1 创建Clance数据库

```
mysql -uroot -padmin

CREATE DATABASE glance;

GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'glance';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'glance';

FLUSH PRIVILEGES;
```

### 4.2.2 创建Glance用户

> 加载 `admin` 凭证，来获取管理员命令的执行权限

```
source admin-openrc
```

> 创建 `glance` 用户

```
openstack user create --domain default --password-prompt glance
User Password:glance
Repeat User Password:glance
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 4f080a83b50d4a118cf8912f2caff239 |
| name                | glance                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

> 给 `glance `用户分配 `admin` 角色，并加入到 `service` 项目

```
openstack role add --project service --user glance admin
```

> 创建``glance``服务

```
openstack service create --name glance --description "OpenStack Image" image
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| enabled     | True                             |
| id          | 926ff884210448eba132cb949a974d95 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+
```

> 创建`glance` API 端点

```
openstack endpoint create --region RegionOne image public http://controller:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | ea975e2fa8c04cbeafe3ad31fc838202 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 926ff884210448eba132cb949a974d95 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+

openstack endpoint create --region RegionOne image internal http://controller:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 0d0fb6f4838c461f89fd465018aef4a4 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 926ff884210448eba132cb949a974d95 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+

openstack endpoint create --region RegionOne image admin http://controller:9292
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 0b924b6956b84495bc92f28ad58791f3 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 926ff884210448eba132cb949a974d95 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+
```

### 4.2.3 安装配置Glance

```
# 安装Glance
yum install -y openstack-glance

# 配置Glance
vim /etc/glance/glance-api.conf
```

> 配置数据库连接

```
[database]
...
connection = mysql+pymysql://glance:glance@controller.alec.com/glance
#                            用户    密码
```

> 配置认证服务访问

```
[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = glance
...
[paste_deploy]
flavor = keystone
```

> 配置本地文件系统存储和镜像文件位置

```
[glance_store] 
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
```

### 4.2.4 初始化数据库

```
su -s /bin/sh -c "glance-manage db_sync" glance
```

### 4.2.5 启动服务

```
systemctl start openstack-glance-api
systemctl enable openstack-glance-api
```

## 4.3 验证服务

> 获取admin凭证执行admin命令

```
source admin-openrc
```

```
dd if=/dev/zero of=test.img bs=1M count=10

openstack image create "test" --file test.img --disk-format qcow2 --container-format bare --public
```



> 下载镜像

```
wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
```

> 使用 `qcow2`磁盘格式、`bare`容器格式 上传到glance服务，并设置为public；让所有项目都可以访问；

```
openstack image create "cirros" \
  --file cirros-0.3.4-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --public
+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | ee1eca47dc88f4879d8a229cc70a07c6                     |
| container_format | bare                                                 |
| created_at       | 2020-07-18T15:21:52Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/2ab0cd11-0aa3-4735-a85f-2c348de0a89d/file |
| id               | 2ab0cd11-0aa3-4735-a85f-2c348de0a89d                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | cirros                                               |
| owner            | c6f8d8041d5c4f128c4d6c489156b875                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 13287936                                             |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2020-07-18T15:21:53Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+
```

> 查看上传后的镜像信息

```
openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 2ab0cd11-0aa3-4735-a85f-2c348de0a89d | cirros | active |
+--------------------------------------+--------+--------+
```

# 5 Placement

## 5.1 说明（略）

Queen版本之后，Nova不连接这个，起不来；

## 5.2 部署Placement(Queen,)

### 5.2.1 创建Placement用户

> 加载 `admin` 凭证，来获取管理员命令的执行权限

```
source admin-openrc
```

> 创建`placement`用户

```
openstack user create --domain default --password-prompt placement
User Password:placement
Repeat User Password:placement
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | c66b45246ff54d539033869ade06be74 |
| name                | placement                        |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

>  给 `placement`用户分配 `admin` 角色，并加入到 `service` 项目

```
openstack role add --project service --user placement admin
```

> 创建 `placement`服务

```
openstack service create --name placement --description "Placement API" placement
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Placement API                    |
| enabled     | True                             |
| id          | e74dcfc92bb14d8ca73d67b16f716060 |
| name        | placement                        |
| type        | placement                        |
+-------------+----------------------------------+
```

> 创建 `placement`API 端点

```
openstack endpoint create --region RegionOne placement public http://controller:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 74722a8921ba41e7bf84f924dd7c407e |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | e74dcfc92bb14d8ca73d67b16f716060 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+

openstack endpoint create --region RegionOne placement internal http://controller:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 5c4d97c24db749179eeb4e0da964307b |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | e74dcfc92bb14d8ca73d67b16f716060 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+

openstack endpoint create --region RegionOne placement admin http://controller:8778
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 0afc2baea05d433488d20d5b31374bcd |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | e74dcfc92bb14d8ca73d67b16f716060 |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+
```

### 5.2.2 安装配置Placement

```
yum install -y openstack-nova-placement-api
```

### 5.2.3 添加权限

```
vim /etc/httpd/conf.d/00-nova-placement-api.conf 
# 追加
<Directory /usr/bin>
   <IfVersion >= 2.4>
      Require all granted
   </IfVersion>
   <IfVersion < 2.4>
      Order allow,deny
      Allow from all
   </IfVersion>
</Directory>
```

## 5.3 启动服务

```
systemctl restart httpd
```

## 5.4 部署Placement（>Queen）







# 6 Nova - 计算服务

## 6.1 Nava说明

### 6.1.1 Nava是啥

OpenStack 是由 Rackspace 和 NASA 共同开发的云计算平台

类似 Amazon EC2 和 S3 的云基础架构服务

Nava 在 OpenStack 中提供云计算服务

### 6.1.2 组件说明 

- **API**

  - **nova-api service**

    接收并相应终端用户计算API调用；

    该服务支持 OpenStack 计算 API，Amazon EC2 和特殊的管理特权 API；

  - **nova-api-metadata service**

    接受从实例元数据发来的请求；

    该服务通常与 nova-network 服务在安装多主机模式下运行；

- **Core**

  - **nova-compute service**

    一个守护进程，通过虚拟化层 API 接口创建和终止虚拟机实例；

    例如：XenAPI for XenServer/XCP， libvirt for KVM or QEMU， VMwareAPI for VMware；

  - **nova-scheduler service**

    从队列中获取虚拟机请求实例，并确认由哪台计算机运行该虚拟机；

    负责虚拟机创建时候的，宿主机负载判断；

  - **nova-conductor module**

    协调 nova-compute 服务和 database 之间的交互数据；

    避免 nova-compute 服务直接访问云数据库；

    不要将该模块部署在 nova-compute 运行的节点上；

- **Networking**

  - **nova-network worker daemon**

    类似于 nova-conpute 服务，接受来自队列的网络任务和操控网络；

    比如这只网卡桥接或改变iptables规则；

  - **nova-consoleauth daemon**

    在控制台代理提供用户授权令牌；

  - **nova-novncproxy daemon**

    提供了一个通过VNC连接来访问运行的虚拟机实例的代理；

    支持基于浏览器的 novnc 客户端；

  - **nova-spicehtml5proxy daemon**

    提供了一个通过spice连接老访问运行的虚拟机实例的代理；

    支持基于浏览器的 HTML5 客户端；

  - **nova-xvpnvncproxy daemon**

    提供了一个通过VNC连接来访问运行的虚拟机实例的代理；

    支持 OpenStack-Specific Java客户端；

  - **nova-cert daemon**

    x509 证书

- **Othor**

  - **nova-objectstore daemon**

    一个 Amazon S3 的接口，用于将 Amazon S3 的镜像注册到 OpenStack euca2ools client 用于兼容 Amazon E2 接口的命令行工具；

  - **nova client**

    nova 命令行工具；

  - **The queue**

    在进程之间传递消息的队列；
  
    通常使用RabbitMQ；
  
  - **SQL database**
  
    保存云计算基础设置，建立和运行时的状态信息；
  

## 6.2 部署 Nova Controller

**在Controller节点部署；**

### 6.2.1 创建 Nova Controller 数据库

```
mysql -uroot -padmin

CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;

GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'nova';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'nova';

GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'nova';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'nova';

GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'nova';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'nova';

FLUSH PRIVILEGES;
```

### 6.2.2 创建Nova Controller用户

> 加载 `admin` 凭证，来获取管理员命令的执行权限

```
source admin-openrc
```

> 创建 `nova` 用户

```
openstack user create --domain default --password-prompt nova
User Password:nova
Repeat User Password:nova
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | c373a827b3f243f7a7e00ff172170cb1 |
| name                | nova                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

> 给 `nova `用户分配 `admin` 角色，并加入到 `service` 项目

```
openstack role add --project service --user nova admin
```

> 创建 `nova` 服务

```
openstack service create --name nova --description "OpenStack Compute" compute
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | 4319f9d4c8b34fc09a066de1171d0c1e |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+
```

> 创建 `nova` API 端点

```
openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 73777313e28a48758b50d4e279c0bb83 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 4319f9d4c8b34fc09a066de1171d0c1e |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://controller:8774/v2.1      |
+--------------+----------------------------------+

openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 9b5e6398e7ff4d92aa81e48e5201a574 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 4319f9d4c8b34fc09a066de1171d0c1e |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://controller:8774/v2.1      |
+--------------+----------------------------------+

openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | b1c1874e043b491ca87f98bbd103e2b2 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 4319f9d4c8b34fc09a066de1171d0c1e |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://controller:8774/v2.1      |
+--------------+----------------------------------+
```

### 6.2.3 安装配置 Nova Controller

```
yum install -y openstack-nova-api openstack-nova-conductor openstack-nova-novncproxy openstack-nova-scheduler
  
# 配置nova
vim /etc/nova/nova.conf
```

> 配置 compute 和 metadata  APIs

```
[DEFAULT]
enabled_apis=osapi_compute,metadata
```

> 配置数据连接

```
[api_database]
connection = mysql+pymysql://nova:nova@controller.alec.com/nova_api

[database]
connection = mysql+pymysql://nova:nova@controller.alec.com/nova
```

> 配置RabbitMQ (如果RabbitMQ和Nova Controller不在同一节点，不能使用RabbitMQ的guest用户)

```
[DEFAULT]
transport_url = rabbit://alec:alec@controller:5672/
```

> 配置认证服务访问

```
[api]
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova
```

> 配置管理IP

```
[DEFAULT]
my_ip=192.168.136.11
```

> 配置Neutron (装好Neutron后再配置，后面再说)

```
[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron
```

> 配置vnc代理

```
[vnc]
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip
```

> 配置Glance API 

```
[glance]
api_servers = http://controller:9292
```

> 配置锁路径

```
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
```



> 配置Placement(略过，只做记录)

```
[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = placement
```

### 6.2.4 初始化数据库

```
# 初始化 nava_api 数据库
su -s /bin/sh -c "nova-manage api_db sync" nova

# 注册 cell0 数据库
su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova

# 创建 cell1 单元
su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
fb8e991a-8c1b-4b73-9802-3fb125cf6335

# 初始化 nava 数据库
su -s /bin/sh -c "nova-manage db sync" nova

# 验证 cell0 和 cell1 是否正确注册
su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova
+-------+--------------------------------------+-------------------------------------+----------------------------------------------------------+
|  名称 |                 UUID                 |            Transport URL            |                        数据库连接                        |
+-------+--------------------------------------+-------------------------------------+----------------------------------------------------------+
| cell0 | 00000000-0000-0000-0000-000000000000 |                none:/               | mysql+pymysql://nova:****@controller.alec.com/nova_cell0 |
| cell1 | fb8e991a-8c1b-4b73-9802-3fb125cf6335 | rabbit://alec:****@controller:5672/ |    mysql+pymysql://nova:****@controller.alec.com/nova    |
+-------+--------------------------------------+-------------------------------------+----------------------------------------------------------+
```

### 6.2.5 启动服务

```
systemctl start openstack-nova-api openstack-nova-scheduler openstack-nova-conductor openstack-nova-novncproxy

systemctl enable openstack-nova-api openstack-nova-scheduler openstack-nova-conductor openstack-nova-novncproxy
```

## 6.3 部署 Nova Compute

**在Compute节点部署**

### 6.3.1 安装配置Nova Compute

```
yum install -y openstack-nova-compute

# 解决qemu-kvm-rhev依赖，在/etc/yum.repos.d/Centos-7.repo1追加virt源
# 会安装qemu-kvm-ev
[virt]
name=centosvirt
baseurl=https://mirrors.aliyun.com/centos/$releasever/virt/$basearch/kvm-common/
gpgcheck=0
enabled=1

vim /etc/nova/nova.conf
```

> 配置 compute 和 metadata  APIs

```
[DEFAULT]
enabled_apis=osapi_compute,metadata
```

> 配置RabbitMQ (如果RabbitMQ和Nova Controller不在同一节点，不能使用RabbitMQ的guest用户)

```
[DEFAULT]
transport_url = rabbit://alec:alec@controller:5672/
```

> 配置认证服务访问

```
[api]
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova
```

> 配置管理IP (配置为compute节点的管理网络IP)

```
[DEFAULT]
my_ip=192.168.136.13
```

> 配置Neutron (装好Neutron后再配置，后面再说)

```
[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron
```

> 配置vnc代理

```
[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html
```

> 配置Glance服务

```
[glance]
api_servers = http://controller:9292
```

> 配置 lock path

```
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
```

> 配置Placement(略过，只做记录，用到再说)

```
[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = placement
```

> 配置虚拟类型

```
# 确定conpute节点是否支持硬件加速
egrep -c '(vmx|svm)' /proc/cpuinfo
# 如果命令返回 1 或者 greater 可以略过这个配置；
# 如果返回 0，说明不支持硬件加速，需要配置libvirtd使用 qemu 而不是 kvm；

[libvirt]
virt_type = qemu
```

### 6.3.2 启动服务

```
systemctl start libvirtd openstack-nova-compute

systemctl enable libvirtd openstack-nova-compute
```

## 6.4 添加计算节点到单元数据库中（controller节点执行）

> 加载`admin`凭证

```
source admin-openrc 
```

> 确认数据库中计算节点的主机

```
openstack compute service list
```

> 发现计算节点主机

```
su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova

Found 2 cell mappings.
Skipping cell0 since it does not contain hosts.
Getting computes from cell 'cell1': fb8e991a-8c1b-4b73-9802-3fb125cf6335
Found 0 unmapped computes in cell: fb8e991a-8c1b-4b73-9802-3fb125cf6335
```

> 查看节点加入信息

```
openstack compute service list
+----+----------------+------------+----------+---------+-------+----------------------------+
| ID | Binary         | Host       | Zone     | Status  | State | Updated At                 |
+----+----------------+------------+----------+---------+-------+----------------------------+
|  4 | nova-scheduler | controller | internal | enabled | up    | 2020-07-18T18:08:30.000000 |
|  5 | nova-conductor | controller | internal | enabled | up    | 2020-07-18T18:08:33.000000 |
|  6 | nova-compute   | compute    | nova     | enabled | up    | 2020-07-18T18:08:25.000000 |
+----+----------------+------------+----------+---------+-------+----------------------------+
```

## 6.5 服务验证

> 加载`admin`凭证

```
source admin-openrc 
```

> 查看节点信息

```
openstack compute service list
+----+----------------+------------+----------+---------+-------+----------------------------+
| ID | Binary         | Host       | Zone     | Status  | State | Updated At                 |
+----+----------------+------------+----------+---------+-------+----------------------------+
|  4 | nova-scheduler | controller | internal | enabled | up    | 2020-07-18T18:08:30.000000 |
|  5 | nova-conductor | controller | internal | enabled | up    | 2020-07-18T18:08:33.000000 |
|  6 | nova-compute   | compute    | nova     | enabled | up    | 2020-07-18T18:08:25.000000 |
+----+----------------+------------+----------+---------+-------+----------------------------+
```

> 在认证服务中列出所有的API端点

```
openstack catalog list
+-----------+-----------+-----------------------------------------+
| Name      | Type      | Endpoints                               |
+-----------+-----------+-----------------------------------------+
| nova      | compute   | RegionOne                               |
|           |           |   public: http://controller:8774/v2.1   |
|           |           | RegionOne                               |
|           |           |   internal: http://controller:8774/v2.1 |
|           |           | RegionOne                               |
|           |           |   admin: http://controller:8774/v2.1    |
|           |           |                                         |
| keystone  | identity  | RegionOne                               |
|           |           |   public: http://controller:5000/v3/    |
|           |           | RegionOne                               |
|           |           |   internal: http://controller:5000/v3/  |
|           |           | RegionOne                               |
|           |           |   admin: http://controller:5000/v3/     |
|           |           |                                         |
| glance    | image     | RegionOne                               |
|           |           |   admin: http://controller:9292         |
|           |           | RegionOne                               |
|           |           |   internal: http://controller:9292      |
|           |           | RegionOne                               |
|           |           |   public: http://controller:9292        |
|           |           |                                         |
| placement | placement | RegionOne                               |
|           |           |   admin: http://controller:8778         |
|           |           | RegionOne                               |
|           |           |   internal: http://controller:8778      |
|           |           | RegionOne                               |
|           |           |   public: http://controller:8778        |
|           |           |                                         |
+-----------+-----------+-----------------------------------------+
```

> 检查 cell 和 Placement API 是否正常

```
nova-status upgrade check
+-------------------------------+
| 升级检查结果                  |
+-------------------------------+
| 检查: Cells v2                |
| 结果: 成功                    |
| 详情: None                    |
+-------------------------------+
| 检查: Placement API           |
| 结果: 成功                    |
| 详情: None                    |
+-------------------------------+
| 检查: Resource Providers      |
| 结果: 成功                    |
| 详情: None                    |
+-------------------------------+
| 检查: Ironic Flavor Migration |
| 结果: 成功                    |
| 详情: None                    |
+-------------------------------+
| 检查: API Service Version     |
| 结果: 成功                    |
| 详情: None                    |
+-------------------------------+
```

# 7 Neutorn - 网络服务

## 7.1 Neutorn说明(略)

不废话了

## 7.2 部署Neutorn(Crontroller节点)

**在 controller节点部署；**

### 7.2.1 创建Neutron数据库

```
mysql -uroot -padmin

CREATE DATABASE neutron;

GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' IDENTIFIED BY 'neutron';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'neutron';

FLUSH PRIVILEGES;
```

### 7.2.2 创建Neutron用户

```
source admin-openrc
```

> 创建 `neutron `用户

```
openstack user create --domain default --password-prompt neutron
User Password:neutron
Repeat User Password:neutron
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | b5fb09e2e84a43308a860fb732d72c84 |
| name                | neutron                          |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

> 给 `neutron `用户分配 `admin` 角色，并加入到 `service` 项目

```
openstack role add --project service --user neutron admin
```

> 创建 `neutron `服务

```
openstack service create --name neutron --description "OpenStack Networking" network
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | 0f45506dab5c4917a7678bd57d997483 |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+
```

> 创建 `neutron `API 端点

```
openstack endpoint create --region RegionOne network public http://controller:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 5c3908dc3b1a40008b5f395e6542699d |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 0f45506dab5c4917a7678bd57d997483 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+

openstack endpoint create --region RegionOne network internal http://controller:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | e432464cd5614537a98223b2b369746e |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 0f45506dab5c4917a7678bd57d997483 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+

openstack endpoint create --region RegionOne network admin http://controller:9696
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | a3ed7b3e88ad41f8b75b2bcb43c31ae7 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 0f45506dab5c4917a7678bd57d997483 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+
```

### 7.2.3 安装配置Neutron

Neutron 有两种网络选项：

- 公共网络：简单，但是只支持实例连接到公有网络(外部网络)；没有私有网络（个人网络），路由器以及浮动IP地址；只有``admin``或者其他特权用户才可以管理公有网络；
- 私有网络：在 `公共网络` 的基础上多了layer－3服务，支持实例连接到私有网络。`demo`或者其他没有特权的用户可以管理自己的私有网络，包含连接公网和私网的路由器；另外，浮动IP地址可以让实例使用私有网络连接到外部网络；

这里使用 `私有网络`；

```
# 安装 Neutron
yum install openstack-neutron openstack-neutron-ml2 openstack-neutron-linuxbridge ebtables

# 配置Neutron
vim /etc/neutron/neutron.conf
```

> 数据库配置

```
[database]
connection = mysql+pymysql://neutron:neutron@controller.alec.com/neutron
```

> 配置  `the Modular Layer 2 (ML2) plug-in`  `router service` `overlapping IP addresses`

```
[DEFAULT]
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
```

> 配置RabbitMQ消息队列

```
[DEFAULT]
transport_url = rabbit://alec:alec@controller
#                        用户  密码
```

> 配置认证服务访问

```
[DEFAULT]
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron
```

> 配置网络服务通知计算节点的网络拓扑变化

```
[DEFAULT]
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[nova]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = nova
```

> 配置锁路径

```
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
```

### 7.2.4 配置 Modular Layer 2 (ML2) 插件

```
vim /etc/neutron/plugins/ml2/ml2_conf.ini
```

> 启用flat，VLAN以及VXLAN网络
>
> 配置完ML2插件后，移除type_drivers项，否则会导致数据库不一致

```
[ml2]
type_drivers = flat,vlan,vxlan
```

> 启用VXLAN私有网络

```
[ml2]
tenant_network_types = vxlan
```

> 启用Linuxbridge和layer－2机制

```
[ml2]
mechanism_drivers = linuxbridge,l2population
```

> 启用端口安全扩展驱动

```
[ml2]
extension_drivers = port_security
```

> 配置公共虚拟网络为flat网络

```
[ml2_type_flat]
flat_networks = provider
```

> 为私有网络配置VXLAN网络识别的网络范围

```
[ml2_type_vxlan]
vni_ranges = 1:1000
```

> 启用 ipset 增加安全组规则的高效性

```
[securitygroup]
enable_ipset = true
```

### 7.2.5 配置Linuxbridge代理

```
vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
```

> 将公共虚拟网络和公共物理网络接口对应起来

```
[linux_bridge]
physical_interface_mappings = provider:ens33
```

> 启用VXLAN覆盖网络，配置覆盖网络的物理网络接口的IP地址，启用layer－2 population

```
[vxlan]
enable_vxlan = true
# 控制节点网卡IP
local_ip = 192.168.136.11
l2_population = true
```

> 启用安全组并配置 Linuxbridge iptables firewall driver

```
[securitygroup]
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

> 配置网络参数

```
# 加载br_netfilter模块
depmod 
modprobe br_netfilter
lsmod | grep br_netfilter
    br_netfilter           22256  0 
    bridge                151336  1 br_netfilter

sysctl -w net.bridge.bridge-nf-call-iptables=1
sysctl -w net.bridge.bridge-nf-call-ip6tables=1
```

### 7.2.6 配置layer－3代理

```
vim /etc/neutron/l3_agent.ini
```

> 配置Linuxbridge接口驱动和外部网络网桥

```
[DEFAULT]
interface_driver = linuxbridge
```

### 7.2.7 配置DHCP代理

```
vim /etc/neutron/dhcp_agent.ini
```

> 配置Linuxbridge驱动接口，DHCP驱动并启用隔离元数据，这样在公共网络上的实例就可以通过网络来访问元数据

```
[DEFAULT]
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
```

### 7.2.8 配置元数据代理

```
vim /etc/neutron/metadata_agent.ini
```

> 配置元数据主机以及共享密码

```
[DEFAULT]
nova_metadata_host = controller
metadata_proxy_shared_secret = METADATA_SECRET
```

### 7.2.9 配置Nova使用Neutron

```
vim /etc/nova/nova.conf
[neutron]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron
service_metadata_proxy = true
metadata_proxy_shared_secret = METADATA_SECRET
```

### 7.2.10 启动服务

> 网络服务初始化脚本需要一个超链接 `/etc/neutron/plugin.ini `指向ML2插件配置文件`/etc/neutron/plugins/ml2/ml2_conf.ini`。如果超链接不存在，使用下面的命令创建它

```
ln -sv /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugin.ini
```

> 同步数据库

```
su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
```

> 重启nova服务

```
systemctl restart openstack-nova-api.service
```

> 启动服务

```
systemctl start neutron-server neutron-linuxbridge-agent neutron-dhcp-agent neutron-metadata-agent
systemctl enable neutron-server neutron-linuxbridge-agent neutron-dhcp-agent neutron-metadata-agent
```

> 私有网络还需要启动`layer-3`服务

```
systemctl enable neutron-l3-agent.service
systemctl start neutron-l3-agent.service
```

## 7.3 部署Neutorn(Compute节点)

### 7.3.1 安装配置neutron

```
# 安装配置 Neutron
yum install -y openstack-neutron-linuxbridge ebtables ipset

# 配置 Neutron
vim /etc/neutron/neutron.conf
```

> 配置rabbitmq

```
[DEFAULT]
transport_url = rabbit://alec:alec@controller
```

> 配置认证服务访问

```
[DEFAULT]
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron
```

> 配置锁路径

```
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
```

### 7.3.2 配置Linuxbridge代理

```
vim /etc/neutron/plugins/ml2/linuxbridge_agent.ini
```



> 将公共虚拟网络和公共物理网络接口对应起来

```
[linux_bridge]
physical_interface_mappings = provider:ens33
```

> 启用VXLAN覆盖网络，配置覆盖网络的物理网络接口的IP地址，启用layer－2 population

```
[vxlan]
enable_vxlan = True
# 管理网络网卡IP
local_ip = 192.168.136.13
l2_population = True
```

> 启用安全组并配置 Linuxbridge iptables firewall driver

```
[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

> 配置网络参数

```
# 加载br_netfilter模块
depmod 
modprobe br_netfilter
lsmod | grep br_netfilter
    br_netfilter           22256  0 
    bridge                151336  1 br_netfilter

sysctl -w net.bridge.bridge-nf-call-iptables=1
sysctl -w net.bridge.bridge-nf-call-ip6tables=1
```

### 3.7.3 配置Nova使用Neutron

```
vim /etc/nova/nova.conf
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron
```

### 3.7.5 启动服务

> 重启nova服务

```
systemctl restart openstack-nova-compute.service
```

> 启动Linux bridge agent

```
systemctl enable neutron-linuxbridge-agent.service
systemctl start neutron-linuxbridge-agent.service
```

## 7.4 检查服务状态

```
openstack network agent list
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host       | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
| 0947fd08-bc7d-4735-bd84-40a058e4fb3a | Linux bridge agent | controller | None              | :-)   | UP    | neutron-linuxbridge-agent |
| 36e61dcd-8521-4ffe-8cf7-d5851898c63b | L3 agent           | controller | nova              | :-)   | UP    | neutron-l3-agent          |
| b21a5f68-f293-4562-a3d3-920043da63c1 | Metadata agent     | controller | None              | :-)   | UP    | neutron-metadata-agent    |
| e0cb00ff-df6d-44aa-8136-36b9afaf2c8f | DHCP agent         | controller | nova              | :-)   | UP    | neutron-dhcp-agent        |
| f29dfff9-fed0-406f-b287-c74932f0fc8e | Linux bridge agent | compute    | None              | :-)   | UP    | neutron-linuxbridge-agent |
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
```

# 8 Horizon - Dashboard

## 8.2 部署Horizon 

### 8.2.1 安装配置Horizon

```
# 安装 dashboard
yum install -y openstack-dashboard

# 配置 dashboard
vim /etc/openstack-dashboard/local_settings
```

> 在 `controller` 节点上配置仪表盘以使用 OpenStack 服务

```
OPENSTACK_HOST = "controller"
```

> 允许所有主机访问仪表板

```
ALLOWED_HOSTS = ['*', ]
```

> 配置 `memcached` 会话存储服务

```
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'controller:11211',
    }
}
```

> 启用第3版认证API

```
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST
```

> 启用对域的支持

```
OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
```

> 配置API版本

```
OPENSTACK_API_VERSIONS = {
    "data-processing": 1.1,
    "identity": 3,
    "image": 2,
    "volume": 2,
    "compute": 2,
}
```

> 通过仪表盘创建用户时的默认域配置为 `default` 

```
OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"
```

> 通过仪表盘创建的用户默认角色配置为 `demo`

```
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "demo"
```

> 如果Neutron是网络1(公共网络)，禁用支持3层网络服务；
>
> 如果是私有网络，跳过；

```
OPENSTACK_NEUTRON_NETWORK = {
    ...
    'enable_router': False,
    'enable_quotas': False,
    'enable_distributed_router': False,
    'enable_ha_router': False,
    'enable_lb': False,
    'enable_firewall': False,
    'enable_vpn': False,
    'enable_fip_topology_check': False,
}
```

> 配置时区

```
TIME_ZONE = "Asia/Shanghai"
```

> 添加 WSGIApplicationGroup 到 `/etc/httpd/conf.d/openstack-dashboard.conf`

```
vim /etc/httpd/conf.d/openstack-dashboard.conf
WSGIApplicationGroup %{GLOBAL}
```

### 8.2.2 启动服务

```
systemctl restart httpd.service memcached.service
```


