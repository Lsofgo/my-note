**Ubuntu的软件包管理命令**

- dpkg 相当于 rpm
- apt-get 相当于 yum

在没有网络的情况下，要想安装软件包，并自动解决依赖问题，需要将事先准备好的deb包，上传到服务器并制作local repo；

**下载软件包**

```shell
sudo apt-get –d install vim
```

**打包软件**

包默认存放在/var/cache/apt/archives/目录下

```shell
cd /var/cache/apt/
sudo tar zcvf /home/pkgs.tar.gz archives/
```

**制作本地源**

```shell
# 把下载的软件包都放到 /home/pkgs
mkdir /home/pgks

#  建立Packages.gz包，里面记录了pakgs下面的软件包信息，包括依赖信息
dpkg-scanpackages $pack_dir /dev/null |gzip > /home/pgks/Packages.gz -r

sudo cp /etc/apt/sources.list{,-bak}  # 备份
sudo vi /etc/apt/sources.list         # 修改
deb file:/// /home/pkgs/              # 内容

# 刷新apt数据并允许不安全的源
sudo apt-get update ---allow-insecure-repositories

# 修复依赖关系
sudo apt-get install –f
```

**安装**

```shell
sudo apt-get install vim
```

