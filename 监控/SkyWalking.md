Skywalking主要包含三点:

- 服务(Service), 整个对外提供的业务系统;
- 端点(Endpoint), 系统的API接口;
- 实例(Instance), 系统的HA/LB的各个节点;

# 安装部署

虚拟机配置: 2c4g

**放开linux文件数限制**

```shell
vim /etc/security/limits.conf
*       soft    nofile  65535
*       hard    nofile  65535
*       soft    nproc   4096
*       hard    nproc   4096
# 保存后重新登录shell
```

**修改内存权限大小**

Elasticsearch需要开辟一个6535字节以上空间的虚拟内存, Linux默认不允许任何用户和应用程序直接开辟这么大的虚拟内存;

```shell
vim /etc/sysctl.conf
vm.max_map_count=262144
# 生效配置
sysctl -p
```

**安装JDK**

```shell
yum install java-11-openjdk
vim /etc/profile
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-11.0.8.10-1.el7.x86_64
export PATH=$PATH:$JAVA_HOME/bin

source /etc/profile
```

**安装elasticsearch**

```shell
tar -xf elasticsearch-7.10.2-linux-x86_64.tar.gz -C /usr/local/
cd /usr/local/elasticsearch-7.10.2
```

**修改elasticsearch配置, 开启远程访问**

```shell
vim config/elasticsearch.yml
cluster.name: elastic
node.name: node-1
network.host: 0.0.0.0
discovery.seed_hosts: ["127.0.0.1"]
cluster.initial_master_nodes: ["node-1"]
```

**创建es启动用户, elasticsearch不能使用root用户启动**

```shell
useradd es
chown es.es -R /usr/local/elasticsearch-7.10.2
```

**启动elasticsearch**

```shell
su - es
cd /usr/local/elasticsearch-7.10.2
./bin/elasticsearch -d
```

浏览器请求http://ip:9200, 得到以下信息

```json
{
  "name" : "localhost.localdomian",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "WwjUlXaYRtaCtC7jPL_QuQ",
  "version" : {
    "number" : "7.9.3",
    "build_flavor" : "default",
    "build_type" : "rpm",
    "build_hash" : "c4138e51121ef06a6404866cddc601906fe5c868",
    "build_date" : "2020-10-16T10:36:16.141335Z",
    "build_snapshot" : false,
    "lucene_version" : "8.6.2",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

**安装skywalking**

```shell
tar -xf apache-skywalking-apm-es7-8.3.0.tar.gz -C /usr/local/
cd /usr/local/apache-skywalking-apm-bin-es7/
```

**skywalking目录说明**

| agent      | 探针文件夹         |
| ---------- | ------------------ |
| bin        | 启动脚本           |
| config     | 配置               |
| LICENSE    |                    |
| licenses   |                    |
| NOTICE     |                    |
| oap-libs   |                    |
| README.txt |                    |
| tools      |                    |
| webapp     | ui jar包和配置文件 |

**配置skywalking数据库连接**

```shell
vim config/application.yml
storage:
  selector: ${SW_STORAGE:elasticsearch7}
  ...
  ...
  elasticsearch7:
    nameSpace: ${SW_NAMESPACE:"elastic"}
```

**启动skywalking**

```shell
./bin/startup.sh
SkyWalking OAP started successfully!
SkyWalking Web Application started successfully!
```

![image-20210728102139002](media/image-20210728102139002.png)

