### pip 指定源

```shell
pip install -r requirements.txt -i https://repos.taikangcloud.com/repository/api/pypi/pypi/simple
```

### npm指定源

```shell
npm --registry https://registry.npm.taobao.org install <package name>
```

### 服务器抓包

```shell
ls -d /data_tkocr_share ||\
mkdir -v /data_tkocr_share &&\
yum install -y nfs-utils &&\
echo >> /etc/fstab &&\
echo -e 'a1.nas.tkuat.com:/sdsfs/online-ocr-share\t/data_tkocr_share\tnfs\tdefaults\t0 0' >> /etc/fstab &&\
mount -a &&\
df -h &&\
ls -ld /data_tkocr_share
```

### nfs挂载

```shell
ls -d /data_tkocr_share ||\
mkdir -v /data_tkocr_share &&\
yum install -y nfs-utils &&\
echo >> /etc/fstab &&\
echo -e 'a1.nas.tkuat.com:/sdsfs/online-ocr-share\t/data_tkocr_share\tnfs\tdefaults\t0 0' >> /etc/fstab &&\
mount -a &&\
df -h &&\
ls -ld /data_tkocr_share
```

### cifs挂载

```shell
mount -t cifs //10.129.191.21/cssta_ingest_bj /cssta_ingest_bj -o username='cssta',password='密码',domain='BJYG-3',sec='ntlm',uid='appadmin',gid='appadmin'
```

### LVM常用

```shell
# 创建pv 物理卷(后面目录文件/dev/sdc根据硬盘修改)
pvcreate /dev/vdc
# 创建vg卷组
vgcreate datavg /dev/vdc
# 创建lv 逻辑卷
lvcreate -l 100%VG -n datalv datavg

mkfs.xfs /dev/datavg/datalv
blkid /dev/datavg/datalv
```

```shell
# 扩容
vgextend datavg /dev/vdd 
lvextend -l 100%VG -n /dev/datavg/datalv01
xfs_growfs /dev/datavg/datalv01
```

```shell
DEVICE=/dev/vdb
MOUNT_POINT=/data
VG=datavg
LV=datalv

ls /data && df -h /data || mkdir -v /data

sleep 5 && \
 \
pvcreate $DEVICE && \
vgcreate $VG $DEVICE && \
lvcreate -l 100%VG -n $LV $VG && \
mkfs.xfs /dev/datavg/datalv && \
UUID=$(blkid /dev/datavg/datalv | awk '{print $2}') && \
echo -e "\n${UUID}\t${MOUNT_POINT}\txfs\tdefaults\t0 0" >> /etc/fstab && \
mount -a
```



### curl

```shell
curl -w "$(date +"%Y-%m-%d %H:%M:%S"),%{time_namelookup},%{time_connect},%{time_starttransfer},%{time_total}" https://www.jd.com -s -o /dev/null; echo ",$?"
```

### python http server

```shell
# python2
python -m SimpleHTTPServer 8080
# python3
python -m http.server 8080
```

### useradmin

```shell
groupadd -g 20000 useradmin && \
useradd -u 20000 -g 20000 useradmin && \
echo <密码> | passwd useradmin --stdin && \
echo 'useradmin ALL=NOPASSWD: ALL' >> /etc/sudoers.d/useradmin && \
chmod 400 /etc/sudoers.d/useradmin && \
history -c
```

### 普通用户rc.local

```shell
chmod u+x /etc/rc.local
echo 'su - appadmin -c /bin/bash /etc/rc.local.appadmin' >> /etc/rc.local
 
touch /etc/rc.local.appadmin
chown appadmin.appadmin /etc/rc.local.appadmin
chmod u+x /etc/rc.local.appadmin
```

