# Nginx 配置 https 证书示例

@ 折腾自己的网站，难免会遇到各种 nginx 的配置问题，经常会用到的无非就是 https 的配置，这里记录一下。

## 配置 https：

```nginx
server {
    listen 443 ssl;
    #配置HTTPS的默认访问端口为443。
    #如果未在此处配置HTTPS的默认访问端口，可能会造成Nginx无法启动。
    #如果您使用Nginx 1.15.0及以上版本，请使用listen 443 ssl代替listen 443和ssl on。
    server_name dabuguowoba.com wwww.dabuguowoba.com; #需要将yourdomain替换成证书绑定的域名。
    root html;
    index index.html index.htm;
    ssl_certificate cert/7764892_www.dabuguowoba.com.pem;
    ssl_certificate_key cert/7764892_www.dabuguowoba.com.key;
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    #表示使用的加密套件的类型。
    ssl_protocols TLSv1.1 TLSv1.2 TLSv1.3; #表示使用的TLS协议的类型。
    ssl_prefer_server_ciphers on;
    location / {
        root /home/www/;  #Web网站程序存放目录。
        index index.html index.htm;
    }
}
```

## http 跳转 https

```nginx
server {
    listen 80;
    server_name dabuguowoba.com www.dabuguowoba.com;
    rewrite ^(.*)$ https://$host$1; #将所有HTTP请求通过rewrite指令重定向到HTTPS。
}
```