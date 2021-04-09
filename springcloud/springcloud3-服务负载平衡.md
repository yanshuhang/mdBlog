# spring cloud loadBalancer

## 集成 spring cloud loadBalancer

当前版本的Eureka client自带spring cloud loadBalancer依赖

eureka简单使用：<https://www.jianshu.com/p/0871254322b4>

## 开启loadBalancer功能

在RestTemplate和WebClient bean上加上`@loadBalanced`注解就开启了客户端负载均衡功能

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

## 测试负载均衡

### 创建测试接口

在`producer`项目中创建接口返回项目的端口`server.port`

``` java
@Value("${server.port}")
private int serverPort;

@GetMapping("port")
public int getPort() {
    return serverPort;
}
```

### 开启多个producer微服务

1. 可以在启动之后，修改端口配置，再次启动
2. 将项目打包，使用java命令启动，在命令中指定端口：`java - Dserver.port=8083 -jar demo.jar`
3. 在IDE中创建配置，例如idea中可以负责原本的启动配置然后修改下VM options：`-Dserver.port=8083`

不管哪种方式，启动后就有了两个producer微服务，端口分别是8081和8083

### consumer

consumer项目中的代码不需要修改，使用RestTemplate和WebClient都会进行负载平衡

``` java
 @GetMapping("consume")
public Integer consumerHello() {
    return restTemplate.getForObject("http://eureka-client-producer/port", Integer.class);
}

@GetMapping("webclient")
public Mono<Integer> consumeWithWebClient() {
    WebClient client = builder.build();
    return client.get()
        .uri("http://eureka-client-producer/port")
        .retrieve()
        .bodyToMono(Integer.class);
    }
```

使用浏览器访问<http://localhost:8082/consume>，连续刷新会看到8081 和 8083交替出现
