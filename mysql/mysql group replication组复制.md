# mysql group replication 组复制

## 环境

虚拟机中使用docker搭建mysql实例, 单主模式, 使用的是最新的mysql镜像版本:8.0.21, 按照官方文档弄的也踩了不少坑, 下面的配置有些跟官方文档有些不一样

## mysql配置

``` conf
[mysqld]
disabled_storage_engines="MyISAM,BLACKHOLE,FEDERATED,ARCHIVE,MEMORY"

# 每个实例配置需要配置成不同的id
server_id=1

gtid-mode=ON
enforce_gtid_consistency=ON
binlog_checksum=NONE
log_bin=binlog
log_slave_updates=ON
binlog_format=ROW
default_authentication_plugin=mysql_native_password

# 不加loose前缀, mysql较新的版本初始化启动时会失败
loose-plugin_load_add='group_replication.so'
# 网上生成的uuid
loose-group_replication_group_name="a31e4bde-840c-11eb-8dcd-0242ac130003"
loose-group_replication_start_on_boot=off
# db1是本MySQL实例的容器名, 同一个docker bridge网络中的容器可以使用容器名互联
loose-group_replication_local_address= "db1:33061"
# 这里填写已经在group中运行的实例的地址, 非主服务器使用, 填写主服务器地址即可
loose-group_replication_group_seeds= "db1:33061"
loose-group_replication_bootstrap_group=off

[client]
default-character-set = utf8
```

## docker-compose配置

文件结构如下: conf文件夹放置配置文件, data文件夹作为容器的数据挂载

``` linux
.
├── db1
│   ├── conf
│   └── data
├── db2
│   ├── conf
│   └── data
├── db3
│   ├── conf
│   └── data
└── docker-compose.yml
```

docker-compose配置文件:

``` yml
version:'3'                                                                                                                              
networks:
  mgr:
    name: mgr-net

services:
  db1:
    image: mysql
    container_name: db1
    volumes:
      - ./db1/conf:/etc/mysql/conf.d
      - ./db1/data:/var/lib/mysql
    ports:
      - 3310:3306
    environment:
      - MYSQL_ALLOW_EMPTY_PASSWORD=yes
  db2:
    image: mysql
    container_name: db2
    volumes:
      - ./db2/conf:/etc/mysql/conf.d
      - ./db2/data:/var/lib/mysql
    ports:
      - 3311:3306
    environment:
      - MYSQL_ALLOW_EMPTY_PASSWORD=yes
  db3:
    image: mysql
    container_name: db3
    volumes:
      - ./db3/conf:/etc/mysql/conf.d
      - ./db3/data:/var/lib/mysql
    ports:
      - 3312:3306
    environment:
      - MYSQL_ALLOW_EMPTY_PASSWORD=yes
```

## 启动mgr组复制

* 启动db1 db2 db3三个mysql实例

``` bash
docker-compose up -d
```

* 分别进入这三个容器中执行命令:  
  进入命令: `docker exec -it db1 mysql`  
  创建用户用于分布式恢复

``` sql
SET SQL_LOG_BIN=0;
CREATE USER rpl_user@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO rpl_user@'%';
GRANT BACKUP_ADMIN ON *.* TO rpl_user@'%';
FLUSH PRIVILEGES;
SET SQL_LOG_BIN=1;
CHANGE MASTER TO MASTER_USER='rpl_user', MASTER_PASSWORD='password' FOR CHANNEL 'group_replication_recovery';
```

* 在主服务器中引导组复制  
在db1 db2 db3 中选择一个作为主服务器,执行以下命令

``` sql
SET GLOBAL group_replication_bootstrap_group=ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group=OFF;
```

* 创建数据用于验证组复制  
在主服务器中创建数据:

``` sql
CREATE DATABASE test;
USE test;
CREATE TABLE t1 (c1 INT PRIMARY KEY NOT NULL AUTO_INCREMENT, c2 VARCHAR(10) NOT NULL);
INSERT INTO t1 VALUES (1, 'AAA');
```

* 从服务器加入到复制组中  
这里就比较简单了, 一条命令就可以了

``` sql
START GROUP_REPLICATION;
```

如果上面的命令报错:`The member contains transactions not present in the group`
可以执行下面的命令进行清理 (尝试部署了很多次这里每次都会报错, 官方文档上也没有说明)

```sql
reset master
```

然后就可以在从服务器中查看到在主服务器中创建的数据了, 如果继续新加服务器到组中, 继续上面的操作即可

## 查看复制组中的信息

```sql
mysql> select member_id, member_state, member_role from performance_schema.replication_group_members;
+--------------------------------------+--------------+-------------+
| member_id                            | member_state | member_role |
+--------------------------------------+--------------+-------------+
| 2de181db-847e-11eb-9524-0242ac190004 | ONLINE       | SECONDARY   |
| 2de2e0f1-847e-11eb-840d-0242ac190003 | ONLINE       | PRIMARY     |
| 2de387b8-847e-11eb-9846-0242ac190002 | ONLINE       | SECONDARY   |
+--------------------------------------+--------------+-------------+
```

## 其他注意

复制组中的服务器在重启之后, 需要手动命令恢复组  

``` sql
# 主服务器
SET GLOBAL group_replication_bootstrap_group=ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group=OFF;

# 从服务器
START GROUP_REPLICATION;
```
