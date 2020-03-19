# 搭建redis集群

## 安装redis

环境为win10 wsl ubuntu18.04，可以直接使用apt-get安装，但是redis是4.0版本，搭建集群需要使用到ruby，有些麻烦，redis5.0搭建集群可以使用自有的命令比较简单，这里安装的是redis5.0.5版本  
提示：wsl最好更换下载源，提升网络速度

### 安装编译器gcc

``` linux
sudo apt-get install gcc
```

### 下载安装redis5.0.5

目前在官网看到的最新版本，解压后移动到/usr/redis目录：

``` linux
sudo wget http://download.redis.io/releases/redis-5.0.5.tar.gz
sudo tar xzf redis-5.0.5.tar.gz
sudo mv redis-5.0.5 /usr/redis
```

在/usr/redis目录下使用make命令

``` linux
sudo make
sudo make install
```

如果提示没有make命令，可以使用以下命令安装

``` linux
sudo apt-get install make
```

之后可以把redis.conf文件移动到/etc/redis目录下，如果是apt-get安装的这里是默认有的

``` linux
sudo cp /usr/redis/redis.conf /etc/redis
```

修改redis为守护进程可以后台运行:在/etc/redis/redis.conf文件中修改daemonize为yes

``` linux
daemonize yes
```

启动redis-server

``` linux
sudo redis-server /etc/redis/redis.conf
```

启动客户端redis-cli

``` linux
redis-cli
```
