# mysqlrouter

## 环境

在vm的docker中使用`mysql router`实现`group replication 组复制`的读写分离和负载均衡  
由于官方的docker镜像需要和InnoDb Cluster一起使用,这里自己创建镜像  
`group replication 组复制`搭建:<https://www.jianshu.com/p/8187f72bcaa6>

## 目录结果

``` linux
/home/ysh/docker/router
├── Dockerfile
└── mysqlrouter.conf
```

## 创建镜像

使用的ubuntu作为基础镜像, 需要添加国内源  
Dockfile文件:

``` linux
FROM ubuntu:20.10
RUN sed -i s@/archive.ubuntu.com/@/mirrors.aliyun.com/@g /etc/apt/sources.list && \
    apt update && apt install -y mysql-router &&\
    rm -rf /var/lib/apt/lists/*
entrypoint ["mysqlrouter"] 
```

创建router镜像:

``` bash
docker build -t router .
```

## 配置文件

``` conf
[logger]                                                                                                                     
level = INFO

[routing:primary]
bind_address = 0.0.0.0
bind_port = 3300
# router容器跟group replication使用同一个docker网络
# 可以使用容器名代替ip地址
destinations = db1:3306
routing_strategy = first-available

[routing:secondary]
bind_address = 0.0.0.0
bind_port = 3301
destinations = db1:3306,db2:3306,db3:3306
routing_strategy = round-robin
```

* `routing:primary`和`routing:secondary`定义了两个路由配置
* bind_address 连接到mysqlrouter的机器的地址, 就是我们的应用在的机器, `0.0.0.0`表示所有的地址都可以连接mysqlrouter
* bind_port 绑定的端口
* destinations是mysqlrouter需要连接的mysql机器地址
* routing_strategy 路由策略
  * first-available: 使用第一个可用的连接,如果失败使用下一个连接,会循环判断直到没有可用的连接
  * round-robin: 每个新连接会连接到下一个可用的连接,轮询的负载均衡

## 启动容器

启动容器:  
这里映射了3300和3301两个端口,连接到了`mysql group replication`使用的docker网络中,自定义的mysqlrouter配置挂载到容器中

``` bash
docker run -d --name router -p 3300:3300 -p 3301:3301 --network mgr -v /home/ysh/docker/router:/etc/mysqlrouter router
```

然后再连接mysql服务器时, 只需使用router的端口即可, 使用`mysql workbench`连接3301端口, 每创建一个连接都会连接到下一个可用的mysql服务器上, 使用`select @@hostname;`查看当前连接到的mysql服务器的hostname就可以验证了

## router不能解决的问题

* 添加或删除mysql服务器时, router需要重启, 不过重启很快
* router连接数量有限制,且有单点失效问题,官方文档建议跟应用配置再同一台机器上
* 应用使用连接池时,由于连接池中的连接数量有限且长时间存在,如果有mysql服务器宕机了,该服务器上的连接会被router连接到其他可用的服务器上,该宕机服务器重新加入`group replication`后会收不到新的连接. 可以通过配置连接池中连接的存活时间,在连接死亡后再次创建连接router后就可以正常的负载到该服务器了
* mysql宕机后,正在进行的连接会失败,router没有重试的功能,需要应用自行实现,重试的包也有很多,例如:`spring retry`
