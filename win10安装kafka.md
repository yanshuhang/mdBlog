# 下载

官网 <https://kafka.apache.org/downloads>

Scala版本随便选一个就行了，没影响

## 解压

tgz文件解压出tar文件再解压就是kafka文件夹了，文件夹随便放个地方就可以了，注意不要放到`C:\Program Files`里，路径里有空格启动时会报错，自己创建的目录名称里也千万不要加空格。

文件夹里就是这些文件，data和kafka-logs不是原有的，自己创建的存放data数据和log记录的文件夹。  
bin\windows目录下存放的是windows可以用的命令，包括启动和关闭kafka、zookeeper  
config目录下存放的是配置文件

## 修改配置

* 修改zookeeper.properties，把`dataDir`改为自己创建的目录

``` properties
dataDir=C:\\tool\\kafka\\data
```

* 修改server.properties，把`log.dirs`改为自己创建的目录

``` properties
log.dirs=C:\\tool\\kafka\\kafka-logs
```

## 启动

先启动zookeeper再启动kafka，关闭时先关闭kafka，不然有可能下次启动kafka时报错  
下面的操作都是再kafka目录下进行的

* 启动zookeeper，启动会不要关闭

``` text
PS C:\Users\ysh> cd c:\tool\kafka
PS C:\tool\kafka> .\bin\windows\zookeeper-server-start.bat .\config\zookeeper.properties
```

* 启动kafka，新开个窗口启动kafka

``` text
PS C:\Users\ysh> cd c:\tool\kafka
PS C:\tool\kafka> .\bin\windows\kafka-server-start.bat .\config\server.properties
```

* 创建topic

``` text
.\bin\windows\kafka-topics.bat --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test
```

* 生产、消费，需要新开两个窗口

``` text
// 启动生产者
.\bin\windows\kafka-console-producer.bat --broker-list localhost:9092 --topic test

// 启动消费者
.\bin\windows\kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic test
```

在生产者窗口输入的消息，在消费者窗口能够显示：
