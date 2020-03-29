# 单机多实例

将config目录下的server.properties文件复制两份：server-1.properties、server-2.properties，修改端口、borker.id、log.dirs  
加上server.properties中的broker.id=0，我们就有了3个broker

``` properties
# server-1.properties
broker.id=1
listeners=PLAINTEXT://:9093
log.dirs=C:\\tool\\kafka\\kafka-logs-1

# server-2.properties
broker.id=2
listeners=PLAINTEXT://:9094
log.dirs=C:\\tool\\kafka\\kafka-logs-2
```

启动zookeeper、3个broker，需要在不同的窗口内

``` text
// 启动zookeeper
bin\windows\zookeeper-server-start.bat config\zookeeper.properties

// 启动broker0-2
bin\windows\kafka-server-start.bat config\server.properties
bin\windows\kafka-server-start.bat config\server-1.properties
bin\windows\kafka-server-start.bat config\server-2.properties
```

创建topic

``` text
bin\windows\kafka-tapics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic my-replicated-topic
```

启动生产者、消费者

``` text
// producer
bin\windows\kafka-console-producer.bat --broker-list localhost:9092 --topic my-replicated-topic

//consumer
bin\windows\kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic my-replicated-topic
```

生产者发送消息，消费者能够收到

``` text
// producer
>this is a test message!

// consumer
this is a test message!
```

查看topic信息，这里第3行显示有0、1、2三个broker，Leader是2

``` text
bin\windows\kafka-topics.bat --describe --zookeeper localhost:2181 --topic my-replicated-topic
Topic: my-replicated-topic      PartitionCount: 1       ReplicationFactor: 3    Configs:
        Topic: my-replicated-topic      Partition: 0    Leader: 2       Replicas: 2,1,0 Isr: 2,1,0
```

在win10中查看broker.id=2的进程，然后杀掉模拟Leader挂掉的情况

``` text
wmic process where "caption = 'java.exe' and commandline like '%server-2.properties%'" get processid
ProcessId
5996

PS C:\tool\kafka> taskkill /pid 5996 /f
```

再次查看topic信息，Leader已更换为1，Isr同步中的broker只剩下了1和0

``` text
 bin\windows\kafka-topics.bat --describe --zookeeper localhost:2181 --topic my-replicated-topic
Topic: my-replicated-topic      PartitionCount: 1       ReplicationFactor: 3    Configs:
        Topic: my-replicated-topic      Partition: 0    Leader: 1       Replicas: 2,1,0 Isr: 1,0
```

消费者中有warn，node 2 无法连接，但是不影响继续接收消息

``` text
WARN [Consumer clientId=consumer-console-consumer-43351-1, groupId=console-consumer-43351] Connection to node 2 (/192.168.1.103:9094) could not be established. Broker may not be available. (org.apache.kafka.clients.NetworkClient)
```

``` text
// producer
>this is a test message!
>haha
>

// consumer
this is a test message!
haha
```
