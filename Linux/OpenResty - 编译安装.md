```shell
# 使用appadmin用户编译安装
tar -xf openresty-1.13.6.2.tar.gz -C /tmp/
cd /tmp/openresty-1.13.6.2/
./configure \
--with-cc-opt="-I/home/appadmin/openresty-1.13.6.2/include" \
--with-ld-opt="-L/home/appadmin/openresty-1.13.6.2/lib" \
--prefix=/home/appadmin/openresty-1.13.6.2 \
--with-http_ssl_module \
--with-stream \
--with-stream_ssl_module \
--with-http_realip_module \
--with-pcre \
--with-http_stub_status_module \
--with-http_v2_module
make -j8 && make install
cd /home/appadmin
ln -s /home/appadmin/openresty-1.13.6.2 /home/appadmin/openresty

# 拷贝openresty源码, 方便日后增加模块二次编译
cd /home/appadmin/openresty
mkdir src
mv -v /tmp/openresty-1.13.6.2 src/

# 复制nginx普通用户启动脚本到openresty
cd /home/appadmin/openresty
cp -r /tmp/nginx-1.13.6.2/admin ./
# 修改原nginx普通用户启动脚本
sed -i 's/nginx/openresty/g' admin/*.sh

# 拷贝启动脚本(root用户)
cp /tmp/nginx-1.13.6.2/nginx /etc/init.d/openresty
# 修改启动脚本(略)
# 可以用 vimdiff /etc/init.d/openresty /tmp/nginx-1.13.6.2/nginx 查看修改内容, 两次 :q! 退出

# 添加开机自启动(root用户)
chkconfig --add openresty
chkconfig openresty on
# 检查开机自启动(root用户)
chkconfig openresty --list
```

