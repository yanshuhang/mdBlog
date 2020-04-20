# auto-offset-reset

## 官方文档解释

>**auto.offset.reset:** What to do when there is no initial offset in Kafka or if the current offset does not exist any more on the server (e.g. because that data has been deleted):  
**earliest:** automatically reset the offset to the earliest offset  
**latest:** automatically reset the offset to the latest offset  
**none:** throw exception to the consumer if no previous offset is found for the consumer's group  
**anything else:** throw exception to the consumer.

官方文档说得很清楚了：kafka中没有offset时，不论是什么原因，offset没了，这是auto.offset.reset配置就会起作用，

1. earliest：从最早的offset开始消费，就是partition的起始位置开始消费
2. latest：从最近的offset开始消费，就是新加入partition的消息才会被消费
3. none：报错

## 测试

最容易测试的方式就是在partition中预存放一些消息，然后新建一个consum group来消费这个partition。

测试步骤：创建两个不同组的消费者，分别设置为earliest和latest

1. 预先生产10条消息
2. 启动两个消费者，观察消息获取
3. 在不关闭消费者的情况下，继续生产10条消息
4. 观察消息获取

### 第一步写入消息

``` java
@Autowired
private KafkaTemplate kafkaTemplate;

@Test
void sendMsg() {
    for (int i = 0; i < 10; i++) {
        kafkaTemplate.send("test", "message" + i);
    }
}
```

### 第二步创建、启动消费者

创建两个消费者consumer1(earliest)、consumer2(lastest)，分别启动后观察到consumer1消费到10条消息，consumer2消费到0条消息

``` text
consumer1: partitions assigned: [test-0]
message0
message1
message2
message3
message4
message5
message6
message7
message8
message9
```

``` text
consumer2: partitions assigned: [test-0]
```

### 第三步继续生产消息

可以观察到两个消费者都消费了新的10条消息

``` text
message10
message11
message12
message13
message14
message15
message16
message17
message18
message19
```

### 第四步消费者关闭，再生产10条消息，启动消费者

此时在kafka服务器已经记录了消费者的offset，重启后两个消费者都从记录中的offset开始消费

``` text
message20
message21
message22
message23
message24
message25
message26
message27
message28
message29
```

## 总结

* 如果kafka服务器记录有消费者消费到的offset，那么消费者会从该offset开始消费
* 如果由于某些offset记录丢失了，此时auto-offset-reset就起了作用，earlist从头开始消费，latest从最新生产的消息开始消费
