# OpenFeign

OpenFeign是一个声明式的网络服务客户端，使用feign只需要创建一个接口并加上注解，spring cloud添加了feign对spring mvc注解的支持

## 引入OPenFeign依赖

* 添加依赖包

继续使用之前的项目工程，OpenFeign是消费服务的客户端，这里只需要在consumer工程下添加依赖

``` xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

* 注解开启功能

``` java
@EnableFeignClients
public class EurekaCientConsumerApplication {
```

## 创建Feign client

Feign client是一个接口，在注解中的内容是远程服务的Eureka服务名称  
`@GetMapping`是和spring mvc一样的注解，注解中的内容是eureka-client-producer服务的port接口

``` java
@FeignClient("eureka-client-producer")
public interface ProducerClient {
    @GetMapping("port")
    int getPort();
}
```

## 使用Feign client

Feign client的使用只需引入ProducerClient接口类型的Bean，调用接口的方法，feign便会发起相应的http请求

``` java
@Autowired
private ProducerClient portClient;

@GetMapping("feign")
public Integer consumePort() {
    return portClient.getPort();
}
```

类似与mybatis的mapper的用法

启动之前创建的eureka server工程，两个producer工程和consumer工程，访问consumer工程的feign接口，结果显示服务调用成功，连续刷新浏览器界面，可以看出openfeign是支持loadbalancerd的负载平衡功能的
