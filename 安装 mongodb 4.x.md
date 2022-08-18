# 安装 mongodb 4.x

@ mongodb 作为一个前端必学的数据库，具有简单好用的特性，尤其配合 mongoose 使用，这里记录以下 各个系统下 安装 mongodb 的过程，方便日后查阅。

## 使用 yum 安装

首先增加一个源配置文件：

```shell
vim /etc/yum.repos.d/mongodb-org-4.x.repo
```

已当前为止最新版 4.2 为例：

```shell
[mongodb-org-4.4]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/7/mongodb-org/4.4/x86_64/
gpgcheck=0
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-4.4.asc
```

之后验证一下：

```shell
yum list | grep mongodb
```

然后安装：

```shell
yum -y install mongodb-org
```


## 在 Mac 上使用 brew 安装

最近使用 brew 安装 mongodb，出现如下错误：

```shell
Error: No available formula with the name ‘mongodb’
```

原因是 mongodb 不再是开源的了，并且已经从Homebrew中移除。

解决：设定一下 brew ：

```shell
brew tap mongodb/brew
```

然后安装：
shell
```
brew install mongodb-community
```

最后启动：

```shell
brew services start mongodb-community
```


## 在 windows 上安装

下载文件：首先在mongodb的官方网站上下载最新版本的mongodb安装程序，下载网址：[MongoDB Community Download | MongoDB](https://www.mongodb.com/try/download/community?tck=docs_server)

![](https://cdn.maxlasting.com/doc-assets/202208182304833.png);


下载好后进行安装，放到对应的目录进行解压缩，并新建**data**文件夹，在**data**目录下面新建**db**文件夹，以及配置好 `mongod.conf`

![](https://cdn.maxlasting.com/doc-assets/202208182306571.png);

```conf
# mongod.conf

# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# where to write logging data.
systemLog:
    destination: file
    logAppend: true
    path: D:\devtool\mongodb\data\mongod.log

# Where and how to store data.
storage:
    dbPath: D:\devtool\mongodb\data\db
#  engine:
#  wiredTiger:
# how the process runs
# processManagement:
#     fork: true  # fork and run in background
#     pidFilePath: /var/run/mongodb/mongod.pid  # location of pidfile
#     timeZoneInfo: /usr/share/zoneinfo

# network interfaces
net:
    port: 27017
    bindIp: 0.0.0.0  # Enter 0.0.0.0,:: to bind to all IPv4 and IPv6 addresses or, alternatively, use the net.bindIpAll setting.


security:
    authorization: enabled #开启密码

#operationProfiling:

#replication:

#sharding:
```

打开 windows 环境变量设置，添加 mongo 的 bin 目录到环境变量中：

![](https://cdn.maxlasting.com/doc-assets/202208182309209.png)


之后就可以在终端中输入`mongod`来测试了一下了，这里可以添加到 windows 服务来自启动：

```shell
mongod --config "C:\MongoDB\server\3.6\mongo.conf"  --install --serviceName "Mongodb"
```

通过以下命令来启动和停止：

```shell
net start Mongodb

net stop Mongodb
```

end!