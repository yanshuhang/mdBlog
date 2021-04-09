# 微服务调用

在微服务架构中，完成一项功能可能需要多个分离的微服务的功能整合起来，服务之间需要相互通信配合工作，http调用就是常见的通信方式。  
spirng中提供了多种http调用的工具：阻塞式的 `RestTemplate` 和异步非阻塞的 `WebClient`，两者都提供了对应http标准的方法：get、post、put、delete、patch等，可以用来发起相应的请求

## 创建微服务

在<https://start.spring.io/>创建两个项目 `consumer` 和 `producer`，引入 `Spring Web` 和 `Spring Reactive Web` 依赖，这两个包分别提供 `RestTemplate` 和 `WebClient` 功能

* 在 `producer` 项目中定义端口8081，创建服务接口：就是一个简单的Controller类

``` Java
@GetMapping("hello")
public String produce() {
    return "pruduce something...";
}
```

## 使用RestTemplate访问producer

``` Java
@Autowired
private RestTemplate restTemplate;

@GetMapping("consume")
public String consumeHello() {
    return restTemplate.getForObject("http://localhost:8081/hello", String.class);
}
```

getForObject方法调用远程服务接口并返回一个对象，第一个参数是需要访问的远程服务的地址，就是 `producer` 中 hello 接口的地址，第二个参数是返回的对象的类型，就是远程服务返回的类型

需要注意的问题，在较新版本中 `RestTemplate` 已经不自动装配了，需要手动创建bean，如下：

``` Java
# 创建 RestTemplate bean
@Bean
public RestTemplate restTemplate(RestTemplateBuilder restTemplateBuilder) {
    return restTemplateBuilder.build();
}
```

定义 `consumer` 项目接口8082，在浏览器中访问 <http://localhost:8082/consume> 可以看到成功返回了 `producer` 服务结果

## 使用WebClient的使用

``` Java
@Autowired
private WebClient.Builder builder;

@GetMapping("webclient")
public Mono<String> consumeWithWebClient() {
    WebClient client = builder.build();
    Mono<String> body = client.get()
        .uri("http://localhost:8081/hello")
        .retrieve()
        .bodyToMono(String.class);
    return body;
}
```

webclient的get() 对应http的get请求，uri()参数是请求的服务的地址，在retrieve()之后将response的body转为Mono对象

注意这里的controller方法返回值需要使用Mono\<String>

在浏览器中访问 `http://localhost:8082/webclient` 也可以看到成功返回了 `producer` 服务结果

也可以使用Mono对象的block方法，这样可以返回具体的对象，但是block是阻塞的不建议使用

``` Java
@GetMapping("webclient")
public String consumeWithWebClient() {
    WebClient client = builder.build();
    Mono<String> body = client.get()
        .uri("http://localhost:8081/hello")
        .retrieve()
        .bodyToMono(String.class);
    String s = body.block();
    return s;
}
```

## RestTemplate对比WebClient

* `RestTemplate` 是一个执行http请求的同步阻塞的客户端，`RestTemplate` 使用了基于每个请求对应一个线程模型（thread-per-request）的 Java Servlet API。这意味着，直到 Web 客户端收到响应之前，线程都将一直被阻塞下去。为处理更多的请求，必然会带来更多的线程开销，并且cpu在线程切换中也有一定的性能损耗

* `WebClient` 是非阻塞式客户端，WebClient 使用 Spring Reactive Framework 所提供的异步非阻塞解决方案。在等待响应返回时不会阻塞正在执行的线程。只有当响应就绪时，才会产生通知。

在spring5.0之后`RestTemplate`已经处于维护状态，spring官方建议使用`WebClient`

`WebClient`官方文档<https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html#webflux-client>
