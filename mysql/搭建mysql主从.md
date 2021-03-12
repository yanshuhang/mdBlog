# 配置mysql主从

## 环境

在虚拟机中使用docker搭建mysql主从

## 配置步骤

### 使用docker启动两个mysql实例

创建自定义网络, mysql从服务器连接主服务器时可以使用主服务器的容器名称.

``` bash
docker network create mysql-net
```

分别创建两个mysql容器:主-source,从-replica, 连接到mysql-net网络, 这里为了方便没有设置root密码.

``` bash
# source服务器
docker run -d -p 3310:3306 -e MYSQL_ALLOW_EMPTY_PASSWORD=yes --name source --network mysql mysql
```

``` bash
# replica服务器
docker run -d -p 3311:3306 -e MYSQL_ALLOW_EMPTY_PASSWORD=yes --name replica --network mysql mysql
```

### 进入mysql实例中命令开启主从

* 进入source容器MySQL实例中, 修改`server-id`, 默认`server-id`是1, 需要主从之间不同即可.

``` bash
# 进入source容器
docker exec -it source mysql

# 修改server-id
mysql> SET GLOBAL server_id = 2;
```

* 查看source容器的`bin-log`信息, 使用命令`show master status`, 记录下flie和position,在配置replica容器时需要使用到.

``` sql
mysql> SHOW MASTER STATUS;
+---------------+----------+--------------+------------------+-------------------+
| File          | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+---------------+----------+--------------+------------------+-------------------+
| binlog.000002 |      156 |              |                  |                   |
+---------------+----------+--------------+------------------+-------------------+
```

* 在source服务器中创建用户, 该用户只用于replica服务器连接source服务器

``` sql
# 创建了用户:repl, 设置密码
mysql> CREATE USER 'repl'@'%' IDENTIFIED WITH 'mysql_native_password' BY 'password';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
```

* 在replica服务器中, 修改`server-id`

``` bash
# 进入source容器
docker exec -it replica mysql

# 修改server-id
mysql> SET GLOBAL server_id = 3;
```

* 在replica服务器中, 连接source服务器

``` sql
# host就是容器名, docker网络会自动解析, 使用的端口是3306, 不是容器对外部开放的3310
mysql> CHANGE MASTER TO
    ->     MASTER_HOST='source',                    # 可以使用容器名
    ->     MASTER_USER='repl',                      # 上一步创建的用户名
    ->     MASTER_PASSWORD='password',              # 密码 
    ->     MASTER_LOG_FILE='bin-log-File',          # binlog.000002
    ->     MASTER_LOG_POS=bin-log-position;         # 156
```

* 然后在replica服务器中输入命令: start slave即可

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

### 简单验证

在source服务器中创建数据库test和表格t1,插入两条数据;
在replica服务器中查看到, 说明主从复制已经搭建成功了

``` sql
mysql> select * From t1;
+----+
| id |
+----+
|  1 |
|  2 |
+----+
```
