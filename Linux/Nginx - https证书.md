> 免费证书网站

```
https://csr.chinassl.net/
https://myssl.com/csr_create.html
```



**签发根证书私钥 和 证书请求文件**

```
openssl req -new -SHA256 -newkey rsa:2048 -nodes \
	-keyout www.alec.com.key \
	-out www.alec.com.csr \
	-subj "/C=CN/ST=BJ/L=BJ/O=alec/OU=/CN=www.alec.com" 
```

**利用私钥和CSR，自签根证书(pem和crt可以改文件后缀实现)**

```
openssl x509 -req \
	-in www.alec.com.csr \
	-signkey www.alec.com.key \
	-days 36500 \
	-out www.alec.com.pem
```

**配置nginx**

```
server {
    listen       443 ssl;
    server_name  www.alec.com;

    ssl_certificate      www.alec.com.pem;
    ssl_certificate_key  www.alec.com.key;
...
```

**curl请求**

- 忽略证书	    curl --insecure https://127.0.0.1
- 带证书请求    curl -Iv https://www.alec.com --cacert www.alec.com.pem

