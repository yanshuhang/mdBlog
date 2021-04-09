# eureka 微服务注册与发现

在 [springcloud1-微服务调用](https://www.jianshu.com/p/9421c7a88a43)，中RestTemplate和WebClient使用的是硬编码的地址，写法繁琐，如果服务的地址发生变化，代码也需要跟着改动

使用`Eureka`，硬编码的地址可以用服务名称代替，这样只要服务的名称不变，服务的地址更换都不会影响到现有代码

由两个组件组成：Eureka服务器和Eureka客户端，Eureka服务器用作服务注册中心，我们的微服务就是Eureka客户端，注册中心负责微服务的注册和发现，微服务在启动时向注册中心注册自己的信息、获取注册中心其他微服务的信息  

## 配置Eureka服务器

Eureka服务器跟微服务一样，也是一个spring boot项目  
在<https://start.spring.io/>创建项目 `Eureka-server-demo`，只需引入`Eureka Server`依赖

### 配置

``` yml
server:
  port: 8761
spring:
  application:
    name: eureka-server-demo
eureka:
  instance:
    hostname: localhost
  server:
    # 关闭保护模式
    enable-self-preservation: false
  client:
    # 单个eureka服务器不需要向自己注册和获取信息
    register-with-eureka: false
    fetch-registry: false
```

* 保护模式：Eureka服务器每隔一定时间收到客户端的心跳，如果客户端掉线，服务器收不到心跳，就将其删除。在少数客户端不可用时，Eureka认为是客户端的问题，然后将其删除；在多数客户端不可用时，Eureka认为是网络问题，然后进入保护模式不再删除客户端  
这里只是简单的学习，微服务较少，关闭保护模式，不然容易进入保护模式

注解开启 Eureka server 功能，在启动类上使用即可

``` java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerDemoApplication {
```

## 配置Eureka客户端

这里继续使用[springcloud1-微服务调用](https://www.jianshu.com/p/9421c7a88a43)的 consumer 和 producer 项目，添加 Eureka client 的依赖

``` xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

注解 `@EnableEurekaClient` 开启 Eureka client 功能，使用在启动类上即可

配置文件：

``` yml
server:
  port: 8081
spring:
  application:
    name: eureka-client-producer
eureka:
  instance:
    hostname: localhost
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

* defaultZone 中配置eureka服务器的地址
* consumer 项目的配置改下端口8082 和 项目名称spring.application.name: eureka-client-consumer 其他的都一样

producker项目的接口不用变化，consumer中的硬编码地址改为服务的名称

``` java
@GetMapping("consume")
public String consumerHello() {
    return restTemplate.getForObject("http://eureka-client-producer/hello", String.class);
}
```

访问显示没有问题

### 报错java.net.UnknownHostException的解决

应该是springboot较新的版本的问题，跟以前的教程有些不一样，RestTemplate 和 WebClient bean需要加上 `@LoadBalanced`注解

``` java
@Bean
@LoadBalanced
public RestTemplate restTemplate(RestTemplateBuilder restTemplateBuilder) {
    return restTemplateBuilder.build();
}

@Bean
@LoadBalanced
public WebClient.Builder loadBalancedWebClientBuilder() {
    return WebClient.builder();
}
```
