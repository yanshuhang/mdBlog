# topic

主题，一个抽象的概念，是已发布消息的分类；producer(生产者)发布消息到某个topic，consumer(消费者)订阅具体topic获取消息。topic由partition组成，每个tipic可以分为1个或多个partition

![topic-partition](..\pic\log_anatomy.png)

## partition

tipic是抽象的概念，partition是存储消息的实体，是有序的、不变的记录序列，消息被连续的附加到这些序列中。  
一个topic的多个partition分布存储在多个broker中，所以kafka只提供了单个partition中消息的顺序性，topic中的多个partition之家没有顺序性保证

* record：消息在partiton中称为一个`record(记录)`
* offset：每个record被分配一个顺序的id，唯一标识partition中的每个记录

## producer

生产者将消息发布到具体的topic，具体消息记录在哪个partition中有多种负载平衡的方式，比如轮询或者根据传递的key来hash等等

## consumer

消费者必须要加入一个consumer group(消费者组)，某个topic，所有订阅它的consumer group都可以获得消息，但每个group中只有一个消费者获得消息，像是点对点队列和发布订阅模式的组合。

* 发布订阅：topic发布消息到所有订阅的consumer group
* 点对点队列：consumer group中只有一个消费者能获得该消息

实际存储消息的是partition，而且partition通常都是多个的。consumer实际上订阅的是partition，kafka会平均分配consumer和partition之间的数量对应，可以一个consumer对应多个partition，如果consumer数量比partition多，就会出现有consumer空闲的情况。

![partition-consumer](..\pic\log_anatomy01.png)
