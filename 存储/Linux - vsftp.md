### 普通用户

创建用户 (nologin有问题, 待解决)

```shell
useradd -s /sbin/nologin ftpuser
```

修改vsftp配置文件

```shell
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
xferlog_enable=YES
connect_from_port_20=YES
xferlog_std_format=YES
chroot_local_user=YES
listen=YES
pam_service_name=vsftpd
userlist_enable=YES
pasv_address=10.130.103.150
```

  anonymous_enable=NO  local_enable=YES  write_enable=YES  local_umask=022  dirmessage_enable=YES  xferlog_enable=YES  connect_from_port_20=YES  xferlog_std_format=YES  chroot_local_user=YES  listen=YES  pam_service_name=vsftpd  userlist_enable=YES  pasv_address=10.130.103.150  

### 虚拟用户

创建用户数据库文件, 第一行是虚拟用户, 第二行是密码

```shell
cat << EOF >> /etc/vsftpd/vusers.txt
ftpuser
ftpuser@123
EOF
```

创建用户, 当ftp使用虚拟用户登录时, 会被映射为这个用户

```shell
useradd -s /sbin/nologin ftpuser
chmod u-w ftpuser
```

设置权限, 编译用户数据库文件

```shell
cd /etc/vsftpd/
db_load -T -t hash -f vusers.txt vusers.db
chmod 600 vusers.db
```

修改pam配置文件

```shell
cat << EOF >> /etc/pam.d/vsftpd
auth required pam_userdb.so db=/etc/vsftpd/vusers
account required pam_userdb.so db=/etc/vsftpd/vusers
EOF
```

修改vsftp配置文件

```shell
cp /etc/vsftpd/vsftpd.conf /etc/vsftpd/vsftpd.conf.default

cat << EOF >> /etc/vsftpd/vsftpd.conf
anonymous_enable=NO
local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
xferlog_enable=YES
connect_from_port_20=YES
xferlog_std_format=YES
listen=NO
listen_ipv6=YES
#设置PAM使用的名称,该名称就是/etc/pam.d/目录下vsfptd文件的文件名
pam_service_name=vusers
userlist_enable=YES

# 追加下面的配置
#将所有用户限制在主目录
chroot_local_user=YES
allow_writeable_chroot=YES
local_root=/home/ftpuser
#虚拟用户和本地用户有相同的权限
virtual_use_local_privs=YES
#表示是否开启vsftpd虚拟用户的功能，yes表示开启，no表示不开启
guest_enable=YES
guest_username=ftpuser
#指定每个虚拟用户账号配置目录
user_config_dir=/etc/vsftpd/vusers.d
pasv_enable=YES
pasv_min_port=40000
pasv_max_port=40080
pasv_promiscuous=YES
EOF
```

为虚拟用户建立独立配置文件

```shell
mkdir /etc/vsftpd/vusers.d/

cat << EOF >> /etc/vsftpd/vusers.d/ftpuser
#表示使用本地用户登录到ftp时的默认目录
local_root=/data/nfsdata
write_enable=YES
EOF
```

