[toc]
# RabbitMQ部署
## 1. 升级Python到2.7.13
```
- yum install -y zlib-devel

- 解压 Python-2.7.13.tgz
    tar -xf Python-2.7.13.tgz
    
- 进入解压好的目录
    cd Python-2.7.13
    
- 安装Python2.7.13
    ./configure --prefix=/usr/local/python27
    make && make install
    
- 替换Python启动文件
    rm /usr/bin/python
    ln -s /usr/local/python27/bin/python /usr/bin/python
    
- 验证修改
    python
Python 2.7.13 (default, Aug 15 2017, 13:03:11)
[GCC 4.4.7 20120313 (Red Hat 4.4.7-18)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>>

- 安装了Python2.7.13后，原本系统中使用python的程序，会由于Python环境改变而无法使用，
    以yum为例：
    vim /usr/bin/yum
        把
    #!/usr/bin/python
        改成
    #!/usr/bin/python2.6
- heartbeat的hb_gui也需要修改，方法同上。
```
## 2. 抽音依赖包安装
```
- 解压 site-package.tar 到 /usr/local/python27/lib/python2.7/ 下的 site-packages 中 
    tar -xf site-package.tar -C /usr/local/python27/lib/python2.7/
```
## 3. 安装erlang
```
- 安装erlang依赖环境
    yum install -y ncurses-devel openssl openssl-devel unixODBC-devel
    
- 安装erlang
    yum install -y erlang-18.3.4.4-1.el6.x86_64.rpm
    
- 在 /etc/profile 中添加erlang环境变量
    vim /etc/profile
        export ERLANG_HOME=/usr/lib64/erlang
        PATH=$PATH:$ERLANG_HOME
    source /etc/profile 
    
- 验证安装，在命令行输入erl
[root@localhost rabbitmq]# erl
Erlang/OTP 18 [erts-7.3] [source] [64-bit] [async-threads:10] [hipe] [kernel-poll:false]

Eshell V7.3  (abort with ^G)
1> 
```
## 4. 安装RabbitMQ
```
    yum install -y socat-1.7.2.3-1.el6.x86_64.rpm compat-readline5-5.2-17.1.el6.x86_64.rpm
    yum install -y rabbitmq-server-3.6.5-1.noarch.rpm

    /etc/init.d/rabbitmq-server start
    
- 启用rabbitmq内置监控界面
    rabbitmq-plugins enable rabbitmq_management
    
- 查看组插件启用情况
    rabbitmq-plugins list
    
- 监控插件启用后会打开一个15672端口
[root@SDPSTT01 ~]# netstat -antp | grep 15672
tcp        0      0 0.0.0.0:15672               0.0.0.0:*                   LISTEN      13586/beam

- 增加rabbitmq账户，默认的guest账户只能通过http://localhost:15672登录
  另外抽音程序也会通过这个账户访问rabbitmq
    rabbitmqctl add_user pachira pachira              # 第一个pachira是账号，第二个pachira是密码
    rabbitmqctl set_user_tags pachira administrator   # 给pachira账号赋予最高权限
    rabbitmqctl list_users

- 在访问界面后需要对pachira用户增加对"/"的权限后，抽音才能使用
```    
    
## 5. 设置rabbitmq集群
```
- 对两台服务器做好域名解析
    vim /etc/hosts
    192.168.111.132  node3
    192.168.111.136  node4

- 使主备机的rabbitmq的cookie文件保持一致
    [root@node4 ~]# scp -r /var/lib/rabbitmq/.erlang.cookie node2:/var/lib/rabbitmq/

- 将node4加入到node3的集群中
    [root@node4 ~]# rabbitmqctl stop_app
    [root@node4 ~]# rabbitmqctl join_cluster rabbit@node3
    [root@node4 ~]# rabbitmqctl start_app

- 查看集群状态
    [root@node4 ~]# rabbitmqctl cluster_status
    Cluster status of node rabbit@node4 ...
    [{nodes,[{disc,[rabbit@node3,rabbit@node4]}]},
     {running_nodes,[rabbit@node3,rabbit@node4]},
     {cluster_name,<<"rabbit@node4">>},
     {partitions,[]},
     {alarms,[{rabbit@node3,[]},{rabbit@node4,[]}]}]
```

## 6. 设置队列镜像
```
对hello队列设置同步策略（镜像只对做过本地化的数据有效）
    [root@node3 ~]# rabbitmqctl set_policy ha-all "hello" '{"ha-mode":"all"}'

对he开头的队列设置同步策略
[root@node3 ~]# rabbitmqctl set_policy ha-all "^ha" '{"ha-mode":"all"}'
```







安装erlang
```
zypper in ncurses-devel openssl openssl-devel unixODBC-devel socat
./otp_build autoconf
./configure --prefix=/usr/data/erlang-21.0.9
make
make install

```

安装rabbitmq
```
后台启动
	rabbitmq-server -detached

同步主备 .erlang.cookie
	appadmin@SCDCA0000774:~> scp /home/appadmin/.erlang.cookie appadmin@SCDCA0000775:/tmp/
	appadmin@SCDCA0000775:~> mv /tmp/.erlang.cookie /home/appadmin/


	

设置同步策略
appadmin@SCDCA0000774:~> rabbitmqctl set_policy ha-all "^" '{"ha-mode":"all"}'
	
	
```



