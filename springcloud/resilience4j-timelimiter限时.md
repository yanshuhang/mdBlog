# resilience4j-timelimiter 限时

## 简介

限制服务调用的时间，超时则返回异常或执行fallback方法，只能用于Reactor和RxJava，springcloud中支持webclient调用外部服务，不支持restTemplate

## 使用

配置文件：

```yml
resilience4j.timelimiter:
  configs:
    default:
      cancelRunningFuture: true
      timeoutDuration: 3s
  instances:
    testClient:
      baseConfig: default
```

外部服务，服务执行4s然后返回，服务使用8080端口，uri为`slow-service`

```java
@Service
public class ExternalSlowService {

    public void callService() {
        try {
            Thread.sleep(4000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

```java
@Autowired
ExternalSlowService slowService;

@GetMapping("slow-service")
public String slowService() {
    return slowService.callService();
} 
```

消费外部服务，调用slow-service，如果超时则执行fallback方法

```java
@Service
public class ConsumeSlowService {
    @Autowired
    WebClient.Builder builder;

    @TimeLimiter(name = "externalSlowService", fallbackMethod = "fallback")
    public Mono<String> call() {
        WebClient client = builder.build();
        Mono<String> response = client.get()
            .uri("http://localhost:8080/slow-service")
            .retrieve()
            .bodyToMono(String.class);
        return response;
    }

    public Mono<String> fallback(Throwable throwable) {
        return Mono.just("fallback");
    }
}
```

消费服务的controller，使用8081端口

```java
@Autowired
ConsumeSlowService consumeSlowService;
@Autowired
private final WebClient.Builder builder;

@GetMapping("consume-slow-service")
public Mono<String> consume() {
    return consumeSlowService.call();
}
```

访问<http://localhost/8081/consume-slow-service>， 返回字符串`fallback`

修改配置的timelimiter超时时间为5s，重启8081服务，再次访问上面的地址，返会字符串`slow-service`
