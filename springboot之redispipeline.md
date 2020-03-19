# springboot redis之使用pipeline

redis是基于客户端-服务器以及请求-响应协议的TCP服务，客户端向服务器发送请求是阻塞模式的，必须在等到服务器响应返回结果之后才能发送下一个请求，使用pipeline可以在服务端未响应时，继续向服务端发送请求，并最终一次性读取所有的服务端响应

* 非pipeline：client一个请求，redis server一个响应，期间client阻塞
* Pipeline：redis的管道命令，允许client将多个请求依次发给服务器，过程中而不需要等待请求的回复，在最后再一并读取结果即可。

## springboot中使用

springboot中redisTemplate有对应的api使用pipeline方式连接：  
```List<Object> executePipelined(RedisCallback<?> action)```  
```List<Object> executePipelined(SessionCallback<?> session)```  
在使用时只需重写RedisCallback的inRedis方法或者SessionCallback的execute方法，两者的区别是使用SessionCallback时支持开启事务(需要方法上加上@Transactional注解)

简单的使用管道测试：

``` java
public void usePipeline() {
    long start = System.currentTimeMillis();
    List<Object> list = redisTemplate.executePipelined(new SessionCallback<Object>() {

        @Override
        public Object execute(RedisOperations operations) throws DataAccessException {
            for (int i = 0; i < 50000; i++) {
                operations.opsForValue().get("city");
            }
            return null;
        }
    });
    long end = System.currentTimeMillis();
    log.info("开启管道用时: {}ms", (end - start));
}

public void noPipeline() {
    long start = System.currentTimeMillis();
    ArrayList<City> cities = new ArrayList<>();
    for (int i = 0; i < 50000; i++) {
        cities.add((City) redisTemplate.opsForValue().get("city"));
    }
    long end = System.currentTimeMillis();
    log.info("没有管道用时: {}ms", (end - start));
}
```

测试结果：

``` text
com.example.demo01.service.RedisService  : 开启管道用时: 3644ms
com.example.demo01.service.RedisService  : 没有管道用时: 10451ms
```
