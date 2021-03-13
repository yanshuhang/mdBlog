# 搭建使用GTID的mysql主从复制

## 环境

在虚拟机中使用docker-compose搭建一主两从mysql实例

## mysql配置文件

### mysql source服务器配置

开启gtid模式, binlog-format必须使用row

``` conf
[mysqld]                                                                          server-id = 1
log-bin = source-binlog
gtid-mode=on
enforce-gtid-consistency=on
binlog-format=row
default_authentication_plugin=mysql_native_password

[client]
# 客户端可以显示中文
default-character-set = utf8
```

### mysql replica服务器配置

配置两份server-id为2和3

``` conf
[mysqld]                                                                          server-id = 2
gtid-mode=on
enforce-gtid-consistency=on

log-bin=replica-binlog
log-slave-updates=1
binlog-format=row
skip-slave-start=1
default_authentication_plugin=mysql_native_password

[client]
# 客户端可以显示中文
default-character-set = utf8
```

## docker-compose配置

* 创建目录结构如下: conf文件夹放置配置文件, data文件夹作为容器的数据挂载

``` tree
mysql-replication/
├── docker-compose.yml
├── replica1
│   ├── conf
│   └── data
├── replica2
│   ├── conf
│   └── data
└── source
    ├── conf
    └── data
```

* docker-compose配置文件

``` yml
version: '3'

# 网络
networks:
  mysql-net:
    external: true

services:
  source-db:
    image: mysql
    container_name: source-db
    # 挂载配置文件和数据
    volumes:
      - ./source/conf:/etc/mysql/conf.d
      - ./source/data:/var/lib/mysql
    networks:
      - mysql-net
    ports:
      - 3307:3306
    environment:
      - MYSQL_ALLOW_EMPTY_PASSWORD=YES

  replica-db1:
    image: mysql
    container_name: replica-db1
    volumes:
      - ./replica1/conf:/etc/mysql/conf.d
      - ./replica1/data:/var/lib/mysql
    networks:
      - mysql-net
    ports:
      - 3308:3306
    environment:
      - MYSQL_ALLOW_EMPTY_PASSWORD=yes

  replica-db2:
    image: mysql
    container_name: replica-db2
    volumes:
      - ./replica2/conf:/etc/mysql/conf.d
      - ./replica2/data:/var/lib/mysql
    networks:
      - mysql-net
    ports:
      - 3309:3306
    environment:
      - MYSQL_ALLOW_EMPTY_PASSWORD=yes
```

* 启动3个mysql实例

``` bash
docker-compose up -d
```

## 配置mysql主从

* 进入source-db服务器中, 创建用户, 用于replica服务器连接

``` sql
# 创建了用户:repl, 设置密码
mysql> CREATE USER 'repl'@'%' IDENTIFIED WITH BY 'password';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
```

* 进入两个replica-db服务器中, 连接到source-db, 不需要指定binlog文件和position了

``` sql
# host就是容器名, docker网络会自动解析, 使用的端口是3306, 不是容器对外部开放的端口
mysql> CHANGE MASTER TO
    ->     MASTER_HOST='source',                  
    ->     MASTER_USER='repl',                      
    ->     MASTER_PASSWORD='password',             
    ->     MASTER_AUTO_POSITION = 1;       
```

* 然后开启主从复制

``` sql
mysql> start slave;
Query OK, 0 rows affected (0.01 sec)

# 查看信息, Slave_IO_Running和Slave_SQL_Running两个都是yes即连接成功
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: source
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: binlog.000002
          Read_Master_Log_Pos: 972
               Relay_Log_File: 749ff0e111d1-relay-bin.000002
                Relay_Log_Pos: 1137
        Relay_Master_Log_File: binlog.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
```
