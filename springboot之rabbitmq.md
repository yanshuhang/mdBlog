# rabbitmq简介

rabbitmq简单介绍：网址

## 包、配置

可以在创建springboot项目时在message选项中勾选rabbitmq，也可以手动添加包

``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

springboot配置application.yml，需要指定virtual-host，这里就用了默认的

``` yml
spring:
  rabbitmq:
    addresses: 127.0.0.1:5672
    username: guest
    password: guest
    virtual-host: /
```

rabbitmq配置，替换默认的MessageConverter，可以将对象自动序列化化为json发布到队列，监听队列获取的消息也会自动发序列化为对象的类型

``` java
@Configuration
public class RabbitConfig {
    /**
     * 替换org.springframework.amqp.support.converter.MessageConverter
     * 在发送和接收消息时使用json格式处理pojo
     */
    @Bean
    public MessageConverter jsonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }
}
```

## 准备工作

传输的实体类

``` java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class User {
    private int id;
    private String name;
}
```

## 代码使用

### 简单队列

生产者Sender发送消息到指定的队列，消费者Receiver监听该队列  
生产者使用`rabbitTemplate`发布消息，`convertAndSend`方法接受队列名称和传输的对象  
消费者中使用`RabbitListener`监听队列，使用queuesToDeclare = @Queue("rabbit")动态创建队列：rabbit  

``` java
@Component
public class Sender {
    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void send(String queue, User user) {
        rabbitTemplate.convertAndSend(queue, user);
    }
}

@Slf4j
@Component
public class Receiver {
    @RabbitListener(queuesToDeclare = @Queue("rabbit"))
    public void getMsg(User user) {
        log.info("get message {}", user);
    }
}
```

测试结果，sender向rabbit队列发送一个user对象

``` java
@SpringBootTest
class ReceiverTest {

    @Autowired
    private Sender sender;

    @Test
    void getMsg() throws InterruptedException {
            sender.send("rabbit", new User(1, "tom"));
            Thread.sleep(1000);
    }
}
```

Receiver监听到rabbit队列，并取出消息user，输出显示

``` text
get message: User(id=1, name=tom)
```

### 使用多个消费者

Receiver中增加一个监听队列的方法

``` java
@RabbitListener(queues = "rabbit")
public void getMsg(User user) {
    log.info("use another Listener get message: {} ", user);
}
```

测试方法：Sender向队列发送5个user，一个消息只能有一个消费者获得

``` java
@Test
void twoReceiver() throws InterruptedException {
    for (int i = 0; i < 5; i++) {
        String randName = RandomStringUtils.random(5, 97, 122, true, false);
        User user = new User(i, randName);
        sender.send("rabbit", user);
    }
    Thread.sleep(1000);
}
```

``` text
get message: User(id=1, name=hrsrw) with Receiver02
get message: User(id=0, name=hhovn) with Receiver01
get message: User(id=3, name=sfiyg) with Receiver02
get message: User(id=2, name=ahthc) with Receiver01
get message: User(id=4, name=efdhg) with Receiver01
```

### 使用Exchange

#### direct模式

在Receiver类中增加方法，使用注解将队列`queueA`绑定到交换器`exchange_direct`上，路由key可以指定一个或多个。  
Sender类增加send方法，发送消息给Exchange使用路由key，消息不是直接发给队列，而是通过Exchange转发  
测试方法中分别传入两个key发送user对象

``` java
// Receiver类
@RabbitListener(bindings = @QueueBinding(
        value = @Queue("queueA"),
        exchange = @Exchange(value = "exchange_direct", type = ExchangeTypes.DIRECT),
        key = {"direct", "exchange"}
))
public void withExchange(User user) {
    log.info("get message: {} with directExchange", user);
}

// Sender类
public void send(String exchange, String key, User user) {
    rabbitTemplate.convertAndSend(exchange, key, user);
}

// 测试类
@Test
void directExchange() throws InterruptedException {
    sender.send("exchange_direct", "direct", new User(2, "jack"));
    sender.send("exchange_direct", "exchange", new User(3, "larry"));
    Thread.sleep(1000);
}
```

测试结果，使用两个key发送的消息都可以接收到

``` text
get message: User(id=2, name=jack) with directExchange
get message: User(id=3, name=larry) with directExchange
```

#### fanout模式

fanout模式下不需要绑定路由键，发给Exchange的消息会被转发到跟其绑定的所有队列。  
Receiver类增加两个RabbitListener方法，分别将`queueB、queueC`绑定到fanout交换器`exchange_fanout`  
Sender类不需要加方法，使用发送给exchange的方法，参数key指定为`空""`

``` java
// Receiver类
@RabbitListener(bindings = @QueueBinding(
        value = @Queue("queueB"),
        exchange = @Exchange(value = "exchange_fanout", type = ExchangeTypes.FANOUT)
))
public void fanOutExchange1(User user) {
    log.info("get message: {} with fanoutExchange", user);
}

@RabbitListener(bindings = @QueueBinding(
        value = @Queue("queueC"),
        exchange = @Exchange(value = "exchange_fanout", type = ExchangeTypes.FANOUT)
))
public void fanOutExchange2(User user) {
    log.info("get message: {} with fanoutExchange", user);
}

// 测试类，不需要指定key
@Test
void fanOutExchange() throws InterruptedException {
    sender.send("exchange_fanout", "", new User(4, "Mike"));
    Thread.sleep(1000);
}
```

测试结果，exchange_fanout上绑定的两个队列都收到了信息

``` text
get message: User(id=4, name=Mike) with fanoutExchange from queueB
get message: User(id=4, name=Mike) with fanoutExchange from queueC
```

#### topic模式

topic模式也需要queue和exchange绑定时设置路由key，不过更direct模式不一样的是，topic模式使用通配符来匹配路由key，*表示匹配一个单词，#表示匹配0个或多个单词  
在topic模式下消息会转发给所有匹配路由key的队列

Receiver类中增加两个方法，使用key`user.login.#`绑定队列`queueD`到交换器`exchange_topic`，使用key`user.register.#`绑定队列`queueE`到交换器`exchange_topic`

测试类中发送消息时需要指定exchange、路由key、消息

``` java
// Receiver类
@RabbitListener(bindings = @QueueBinding(
        value = @Queue("queueD"),
        exchange = @Exchange(value = "exchange_topic", type = ExchangeTypes.TOPIC),
        key = "user.login.#"
))
public void topicExchange(User user) {
    log.info("get message: {} with topicExchange from queueD", user);
}

@RabbitListener(bindings = @QueueBinding(
        value = @Queue("queueE"),
        exchange = @Exchange(value = "exchange_topic", type = ExchangeTypes.TOPIC),
        key = "user.register.#"
))
public void topicExchange2(User user) {
    log.info("get message: {} with topicExchange from queueE", user);
}

// 测试类
@Test
void topicExchange() throws InterruptedException {
    sender.send("exchange_topic", "user.login", new User(5, "bob"));
    Thread.sleep(1000);
}
```

测试结果，只有匹配的queueD收到了消息

``` java
get message: User(id=5, name=bob) with topicExchange from queueD
```
