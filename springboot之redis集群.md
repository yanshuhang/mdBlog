# springboot连接redis集群

首先要搭建好redis的集群，搭建的教程：<https://www.jianshu.com/p/7fe101dba5d0>

## application.yml 配置

``` yml
redis:
  cluster:
    nodes:
      - 127.0.0.1:7000
      - 127.0.0.1:7001
      - 127.0.0.1:7002
      - 127.0.0.1:7003
      - 127.0.0.1:7004
      - 127.0.0.1:7005
```

其余操作跟单机版一样：  

``` java
@Slf4j
@Service
public class RedisService {
    @Resource
    private RedisTemplate<String, Object> redisTemplate;

    public void writeValue() {
        City city = new City();
        city.setId(11);
        city.setName("北京");
        redisTemplate.opsForValue().set("city", city);
    }

    public void readValue() {
        City city = (City) redisTemplate.opsForValue().get("city");
        log.info("city {}", city);
    }

    public void redisTemplateInfo() {
        log.info("keySerializer {}, valueSerializer {}", redisTemplate.getKeySerializer(), redisTemplate.getValueSerializer());
    }
}
```

测试读写都没有问题，在客户端里查看redis，可以看到city存储在7002节点里

``` linux
redis-cli -c -p 7000 --raw
127.0.0.1:7000> get city
-> Redirected to slot [11479] located at 127.0.0.1:7002
{"@class":"com.example.demo01.entity.City","id":11,"name":"北京"}
```
