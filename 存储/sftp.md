http://blog.csdn.net/xinxin19881112/article/details/46831311 sftp搭建

```shell
mkdir -p /data/sftp
groupadd -g 5001 sftp
useradd -g 5001 -u 5001 -s /sbin/nologin -d /data/sftp/sftpuser sftpuser
echo "aaa@2015" | passwd sftpuser --stdin
 
vim /etc/ssh/sshd_config
#Subsystem      sftp    /usr/libexec/openssh/sftp-server
#最后加入
Subsystem       sftp    internal-sftp
Match Group sftp
ChrootDirectory /data/sftp/%u
ForceCommand    internal-sftp
AllowTcpForwarding no
X11Forwarding no
 
chown root:sftp /data/sftp/sftpuser
chmod 755 /data/sftp/sftpuser
mkdir /data/sftp/sftpuser/upload
chmod 755 /data/sftp/sftpuser/upload
chown sftpuser:sftp /data/sftp/sftpuser/upload

service sshd restart
```

