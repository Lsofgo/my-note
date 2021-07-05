```shell
# 创建docker用户
groupadd -g 8000 docker
useradd -g 8000 -u 8000 docker
echo <密码 | passwd docker --stdin
history -c

# 安装docker-ce
yum localinstall docker-ce-17.06.2.ce-1.el7.centos.x86_64.rpm container-selinux-2.10-2.el7.noarch.rpm

mkdir /etc/docker
tee /etc/docker/daemon.json <<-'EOF'
{
"registry-mirrors": ["https://mirror.ccs.tencentyun.com"]
}
EOF

# 重启docker
systemctl daemon-reload
systemctl enable docker
systemctl restart docker
```

