# springboot之kafak

## 配置

创建springboot项目的时候勾选messaging选项中的kafka，springboot自动配置了很多，只需要很少的配置就可以运行了

`missing-topics-fatal`这一条要加上，不然kafka中没有提前创建topic时会报错，加上配置之后就可以自动创建topic了

``` yml
spring:
  kafka:
    # kafka服务器地址和端口，如果是kafka集群，配置其中一个即可
    # 为保证kafka服务器挂掉没影响，应该多配置几个地址
    bootstrap-servers: 127.0.0.1:9092
    consumer:
      # 消费者所属的群组
      group-id: test-group
      # 当partiton没有该group的offset信息时
      # earliest：从头开始消费
      # latsest ：从新生产的消息开始消费
      # 如果保存有offset信息，二者都从offset开始消费
      auto-offset-reset: earliest
      # 消息使用json反序列化
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      # 反序列化时，需要确认可信任的包，具体配置为消息类所在的包，*表示所有都可信任
      # 如果消息对象类型不在信任的包内，会报发序列化异常
      properties:
        spring:
          json:
            trusted:
              packages: "*"
    listener:
      # 未发现topic时不报错: 自动创建topic需要设置未false
      missing-topics-fatal: false
    producer:
      # Leader及ISR中的副本都写完消息，才能确认
      acks: all
      # 消息使用json序列化
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

## 消费者

``` java
@Slf4j
@Service
public class Receiver {
    @KafkaListener(topics = "testJson")
    public void listen(ConsumerRecord<?, ?> record) {
        log.info("message: {}", record.value());
    }
}
```

## 生产者

``` java
@Service
public class Sender {
    @Autowired
    private final KafkaTemplate<String, Message> kafkaTemplate;

    public Sender(KafkaTemplate<String, Message> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void send(Message message) {
        kafkaTemplate.send("testJson", message);
    }
}
```

## 测试

``` java
@SpringBootTest
class SenderTest {

  @Autowired
  private Sender sender;

  @Test
  void send() throws InterruptedException {
      sender.send(new Message(new Date(), "hello this is a json message"));
      Thread.sleep(1000);
  }
}
```

测试结果，消息者收到消息

``` text
com.example.demo.service.Receiver        : message: Message(date=Sat Apr 11 22:32:58 CST 2020, message=hello this is a json message)
```
