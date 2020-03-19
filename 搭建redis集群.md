# 搭建redis集群

redis 5.0开启集群已经不需要ruby了，一个命令就可以了，简单了很多

## 配置多个redis

创建目录/etc/redis/cluster，在此目录下创建7000-7005 6个目录

``` linux
sudo mkdir 7000 7001 7002 7003 7004 7005
```

然后复制redis.conf文件到这6个目录，修改redis.conf，主要修改

```linux
# 修改下断开、开启集群配置
port 6379
pidfile /var/run/redis_6379.pid
logfile /var/log/redis_6379.log
dir /var/lib/redis/6379
cluster-enabled yes
cluster-config-file nodes-6379.conf
cluster-node-timeout 5000
```

启动这6个redis server

``` linux
sudo redis-server cluster/7000/redis.conf
sudo redis-server cluster/7001/redis.conf
sudo redis-server cluster/7002/redis.conf
sudo redis-server cluster/7003/redis.conf
sudo redis-server cluster/7004/redis.conf
sudo redis-server cluster/7005/redis.conf
```

查看redis server 启动成功

``` linux
ysh@DESKTOP-B40KVIO:/etc/redis$ ps -ef | grep redis
root      8910     1  0 13:13 ?        00:00:00 redis-server 127.0.0.1:7000 [cluster]
root      8917     1  0 13:14 ?        00:00:00 redis-server 127.0.0.1:7001 [cluster]
root      8923     1  0 13:14 ?        00:00:00 redis-server 127.0.0.1:7002 [cluster]
root      8929     1  0 13:14 ?        00:00:00 redis-server 127.0.0.1:7003 [cluster]
root      8935     1  0 13:14 ?        00:00:00 redis-server 127.0.0.1:7004 [cluster]
root      8941     1  0 13:14 ?        00:00:00 redis-server 127.0.0.1:7005 [cluster]
ysh       8946     8  0 13:14 tty1     00:00:00 grep --color=auto redis
```

## 开启集群

命令：

``` linux
redis-cli --cluster create 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005 --cluster-replicas 1
```

开启显示3个master、3个slave

``` linux
>>> Performing Cluster Check (using node 127.0.0.1:7000)
M: 13ab7020d3e23fe1ef440aff206f738704eda193 127.0.0.1:7000
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: b16edb551a6b4a76d26a44484ec88a1e98bd9b82 127.0.0.1:7002
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
M: 461e64ea88f7075291151c0556517536ef0db00e 127.0.0.1:7001
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 7741af4fc93f63fcb55ce8b198440028ac5b71cc 127.0.0.1:7003
   slots: (0 slots) slave
   replicates b16edb551a6b4a76d26a44484ec88a1e98bd9b82
S: c8800a0a0098e38a437d98931b0fce4d6917f516 127.0.0.1:7005
   slots: (0 slots) slave
   replicates 461e64ea88f7075291151c0556517536ef0db00e
S: 6ed7ace660ac2f9e317532d327f4584ce36d5a60 127.0.0.1:7004
   slots: (0 slots) slave
   replicates 13ab7020d3e23fe1ef440aff206f738704eda193
```

使用命令`redis-cli --cluster check 127.0.0.1:7000`可以查看所有server的状态

开启对应server的客户端，只需在正常命令之后加上端口号

``` linux
redis-cli -p 7000
```

在客户端里使用命令关闭server

``` linux
shutdown
```

在7000端口的server里添加一个key-value，可以看到在slave7004里已经自动同步了

``` linux
127.0.0.1:7000> set hello world
OK
127.0.0.1:7000> get hello
"world"
127.0.0.1:7000> exit
```

注意集群中slave查看key-value，需要在启动客户端时加上`-c`参数

``` linux
redis-cli -c -p 7004
127.0.0.1:7004> get hello
-> Redirected to slot [866] located at 127.0.0.1:7000
"world"
127.0.0.1:7000> exit
```

## 新增节点

按照上面新配置一个redis-server：7006  
命令：7006是要添加的节点，7000是集群里的一个节点(任意一个都可以)

### 命令

``` linux
redis-cli --cluster add-node 127.0.0.1:7006 127.0.0.1:7000
```

还可以在添加为slave节点指定master节点，如果未指定其为slave节点则默认为master节点(7006就是默认为master节点)

--cluster-slave 添加为slave节点 --cluster-master-id 指定其所属的master节点id，id就是
`M: b16edb551a6b4a76d26a44484ec88a1e98bd9b82 127.0.0.1:7002`中间的一串字符

``` linux
redis-cli --cluster add-node 127.0.0.1:7007 127.0.0.1:7000 --cluster-slave --cluster-master-id b16edb551a6b4a76d26a44484ec88a1e98bd9b82

```

### 分配slot

redis的集群是基于slot的，slot是存储数据的基本单位，每个master节点管理一部分的slot，新加进来的master节点并没有被分配slot，需要手动分配

``` linux
redis-cli --cluster reshard 127.0.0.1:7000 #输入任意一个集群中的节点都可以
```

需要移动多少slot，这里填写了2000个

``` linux
How many slots do you want to move (from 1 to 16384)? 2000
```

接收slot的节点ID，填写需要slot的节点id，这里就是刚刚加入的7006的ID

``` linux
What is the receiving node ID? 6e9842db5776361a37feeda650cf2c74b0b49c28
```

slot的来源节点，all就是从所有master节点中平均移动slot，也可以指定ID

``` linux
Please enter all the source node IDs.
  Type 'all' to use all the nodes as source nodes for the hash slots.
  Type 'done' once you entered all the source nodes IDs.
Source node #1: all
```

在打印一堆移动的slot之后，出现选项：是否执行，yes就行了

``` linux
Do you want to proceed with the proposed reshard plan (yes/no)? yes
```

check查看结果,7006拥有1999个slots，还多了一个key，应该是拥有key的slot被移动到了7006节点

``` linux
redis-cli --cluster check 127.0.0.1:7000
127.0.0.1:7000 (13ab7020...) -> 2 keys | 4795 slots | 1 slaves.
127.0.0.1:7002 (b16edb55...) -> 0 keys | 4795 slots | 2 slaves.
127.0.0.1:7001 (461e64ea...) -> 0 keys | 4795 slots | 1 slaves.
127.0.0.1:7006 (6e9842db...) -> 1 keys | 1999 slots | 1 slaves.
```

## 删除节点

7008是7002的一个slave节点，直接命令即可

格式：`del-node       host:port node_id`

``` linux
redis-cli --cluster del-node 127.0.0.1:7008 2b5a69b300f81297a0b0b2469719caecfaa21892
>>> Removing node 2b5a69b300f81297a0b0b2469719caecfaa21892 from cluster 127.0.0.1:7008
>>> Sending CLUSTER FORGET messages to the cluster...
>>> SHUTDOWN the node.
```

删除7007，7007是7006的slave节点，7006中存储有一个key。也是直接删除

``` linux
redis-cli --cluster del-node 127.0.0.1:7007 f4f4d04de89725060b2ffa5378040c61ba3d4572
>>> Removing node f4f4d04de89725060b2ffa5378040c61ba3d4572 from cluster 127.0.0.1:7007
>>> Sending CLUSTER FORGET messages to the cluster...
>>> SHUTDOWN the node.
```

删除7006，7006是master节点，拥有的slot需要释放

``` linux
redis-cli --cluster del-node 127.0.0.1:7006 6e9842db5776361a37feeda650cf2c74b0b49c28
>>> Removing node 6e9842db5776361a37feeda650cf2c74b0b49c28 from cluster 127.0.0.1:7006
[ERR] Node 127.0.0.1:7006 is not empty! Reshard data away and try again.
```

需要将7006的slot转移动其他的master节点，也是使用`redis-cli --cluster reshard 127.0.0.1:7000`命令，不过移出的节点id是7006的id，移入的节点id可以填写任一master节点，将7006所有的节点移除即可
