# springboot之redis

redis的安装：

## 项目导包

``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

## application.yml 配置

如果redis配置了密码，需要加上password

``` yml
redis:
  host: 127.0.0.1
  port: 6379
```

## redis简单使用

直接使用springboot封装好的redisTemplate即可，这里就简单的将一个city对象写入redis，然后读取

``` java
@Slf4j
@Service
public class RedisService {
    @Resource
    private RedisTemplate<String, Object> redisTemplate;

    public void writeValue() {
        City city = new City();
        city.setId(11);
        city.setName("上海");
        redisTemplate.opsForValue().set("city", city);
    }

    public void readValue() {
        City city = (City) redisTemplate.opsForValue().get("city");
        log.info("city {}", city);
    }

    // 查看redisTemplate使用的序列化工具
    public void redisTemplateInfo() {
        log.info("keySerializer {}, valueSerializer {}", redisTemplate.getKeySerializer(), redisTemplate.getValueSerializer());
    }
}
```

读取结果

``` java
[main] com.example.demo01.service.RedisService  : city City(id=11, name=北京)
```

## redis序列化

redisTemplate默认使用的是jdk序列化，如果需要使用json序列化，可以自定义redisTemplate Bean

``` java
@Configuration
public class RedisConfig {
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(connectionFactory);

        // json序列化工具
        StringRedisSerializer keySerializer = new StringRedisSerializer();
        GenericJackson2JsonRedisSerializer valueSerializer = new GenericJackson2JsonRedisSerializer();

        template.setValueSerializer(valueSerializer);
        template.setHashValueSerializer(valueSerializer);
        template.setKeySerializer(keySerializer);
        template.setHashKeySerializer(keySerializer);

        return template;
    }
}
```

在redis中查看，这里的json包含了对象的类型信息用于反序列化

``` linux
redis-cli -c -p 7000 --raw
127.0.0.1:7000> get city
-> Redirected to slot [11479] located at 127.0.0.1:7002
{"@class":"com.example.demo01.entity.City","id":11,"name":"北京"}
```
