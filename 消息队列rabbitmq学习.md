# 消息模型

消费者（consumer）订阅某个队列。生产者（producer）创建消息，然后发布到队列（queue）中，最后将消息发送到监听的消费者。

## 概念介绍

1. message：消息，生产者、消费者之间传递的内容
2. Publisher：消息的生产者，将消息发布到队列
3. Consumer：消息的消费者，监听某个队列
4. Queue：队列，消息的容器，接收生产者发送的消息，等待消费者消费消息
5. Exchange：交换器，接收消息，根据一定的规则将消费发布到队列
6. Binding：将交换器和队列安装路由规则绑定起来
7. Virtual host：虚拟主机，交换器、队列都在虚拟主机上，默认vhost是/
8. Broker：队列服务器实体

## 队列模式

1. 简单队列：一个生产者对应一个消费者
2. work模式：一个生产者对应多个消费者，一个消息只能有一个消费者获得
3. 使用Exchange交换器

## Exchange交换器类型

队列通过路由规则绑定到Exchange上，生产者发布通过Exchange发布消息，Exchange根据路由规则转发给队列

1. direct：消息发布到路由键完全匹配队列上
2. fanout：消息会发布到Exchange上绑定的所有队列
3. topic：模糊的绑定方式，* 匹配一个单词，# 匹配0个或多个字符，*、# 只能写在.号左右，且不能挨着字符，单词和单词之间需要用.隔开。

topic匹配规则示例：  
Routing key是user.log.#，#是匹配0个或多个字符，下面的都可以匹配：

``` text
user.log
user.log.info
user.log.a
user.log.info.login
```

Routing key是user.log.*， *匹配一个单词

``` text
user.log.info 可以匹配
user.log 不能匹配
user.log.info.login 二个单词，不能匹配
```

\*和#也可以放到前面，Routing key是*.action.#：action前面必须一个单词，后面0或多个

``` text
action 不符合
action.log 不符合
user.action 符合
user.action.log 符合
user.action.log.info 符合
user.log.action 不符合
```

springboot结合rabbitmq示例：网址
