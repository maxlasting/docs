# 使用 yum 安装 redis

@ redis 作为常用的缓存数据库，其安装方法还是比较简单的，但是发现已经停止对 windows 系统的更新，这里记录下在 centos 上安装 redis 的方法。

1 下载 fedora 的 epel 仓库


```sh
yum install epel-release
```


2 安装redis数据库


```sh
yum install redis
```


3 安装完毕后，使用下面的命令启动redis服务


```sh
systemctl start redis.service
```



4 修改端口等配置项


```sh
vim /etc/redis.conf
```

end!