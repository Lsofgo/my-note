# 1. 安装Docker

```
wget https://mirrors.aliyun.com/repo/Centos-7.repo -P /etc/yum.repos.d/
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -P /etc/yum.repos.d/

yum clean all
yum install -y docker-ce
```

```
systemctl start docker
systemctl enable docker
```

修改docker数据存储目录

```
[root@localhost ~]# vim /usr/lib/systemd/system/docker.service
#ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
ExecStart=/usr/bin/dockerd --graph=/data/docker -H fd:// --containerd=/run/containerd/containerd.sock

[root@localhost ~]# systemctl daemon-reload
[root@localhost ~]# systemctl restart docker

[root@localhost ~]# ll /data/docker/
总用量 0
drwx--x--x 4 root root 120 5月  12 19:54 buildkit
drwx-----x 2 root root   6 5月  12 19:54 containers
drwx------ 3 root root  22 5月  12 19:54 image
drwxr-x--- 3 root root  19 5月  12 19:54 network
drwx-----x 3 root root  40 5月  12 19:54 overlay2
drwx------ 4 root root  32 5月  12 19:54 plugins
drwx------ 2 root root   6 5月  12 19:54 runtimes
drwx------ 2 root root   6 5月  12 19:54 swarm
drwx------ 2 root root   6 5月  12 19:54 tmp
drwx------ 2 root root   6 5月  12 19:54 trust
drwx-----x 2 root root  50 5月  12 19:54 volumes
```

配置阿里云镜像加速器

```
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<-'EOF'
{
"registry-mirrors": ["https://rfj1yucr.mirror.aliyuncs.com"]
}
EOF

systemctl daemon-reload
systemctl restart docker
```

Docker默认使用本机sock连接

设置Docker使用远程TCP连接

```
[root@localhost ~]# vim /usr/lib/systemd/system/docker.service 
ExecStart=/usr/bin/dockerd --graph=/data/docker -H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock -H fd:// --containerd=/run/containerd/containerd.sock

[root@localhost ~]# systemctl daemon-reload
[root@localhost ~]# systemctl restart docker

[root@localhost ~]# netstat -antp | grep 2375
tcp6       0      0 :::2375                 :::*                    LISTEN      1995/dockerd

[root@localhost ~]# docker -H 192.168.88.128:2375 images
REPOSITORY    TAG       IMAGE ID       CREATED        SIZE
hello-world   latest    d1165f221234   2 months ago   13.3kB
```

# 2 常用操作

## 2.1 镜像操作

### 2.1.1 docker image search

```
从Docker Hub搜索镜像

旧版命令： 
        docker search

示例：
        [root@localhost ~]# docker search nginx
        NAME                                                   DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
        nginx                                                  Official build of Nginx.                        11081               [OK]                
        jwilder/nginx-proxy                                    Automated Nginx reverse proxy for docker con…   1562                                    [OK]
```

### 2.1.2 docker image pull

```
从Docker Hub下载镜像

示例：
        [root@localhost ~]# docker image pull nginx:1.14-alpine

旧版命令：
        docker pull

```

### 2.1.3 docker image ls

```
列出当前镜像

示例：
        [root@localhost ~]# docker image ls
        REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
        nginx               1.14-alpine         cafef9fe265b        9 days ago          16MB
        busybox             latest              d8233ab899d4        4 weeks ago         1.2MB

选项：
        --no-trunc      完整显示 IMAGE ID

旧版命令：
        docker images
```

### 2.1.4 docker image rm

```
删除镜像

旧版命令：
        docker rmi
```

### 2.1.5 docker image tag

```
docker tag 
```


## 2.2 容器操作

### 2.2.1 docker container create

```
创建一个容器
```

### 2.2.2 docker container run

```
创建并运行一个容器  docker container run [OPTIONS] IMAGE [COMMAND] [ARG...]

选项：
        -t, --tty           分配一个终端
        -i, --interactive   保持标准输入
        --name string       给容器起一个NAME
        --network string    指定容器使用的网络，默认是default(docker0)(docker network ls)
        --rm                当容器停止，就自动删除
        -d, --detach        容器启动后在后台运行

示例：
        创建交互运行的容器
        docker run --name b1 -it busybox:latest

        创建后台运行的容器
        [root@localhost ~]# docker run --name web1 -d nginx:1.14-alpine


其他命令：
        docker run
```

### 2.2.3 docker container ls

```
查看当前运行中的容器

选项：
        -a      列出所有容器，包括未启动的容器

示例：
        [root@localhost ~]# docker ps
        CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
        5a7313dfd89c        busybox:latest      "sh"                8 minutes ago       Up 17 seconds                           b1
        容器ID              依赖镜像             容器中运行的命令      创建时间             当前状态            映射的端口           容器名(可缺省)

其他命令：
        docker ps
```

### 2.2.4 docker container [start|stop|restart|kill|status]

```
[启动|停止|重启|强行终止|查看]容器

示例：
        交互方式启动容器：
        [root@localhost ~]# docker start -i -a b1
```

### 2.2.5 docker container rm

```
删除一个停止运行的镜像

选项：
        -f      强制删除运行中的镜像(会使用kill信号)

示例：
        [root@localhost ~]# docker container rm b1
        b1
        [root@localhost ~]# docker ps -a
        CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES

其他命令：
        docker rm
```

### 2.2.6 docker container exec

```
在容器中执行命令

选项：
        -t, --tty           分配一个终端
        -i, --interactive   保持标准输入

示例：
        docker container exec -it kvstore1 /bin/sh

其他命令： 
        docker exec
```

### 2.2.7 docker container logs

```
查看容器的日志
        因为daocker容器中只会运行一个进程，所以会把日志输出到标准输出；

选项：
        -f, --follow        跟踪日志

其他命令：
        docker logs
```

### 2.2.7 docker container [pause|unpause]

```
[暂停|取消暂停]容器
```

### 2.2.8 docker container top

```
显示容器内运行的进程
```



# 3 docker镜像管理

## 3.1 镜像生成

- Dockerfile
- 基于容器制作
- Docker Hub automated builds (基于dockerfile)

## 3.2 基于容器制作镜像 docker commit

```
docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]

选项：
        -a, --author string    作者信息
        -c, --change list      修改dockerfile指令，比如容器启动时运行的命令
        -m, --message string   Commit message
        -p, --pause            commit时，暂停容器
```

```
示例1：
        生成一个busybox容器，并在里面创建网页文件
            [root@localhost ~]# docker run --name b1 -it busybox
                / # mkdir -pv /data/html
                / # echo "busybox httpd server" > /data/html/index.html
        
        通过容器b1 生成镜像，生成时暂停容器
            [root@localhost ~]# docker commit -p b1

        查看新生成的镜像，没有仓库名和标签
            [root@localhost ~]# docker image ls
            REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
            <none>                   <none>              a92689fb5cb8        17 seconds ago      1.2MB
            nginx                    1.14-alpine         cafef9fe265b        9 days ago          16MB

        给镜像打标签
            [root@localhost ~]# docker image tag a92689fb5cb8 lance/httpd:v0.1-1
            [root@localhost ~]# docker images
            REPOSITORY               TAG                 IMAGE ID            CREATED              SIZE
            lance/httpd              v0.1-1              a92689fb5cb8        About a minute ago   1.2MB

        给镜像再打一个标签
            [root@localhost ~]# docker image tag lance/httpd:v0.1-1 lance/httpd:latest
            [root@localhost ~]# docker images
            REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
            lance/httpd              latest              a92689fb5cb8        2 minutes ago       1.2MB
            lance/httpd              v0.1-1              a92689fb5cb8        2 minutes ago       1.2MB
```

```
示例2：
        commit busybox容器，指定作者，修改启动时运行的命令为"前台运行httpd"
            [root@localhost ~]# docker commit -a "lance" -c 'CMD ["/bin/httpd","-f","-h","/data/html"]' -p b1 lance/httpd:v0.2

        运行这个容器
            [root@localhost ~]# docker run --name t2 -d lance/httpd:v0.2

        查看容器运行状态
            [root@localhost ~]# docker ps
            CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
            f423d5e4716a        lance/httpd:v0.2    "/bin/httpd -f -h /d…"   6 seconds ago       Up 4 seconds                            t2

        查看运行的容器的IP
            [root@localhost ~]# docker inspect t2 | grep IPAddress

        访问验证
        [root@localhost ~]# curl 172.17.0.5
        busybox httpd server
```

## 3.2 上传镜像 docker push

```
把打包好的镜像推送到远程Repositories仓库

示例1：
        把制作好的镜像推送到DockerHub
            浏览器访问dockerhub，登录账户，创建镜像仓库
        
        本地登录DockerHub
            [root@localhost ~]# docker login -u lance
        
        镜像推送到dockerhub(不加tag会推送全部)
            [root@localhost ~]# docker push lance/httpd:v0.2
```

## 3.3 导入导出镜像

### 3.3.1 导出镜像 docker save

```
导出镜像到文件
        docker image save [OPTIONS] IMAGE [IMAGE...]

选项：
        -o, --output string     输出到string文件

示例：
        docker image save -o lacne_http_v0.2.tgz  lance/httpd:v0.2
```

### 3.3.2 导入镜像 docker load

```
从tar包导入镜像
        docker load [OPTIONS]

选项：
        -i, --input string      从tar包读取文件

示例：
        [root@localhost ~]# docker image load -i lacne_http_v0.2.tgz
```



# 4 docker网络管理

docker的网络名称空间、UTS、IPC可以被容器共享；

可以构建出类似KVM的 隔离、桥接、NAT、物理桥接 网络模型；

## 4.1 指定网络启动

### 4.1.1 bridge(默认)

```
[root@localhost ~]# docker run --name t1 -it --network bridge  --rm busybox:latest
    / # ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
        valid_lft forever preferred_lft forever
    22: eth0@if23: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue 
        link/ether 02:42:ac:11:00:04 brd ff:ff:ff:ff:ff:ff
        inet 172.17.0.4/16 brd 172.17.255.255 scope global eth0
        valid_lft forever preferred_lft forever

```

### 4.1.2 none 隔离网络

容器中只有lo网卡
```
[root@localhost ~]# docker run --name t1 -it --network none --rm busybox:latest
    / # ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
        valid_lft forever preferred_lft forever
```

### 4.1.3 联盟式容器

联盟式容器是指使用某个已存在容器的网络接口的容器，联盟内的各个容器共享使用；因此，联盟式容器彼此间完全无隔离；

联盟式容器彼此间虽然共享一个网络名称空间，但其他名称空间如User、Mount等还是隔离的；

联盟式容器彼此间存在端口冲突的可能系，因此通常只会在多个容器上的程序需要程序lookback接口互相通信、或对某已存在的容器的网络属性进行监控时才使用此种模式的网络模型。

```
示例1：
        容器b2和b1共享b1网络
        创建一个监听80端口的http服务容器
                [root@localhost ~]# docker run --name b1 -it --rm busybox
                    / # httpd -f -h /home/

        创建一个联盟式容器，并查看其监听的端口
                [root@localhost ~]# docker run --name b2 -it --rm --network container:b1 busybox
                    / # netstat -antlp | grep 80
                    tcp        0      0 :::80                   :::*                    LISTEN      -
                    / # wget -O - -q 127.0.0.1
                    hello world
```

### 4.1.4 host模式

host模式下，容器共享主机的网络，于主机同ip；

```
示例2：
        容器b2共享宿主机网络
        [root@localhost ~]# docker run --name b2 -it --rm --network host  busybox
            / # ip a
            1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
                link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
                inet 127.0.0.1/8 scope host lo
                valid_lft forever preferred_lft forever
                inet6 ::1/128 scope host 
                valid_lft forever preferred_lft forever
            2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000
                link/ether 00:0c:29:bb:36:eb brd ff:ff:ff:ff:ff:ff
                inet 192.168.146.134/24 brd 192.168.146.255 scope global dynamic ens33
                valid_lft 1236sec preferred_lft 1236sec
                inet6 fe80::e3c4:308:4d7e:4650/64 scope link 
                valid_lft forever preferred_lft forever
            3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue 
                link/ether 02:42:66:8e:57:e3 brd ff:ff:ff:ff:ff:ff
                inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
                valid_lft forever preferred_lft forever
                inet6 fe80::42:66ff:fe8e:57e3/64 scope link 
                valid_lft forever preferred_lft forever

        在容器内启动一个80端口的http服务
            / # echo "hello world" > /tmp/index.html
            / # httpd -h /tmp/
            / # netstat -antp | grep 80
            tcp        0      0 :::80                   :::*                    LISTEN      9/httpd

        在宿主机请求127.0.0.1
            [root@localhost ~]# curl 127.0.0.1
            hello world
```



## 4.2 容器端口映射

运行容器时 使用 -P 或者 -p 参数指定容器的端口

- -P Docker 会 随机映射一个端口, 到容器开放的网络端口;
- -p Docker 会 指定一个端口, 到容器开放的网络端口;

使用 `docker ps` 可以看到容器映射的端口;

> -P 随机映射容器暴露的端口

```
[root@localhost ~]# docker run -d --rm --name nginx -P nginx
64eb25840556d152925dbc794be95dc74e3f4ebc37f4350f517b3ce2cdd923bf

[root@localhost ~]# docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED         STATUS         PORTS                                     NAMES
64eb25840556   nginx     "/docker-entrypoint.…"   7 seconds ago   Up 5 seconds   0.0.0.0:49153->80/tcp, :::49153->80/tcp   nginx

[root@localhost ~]# curl http://127.0.0.1:49153
```

> -p ip::容器端口  指定ip, 随机映射端口

```
[root@localhost ~]# docker run -d --rm --name nginx -p 192.168.88.128::80 nginx
9d77d2fa1c9089c000deebf17baa81b235143b63102fbc691c15414a217e8b75

[root@localhost ~]# docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS                          NAMES
9d77d2fa1c90   nginx     "/docker-entrypoint.…"   14 seconds ago   Up 13 seconds   192.168.88.128:49153->80/tcp   nginx

[root@localhost ~]# netstat -antp | grep docker
tcp        0      0 192.168.88.128:49153    0.0.0.0:*               LISTEN      17398/docker-proxy
```

> -p 外部端口:容器端口  指定端口映射

```
[root@localhost ~]# docker run -d --rm --name nginx -p 8888:80 nginx
8974acf9faa5f3210bc9e8e6fe0655e950f08a8b26882ced51c8ac52ad3ee9ae

[root@localhost ~]# docker ps -a
CONTAINER ID   IMAGE     COMMAND                  CREATED         STATUS        PORTS                                   NAMES
8974acf9faa5   nginx     "/docker-entrypoint.…"   3 seconds ago   Up 1 second   0.0.0.0:8888->80/tcp, :::8888->80/tcp   nginx

[root@localhost ~]# curl http://127.0.0.1:8888
```

> -p 主机端口:容器端口/udp	映射udp端口

> -p 主机端口:容器端口 -p 主机端口:容器端口 映射多个端口



# 5 数据卷管理

数据卷是一个可供多个容器使用的本地文件目录, 存储卷的特点:

1. 数据卷可以在容器之间共享和复用, 多挂载时考虑数据同时读写的一致性; (可以使用nfs或主机磁盘挂载代替)
2. 对数据卷的修改会立即生效(数据不会延时生效)
3. 对数据卷的更新, 不会影响镜像
4. 数据卷默认会一直存在, 即使容器被删除

数据卷的使用, 类似Linux下对目录进行mount;

**使用场景:**

- 程序日志
- 访问共同数据目录

## 5.1 文件拷贝 docker cp

### 5.1.1 拷贝容器内文件到主机

```
[root@localhost ~]# docker run -it --rm --name c1 -h c1-test centos /bin/bash
[root@c1-test /]# echo 'docker' > /test.txt

[root@localhost ~]# docker cp c1:/test.txt /tmp/
[root@localhost ~]# cat /tmp/test.txt 
docker
```

### 5.1.2 拷贝文件到容器内

```
[root@localhost ~]# echo 'ceshi' > ceshi.txt
[root@localhost ~]# docker cp ./ceshi.txt c1:/

[root@c1-test /]# cat /ceshi.txt 
ceshi
```



## 5.2 创建数据卷

使用 docker run命令中 -v 来给容器内添加一个数据卷, 也可以在一次docker run命令中, 多次使用 -v 来挂载多个数据卷;

- -v 容器目录	docker分配一个主机路径挂载

- -v 主机目录:容器目录:[ro/rw]

> docker分配主机目录

```
[root@localhost ~]# docker run -it --rm -v /mnt --name centos -h centos-test centos /bin/bash

[root@centos-test ~]# touch /mnt/test.txt

[root@localhost ~]# find /data/docker/volumes/ -name test.txt
/data/docker/volumes/a79eeff6048576458ee12f95e87205f3f26f44fc305d7949a7d536c91a2d0bcf/_data/test.txt
```



> 创建一个新容器并挂载本地目录

```
[root@localhost ~]# echo 'local' > /mnt/local.txt
[root@localhost ~]# docker run -it --rm -v /mnt:/media --name centos -h centos-test centos /bin/bash

[root@centos-test /]# ls /media/
local.txt
```

> 创建一个只读挂载

```
[root@localhost ~]# docker run -it --rm -v /mnt:/media:ro --name centos -h centos-test centos /bin/bash

[root@centos-test /]# touch /media/container.txt
touch: cannot touch '/media/container.txt': Read-only file system
```



## 5.2 docker管理卷

```
docker会自动在宿主机创建一个目录挂载到容器内，这个目录的路径是/var/lib/docker/volumes
docker run -it --name bbox1 -v HOSTDIR busybox

示例：
        [root@localhost ~]# docker inspect b2 -f {{".Mounts"}}
        [{volume 8bf3c6c5d88d287a6a8e283a5d99e654300843a71800f287dc717077fd8ff5c4 /var/lib/docker/volumes/8bf3c6c5d88d287a6a8e283a5d99e654300843a71800f287dc717077fd8ff5c4/_data /data local  true }]
```



# 6 dockerfile

```
FROM 		镜像
MAINTAINER	作者
RUN			执行命令
ENV			环境变量
ADD			添加文件
COPY		复制文件
WORK		工作目录
EXPOSR		暴露端口
CMD			启动命令
```



## 6.1 docker build 工程目录结构

```
[root@localhost img_test]# tree -a ./
    ./
    ├── Dockerfile          # 核心文件
    ├── .dockerignore       # 将在构建镜像时需要忽略的文件、目录写入此文件
    └── PQA                 # 需要向镜像中添加的文件、目录
        ├── logs
        ├── PQA
        └── qa

[root@localhost img_test]# cat .dockerignore 
    ./PQA/logs
```

## 6.2 语法

### 6.2.1 docker build

```
docker build -t 仓库名[:tag] ./
```

### 6.2.2 FROM

- FROM指令是最重要的一个且必须为Dockerfile文件开篇的第一个非注释行，用于为镜像文件构建过程指定基准镜像，后续的指令运行于此基准镜像所提供的运行环境；
- 实践中，基准镜像可以是认可可用镜像文件，默认情况下，docker build会在docker主机上查找指定的镜像文件，在其不存在是，则会从Docker Hub Registry上拉取所需的镜像文件
  - 如果找不到指定的镜像文件，docker build会返回一个错误信息
  - 
```
语法：
    FROM <repository>[:<tag>]
    FROM <repository>@<digest>
        <repository>：指定作为base image的名称；
        <tag>：base image的标签，为可选项，省略是模人物latest；
        <digest>：哈希码
```

### 6.2.2 MAINTAINER(docker_17已废弃，替换为LABEL)

- 用于让Dcokerfile制作者提供本人的详细信息
- Dockerfile并不限制MAINTAINER指令可出现的位置，但推荐将其防止于FROM指令之后

```
语法：
    MAINTAINER <author's detail>
        <author's detail> 可以是任何文本信息，但约定俗成的使用作者名以及邮件地址；
```

### 6.2.3 LABEL

- LABEL指令能够内一个镜像添加元数据
- 一个LABEL是一个键值对

```
LABEL amintainer="lance <wangkuncoin@163.com>"
```



### 6.2.4 COPY

- 用于从Docker主机复制文件至创建的新镜像文件

```
语法：
    COPY <src>...<dest>   ["aaa  vvv",  "aaa"]
    COPY ["<src>",..."<dest>"]
        <src>：要复制的源文件或目录，支持使用通配符；
            一般是相对路径，当前docker build的工作目录；
        <dest>：目标路径，即正在创建的image的文件系统路径；
            建议<dest>使用绝对路径，否则，COPY指定则以WORKDIR为起始路径；
    注意：在路径中有空白字符时，通常使用第二种格式；
```

```
文件复制准则：
        <src>必须是build上下文中的路径，不能是其父目录中的文件；
        如果<src>是目录，则其内部文件或子目录会被递归复制，但<src>目录自身不会被复制；
        如果指定了多个<src>，或在<src>中使用了通配符，则<dest>必须是一个目录，且必须以"/"结尾；
        如果<dest>事先不存在，它将会被自动创建，这包括其父目录路径；
```

```
[root@localhost img01]# vim Dockerfile 
    # Description: test image
    FROM busybox:latest
    # MAINTANIER "lance <wangkuncoin@163.com>"
    LABEL amintainer="lance <wangkuncoin@163.com>"
    COPY index.html /data/web/html/

[root@localhost img01]# vim index.html
    <h1>Busybox httpd service</h1>

[root@localhost img01]# docker build -t lancehttpd:v0.1-1 ./
Sending build context to Docker daemon  3.072kB
Step 1/3 : FROM busybox:latest
 ---> d8233ab899d4
Step 2/3 : LABEL amintainer="lance <wangkuncoin@163.com>"
 ---> [Warning] IPv4 forwarding is disabled. Networking will not work.
 ---> Running in a753cde8d9c3
Removing intermediate container a753cde8d9c3
 ---> a7f43a956a2a
Step 3/3 : COPY index.html /data/web/html/
 ---> f0d79282d532
Successfully built f0d79282d532
Successfully tagged lancehttpd:v0.1-1

[root@localhost img01]# docker images
REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
lancehttpd               v0.1-1              f0d79282d532        14 seconds ago      1.2MB

[root@localhost img01]# docker run --rm --name lacnehtml lancehttpd:v0.1-1 cat /data/web/html/index.html
<h1>Busybox httpd service</h1>
```

### 6.2.5 ADD

- ADD指令类似于COPY指令，ADD支持使用TAR文件和URL路径

```
语法：
    ADD <src>...<dest>
    ADD ["<src>",..."<dest>"]
```

```
使用准则：
        同COPY准则；
        如果<src>为URL且<dest>不以/结尾，则<src>指令的文件将被下载并且直接被创建为<dest>；
                如果<dest>以/结尾，则文件名URL指定的文件将会被直接下载并保存为<dest>/<filename>
        如果<src>是一个本地系统上的压缩格式的tar文件，它将被展开为一个目录，其行为类似于"tar -x"命令；然而，通过url获取到的tar文件将不会自动展开；
        如果<src>有多个，或其简介或直接使用了通配符，则<dest>必须是一个以/结尾的目录路径；如过<dest>不以/结尾，则其被视作一个普通文件，<src>的内容将被直接写入到<dest>；
```

### 6.2.6 WORKDIR

- 用于为Dockerfile中所有的RUN、CMD、ENTRYPOINT、COPY和ADD设定工作目录
- 也是容器交互式运行时登录后的目录

```
语法：
        WORKDIR <dirpath>
            在Dockerfile文件中，WORKDIR可以出现多次，全路径也可以为相对路径，不过，其是相对此前一个WORKDIR指令的路径
            WORKDIR也可以调用由ENV指定定义的变量

示例：
        WORKDIR /var/log
        WORKDIR $STATEPATH
```

### 6.2.7 VOLUME

- 用于在image中创建一个挂载点目录，以挂载Docker host上的卷或其他容器上的卷

```
语法：
        VOLUME <mountpoint>
        VOLUME ["<mountpoint>"]
```
- 如果挂载点目录路径下此前有文件存在，docker run命令会在卷挂载完成后将此前的所有文件复制到新挂载的卷中；

### 6.2.8 EXPOSE

- 用于为容器打开指定要监听的端口以实现与外部通信；(docker run时要使用 -P 参数)
- EXPOSE暴露的端口在run时会分配一个宿主机的随机端口；

```
语法：
        EXPOSE <port>[/<protocol>] <port>[/<protocol>] ...
            <protocol>用于指定传输层协议，可为tco或udp二者之一，默认为tcp协议；

        EXPOSE指令可以一次指定多个端口：
            EXPOST 11211/udp 11211/tcp
```

### 6.2.9 ENV

- 用于为镜像定义所需的环境变量，并可被Dockerfile文件中位于其后的其他指令(如ENV、ADD、COPY等)所调用
- 调用 格式为 $variable_name 或者 ${variable_name}
- ENV指令定义的变量会在容器中存在，使用docker run -e \<key\>=\<value\> 可以对这些变量重新赋值；

```
语法：
        ENV <key> <value>
        ENV <key>=<value>...

        第一种格式中，<key>之后的所有内容均会被视作其<value>的组成部分，因此一次只能设置一个变量；
        第二种格式可用一次设置多个变量，每个变量为一个"<key>=<value>"的键值对；
            如果<value>中包换空格，可以反斜线(\)进行转义，也可以通过对<value>加引号进行标识；
            另外反斜线也可以用于续行；
        定义多个变量时，建议使用第二种方式，以便在同一层中完成所有功能；
```
```
环境变量替换
        $变量名       或者  ${变量名}
                
        ${变量名:-world}          # 如果变量未设置或者为空，则引用word；
        ${变量名:+world}          # 如果变量已存在，则引用word，变量无值则不显示；
```

### 6.2.10 RUN

- 用于指定docker build过程中运行的程序，其可以是任何命令；

```
语法：
        RUN <command>
        RUN ["<executable>","<param01>","<param02>"]

        第一种格式，<command>通常是一个shell命令，且以"/bin/sh -c"来运行它，这意味着此进程在容器中的PID不为1，不能接受Unix信号，
            因此，当使用docker stop <container>命令停止容器时，此进程接收不到SIGTERM信号；
        第二种语法格式中的参数是一个JSON格式的数组，其中<executable>为要运行的命令，后面的<paramN>为要传递的参数；
            然而，此种格式的命令不会以"/bin/sh -c"来发起，因此常见的shell操作如变量替换以及通配符(.*?等)替换将不会进行；
            不过，如果要运行的命令依赖于此shell特性的话，可以将其替换为类似下面的格式：
                RUN ["/bin/bash", "-c", "<executable>", "param01"]
```


### 6.2.11 CMD

- 类似于RUN指令，CMD指令也可用于运行任何命令或应用程序，不过，二者的运行时间点不同；
  - RUN指令运行于镜像的构建过程中，而CMD指令运行于基于Dockerfile构建出的新镜像文件启动一个容器时；
  - CMD指令的首要目的在于为启动的容器指定默认要运行的程序，且其运行结束后，容器也将终止；不过，CMD指定的命令其可以被docker run的命令行选项所覆盖；
  - 在Dockerfile中可以存在多个CMD指令，但进最后一个会生效

```
语法：
        CMD <command>
        CMD ["<executable>", "<param01>", "<param02>"]
        CMD ["<param01>", "<param02>"]

        前两种语法格式的意义同RUN：
            第一种格式，<command>通常是一个shell命令，且以"/bin/sh -c"来运行它，这意味着此进程在容器中的PID不为1，不能接受Unix信号，
                因此，当使用docker stop <container>命令停止容器时，此进程接收不到SIGTERM信号；
            第二种语法格式中的参数是一个JSON格式的数组，其中<executable>为要运行的命令，后面的<paramN>为要传递的参数；
                然而，此种格式的命令不会以"/bin/sh -c"来发起，因此常见的shell操作如变量替换以及通配符(.*?等)替换将不会进行；
                不过，如果要运行的命令依赖于此shell特性的话，可以将其替换为类似下面的格式：
                    CMD ["/bin/bash", "-c", "<executable>", "param01"]
        第三种则用于为ENTRYPOINT指令提供默认参数列表；
```

### 6.2.12 ENTRYPOINT

- 类似CMD指令的功能，用于为容器指定默认的运行程序，从而使得容器像是一个单独的可执行程序；
- 与CMD不同的是，由ENTRYPOINY启动的程序不会被docker run命令行指定的参数所覆盖，而且，这些命令行参数会被当作参数传递给ENTRYPOINT指定的程序；
  - 不过，docker run命令的--entrypoint选项的参数可覆盖ENTRYPOINT指令指定的程序；

- docker run命令传入的命令参数，会覆盖CMD指令的内容并且附加到ENTRYPOINT命令最后，作为其参数使用；
- Dockerfile文件中可以存在多个ENTRYPOINT指令，但仅有最后一个会生效；

```
语法：
        ENTRYPOINT <command>
        ENTRYPOINT ["exectable", "<param01>", "<param02>"]
```

```
示例：
        [root@localhost img02]# docker run --rm lancehttpd:v0.2-5 ls /data/web/

        [root@localhost img02]# docker inspect f11371c80029 -f {{.Config.Cmd}}
            [ls /data/web/]
            
        [root@localhost img02]# docker inspect f11371c80029 -f {{.Config.Entrypoint}}
            [/bin/sh -c /bin/httpd -f -h ${WEB_DOC_ROOT}]

        [root@localhost img02]# docker inspect f11371c80029 -f {{.Args}}
            [-c /bin/httpd -f -h ${WEB_DOC_ROOT} ls /data/web/]
```


### 6.2.13 USER

- 用于指定运行image时的或运行Dockerfile中任何RUN、CMD或ENTRYPOINT指令指定的程序的用户名或UID
- 默认情况下，container的运行身份是root

```
语法：
    USER <UID>|<UserName>

    需要注意的是，<UID>可以为任意数字，但实践中必须为/etc/passwd中某用户的有效UID，否则，docker run命令会运行失败；
```

### 6.2.14 HEALTHCHECK

- HEALTHCHECK指令告诉docker如何检验一个container是否还在继续工作

```
语法：
        HEALTHCHECK [OPTIONS] CMD command
            这里的CMD是一个关键词
            通过在container中运行命令来检查container健康状态；
            OPTIONS：
                --interval=DURATION(default: 30s)       # 间隔时间
                --timeout=DURATION(default: 30s)        # 超时时间
                --start-period=DURATION(default: 0s)    # 启动延时
                --retries=N(default: 3)                 # 重试次数
            命令的退出状态代表了container的健康状态：
                0： 健康
                1： 不健康
                2： 保留值，不要使用这个值
            示例：
                HEALTHCHECK --interval=5m --timeout=3s \
                    CMD curl -f http://localhost/ || exit 1

        HEALTHCHECK NONE
            拒绝任何的健康监测，包括默认的来自base image的健康监测
```

### 6.2.15 SHELL

- SHELL指令允许使用其他shell程序来代替默认的shell；(在RUN，CMD使用非JSON格式时会用到)
- 默认的shell在linux是["/bin/sh", "-c"]，在windows是["cmd", "/S", "/C"];

```
语法：
        SHELL ["executable", "param01", "param02"]

        SHELL指令必须以JSON格式写入Dockerfile；
        SHELL指令可以被使用多次；
        每条SHELL指令都会覆盖所有先前的SHELL指令，并影响所有后续指令；
```

### 6.2.16 STOPSIGNAL

- 该STOPSIGNAL指令设置将发送到容器的系统调用信号以退出；
- 此信号可以是与内核的系统调用表中的位置匹配的有效无符号数，例如9，或SIGNAME格式的信号名，例如SIGKILL；

```
语法：
    STOPSIGNAL <signal>
```

### 6.2.17 ARG

- 设置变量命令，用于指定传递给构建运行时的变量。
- ARG命令定义了一个变量，在docker build创建镜像的时候，使用 --build-arg \<varname\>=\<value\>来指定参数；

```
语法：
        ARG <name>[=<default value>]

        如果用户在build镜像时指定了一个没有定义在Dockerfile中的参数，那么将有一个Warning；
        ARG定义的参数默认值，当build镜像时没有指定参数值，将会使用这个默认值；

```

### 6.2.18 ONBUILD

- 用于在Dockerfile中定义一个触发器；
- Dockerfile用于build镜像文件01后，当镜像文件01作为base image被另一个Dockerfile用作FROM指令的参数，构建新的镜像02时；
- 在后面这个镜像02中的FROM指令在build过程被执行时，将会“触发”创建在镜像01Dockerfile中的ONBUILD指令定义的触发器；

```
语法：
        ONBUILD <Dockerfile_INSTRUCTION>
        尽管任何指令都可以被作为触发器指令，但ONBUILD不能自我嵌套，另外也不会触发FROM和MAINTAINER指令；
        使用包含ONBUILD指令的Dockerfile构建的奖项应该使用特殊的标签，例如 python:3.6.5-onbuild
        在ONBUILD指令中使用ADD或COPY指令应该格外小心，因为新构建过程的上下文在缺少指定的源文件时会失败；
```

## 6.3 构建my-nginx镜像

> DockerFile

```
FROM centos:latest
MAINTAINER alec
ADD ["nginx-1.20.0.tar.gz", "Centos-8.repo", "nginx.conf", "/tmp/"]
RUN rm -rvf /etc/yum.repos.d/* && \
    mv -v /tmp/Centos-8.repo /etc/yum.repos.d/ && \
    yum install -y gcc gcc-c++ make pcre-devel openssl-devel zlib-devel && \
    mkdir -v /home/appadmin && \
    cd /tmp/nginx-1.20.0 && \
    ./configure --prefix=/home/appadmin/nginx-1.20.0 --with-http_ssl_module &&\
    make && make install && \
    ln -vs /home/appadmin/nginx-1.20.0 /home/appadmin/nginx && \
    mv -vf /tmp/nginx.conf /home/appadmin/nginx/conf && \
    rm -rvf /tmp/nginx-1.20.0
EXPOSE 80/tcp
WORKDIR /home/appadmin
CMD ["/bin/bash", "-c", "/home/appadmin/nginx/sbin/nginx"]
```

> nginx.conf

```
user  nobody;
worker_processes  1;
daemon off;
events {
    worker_connections  1024;
}
http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
    server {
        listen       80;
        server_name  localhost;
        location / {
            root   html;
            index  index.html index.htm;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}
```

构建

```
[root@localhost nginx]# docker build -f DockerFile -t my-nginx:v1.1 .
```

运行测试

```
[root@localhost nginx]# docker run -d -P --rm --name my-nginx my-nginx:v1.1

[root@localhost nginx]# docker ps
CONTAINER ID   IMAGE           COMMAND                  CREATED              STATUS              PORTS                                     NAMES
e5016b4c00bf   my-nginx:v1.1   "/bin/bash -c /home/…"   About a minute ago   Up About a minute   0.0.0.0:49153->80/tcp, :::49153->80/tcp   my-nginx

[root@localhost nginx]# curl http://127.0.0.1:49153
```





# 7 搭建私有Registry



## 7.1 通过registry镜像

配置docker信任仓库地址

```shell
vim /etc/docker/daemon.json
{
"registry-mirrors": ["https://rfj1yucr.mirror.aliyuncs.com"],
"insecure-registries":["my-registry.alec.com:5000"]
}

systemctl daemon-reload
systemctl restart docker
```



拉取运行 registry镜像

```
[root@localhost ~]# docker search registry
NAME                                 DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
registry                             The Docker Registry 2.0 implementation for s…   3283      [OK]       

[root@localhost ~]# docker pull registry

[root@localhost ~]# docker run -d --restart always -p 5000:5000 --name my-registry registry:latest
```

```
[root@localhost ~]# echo '127.0.0.1 my-registry.alec.com' >> /etc/hosts
```

修改my-nginx:1.1标签 为 私有仓库域名(ip):port/my-nginx:1.1

推送镜像到私有仓库

```
[root@localhost ~]# docker tag my-nginx:v1.1 my-registry.alec.com:5000/my-nginx:v1.1

[root@localhost ~]# docker push my-registry.alec.com:5000/my-nginx:v1.1
```

进入到私有仓库容器, 可以看到上传的镜像文件

```
[root@localhost ~]# docker exec -it my-registry /bin/sh

~ # ls /var/lib/registry/docker/registry/v2/repositories
my-nginx
```

删除my-registry.alec.com:5000/my-nginx:v1.1, 重新拉取

```
[root@localhost ~]# docker rmi my-registry.alec.com:5000/my-nginx:v1.1

[root@localhost ~]# docker pull my-registry.alec.com:5000/my-nginx:v1.1
```



## 7.2 通过docker-distribution软件构建私有registry



### 7.2.1 安装docker-registry (一个额外封装的docker registry)

```
[root@localregistry ~]# yum install docker-registry

[root@localregistry ~]# rpm -ql docker-distribution
    /etc/docker-distribution/registry/config.yml
    /usr/bin/registry
    /usr/lib/systemd/system/docker-distribution.service
    /usr/share/doc/docker-distribution-2.6.2
    /usr/share/doc/docker-distribution-2.6.2/AUTHORS
    /usr/share/doc/docker-distribution-2.6.2/CONTRIBUTING.md
    /usr/share/doc/docker-distribution-2.6.2/LICENSE
    /usr/share/doc/docker-distribution-2.6.2/MAINTAINERS
    /usr/share/doc/docker-distribution-2.6.2/README.md
    /var/lib/registry

[root@localregistry ~]# vim /etc/docker-distribution/registry/config.yml
        version: 0.1
        log:
        fields:
            service: registry
        storage:
            cache:
                layerinfo: inmemory
            filesystem:
                rootdirectory: /var/lib/registry        # 上传后的镜像的存储位置
        http:
            addr: :5000     # 绑定ip http的0.0.0.0::5000，如果是https需要改成443

[root@localregistry ~]# systemctl start docker-distribution
[root@localregistry ~]# systemctl enable docker-distribution

[root@localregistry ~]# netstat -antlp | grep 5000
tcp6       0      0 :::5000                 :::*                    LISTEN      44510/registry
```


### 7.2.2 推送镜像

```
1.  给镜像打上私有registry的标签 url和仓库间没有其他的层，代表使用顶层仓库
[root@localregistry ~]# docker tag lancehttpd:v0.2-7 localregistry.docker.com:5000/lancehttpd:v0.2-7

[root@localregistry ~]# docker image ls
    REPOSITORY                                 TAG                 IMAGE ID            CREATED             SIZE
    localregistry.docker.com:5000/lancehttpd   v0.2-7              7b49cf760ce2        4 hours ago         1.2MB


2. 在docker客户端修改docker配置，接受不安全的url
[root@localregistry ~]# vim /etc/docker/daemon.json
    {
    "registry-mirrors":["https://rfj1yucr.mirror.aliyuncs.com"],
    "insecure-registries":["localregistry.docker.com:5000"]
    }


3. 推送镜像
[root@localregistry ~]# docker push localregistry.docker.com:5000/lancehttpd:v0.2-7 
    The push refers to repository [localregistry.docker.com:5000/lancehttpd]
    0c5af919864c: Pushed 
    adab5d09ba79: Pushed 
    v0.2-7: digest: sha256:3ae5d2187d0ef3520a2b43174601dd9b8d87822ea1f19e5d7b8adb4dadda3448 size: 734

4. 推送后镜像在registry服务器的存储位置
[root@localregistry registry]# tree -L 6 /var/lib/registry
    /var/lib/registry
    └── docker
        └── registry
            └── v2
                ├── blobs
                │   └── sha256
                │       ├── 3a
                │       ├── 69
                │       ├── 7b
                │       └── 7e
                └── repositories
                    └── lancehttpd
                        ├── _layers
                        ├── _manifests
                        └── _uploads
```



### 7.2.3 拉取镜像

```
[root@localregistry registry]# docker pull localregistry.docker.com:5000/lancehttpd:v0.2-7 
    v0.2-7: Pulling from lancehttpd
    Digest: sha256:3ae5d2187d0ef3520a2b43174601dd9b8d87822ea1f19e5d7b8adb4dadda3448
    Status: Image is up to date for localregistry.docker.com:5000/lancehttpd:v0.2-7
```



## 7.3 部署harbor



# 8 资源限制

## 8.2 容量估算

```
存储估算:
	每个容器的大小
	容器数量 + 每天的日志增长量 + 程序文件
```





# 9 docker-compose

> https://docs.docker.com/compose/overview/



## 9.1 安装docker-compose

```
在有epel源的情况下：
[root@localregistry ~]# yum install -y docker-compose

或者执行下面的命令(速度很慢)：
[root@localregistry ~]# sudo curl -L "https://github.com/docker/compose/releases/download/1.23.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

```

## 9.2 docker-compose常用命令

```
docker-compose [-f <arg>...] [options] [COMMAND] [ARGS...]
选项：
        -f  指定compose配置文件的位置；当指定多个文件时，compose会将它们合并为一个配置；
            按照提供文件的顺序，后面的文件会被添加到前面的文件中；
```

### 9.2.1 docker-compose build

### 9.2.2 docker-compose config

```
检查compose配置文件语法；
```



