# resilience4j-ratelimiter 限流

## 简介

ratelimiter限制了服务被调用的次数，每隔一段时间重置该次数，服务在超出等待时间之后返回异常或者fallback方法

## 配置介绍

|配置属性 |默认值 |描述|
|:----|:----|:----|
|limitRefreshPeriod | 500ns |时间段，每隔该时间段，服务被调用的总次数重置|
|limitForPeriod | 50 |每个时间段内服务可以被调用的最大总次数|
|timeoutDuration | 5s |等待调用服务的时间，超出时间返回异常|

## 演示使用

1. 创建一个模拟的外部服务，服务执行完成需要2s的时间
2. 自定义RateLimiterConfig，每5s服务只能调用5次，调用等待时间为10s
3. 每隔50ms开启一个新线程调用外部服务

需要准备的依赖包，resilience4j的spring boot包，里面包括了resilience4j所有功能的包和自动配置功能

``` xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot2</artifactId>
    <version>1.7.0</version>
</dependency>
```

### 模拟服务

服务执行2s，打印时间和当前线程名称后返回

```java
@Service
public class ExternalLowRateService {
    public void callService() {
        try {
            Thread.sleep(2000);
            System.out.println(LocalTime.now() + " Call processing finished = " + Thread.currentThread().getName());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

### ratelimiter配置和使用

```java
@SpringBootTest
class ExternalLowRateServiceTest {

    @Autowired
    ExternalLowRateService service;

    @Test
    void callService() throws InterruptedException {
        /*
          自定义RateLimiterConfig, 限制服务调用次数为5，每隔5s重置，最大等待时间为10s
         */
        RateLimiterConfig config = RateLimiterConfig.custom()
            .limitRefreshPeriod(Duration.ofSeconds(5))
            .limitForPeriod(5)
            .timeoutDuration(Duration.ofSeconds(10))
            .build();
        RateLimiter rateLimiter = RateLimiter.of("externalLowRateService", config);
        for (int i = 1; i <= 20; i++) {
            System.out.println(LocalTime.now() + " Starting service call = " + i);
            // 使用rateLimiter调用外部服务
            new Thread(() -> rateLimiter.executeRunnable(service::callService),"service-call-" + i).start();
            Thread.sleep(50);
        }
        Thread.sleep(12000);
    }
}
```

### 输出结果

* 1-20次调用依次开始
* 1-5次开始执行并返回
* 5s后6-10次开始执行并返回
* 16-20次调用已等待超过10s，返回异常
* 11-15次调用执行并返回

```text
04:41:12.846114900 Starting service call = 1
04:41:12.914934300 Starting service call = 2
04:41:12.977763900 Starting service call = 3
04:41:13.041593400 Starting service call = 4
04:41:13.104454900 Starting service call = 5
04:41:13.166494600 Starting service call = 6
04:41:13.230321600 Starting service call = 7
04:41:13.293139600 Starting service call = 8
04:41:13.355637600 Starting service call = 9
04:41:13.417592800 Starting service call = 10
04:41:13.480228600 Starting service call = 11
04:41:13.542253700 Starting service call = 12
04:41:13.604270 Starting service call = 13
04:41:13.665588600 Starting service call = 14
04:41:13.728248400 Starting service call = 15
04:41:13.789320300 Starting service call = 16
04:41:13.850204 Starting service call = 17
04:41:13.912728400 Starting service call = 18
04:41:13.975277900 Starting service call = 19
04:41:14.038369600 Starting service call = 20
04:41:14.859399800 Call processing finished = service-call-1
04:41:14.921328200 Call processing finished = service-call-2
04:41:14.983447400 Call processing finished = service-call-3
04:41:15.045475300 Call processing finished = service-call-4
04:41:15.107320600 Call processing finished = service-call-5
04:41:19.864071500 Call processing finished = service-call-9
04:41:19.864071500 Call processing finished = service-call-6
04:41:19.864071500 Call processing finished = service-call-8
04:41:19.864071500 Call processing finished = service-call-10
04:41:19.864071500 Call processing finished = service-call-7
Exception in thread "service-call-16" io.github.resilience4j.ratelimiter.RequestNotPermitted: RateLimiter 'externalLowRateService' does not permit further calls
    at io.github.resilience4j.ratelimiter.RequestNotPermitted.createRequestNotPermitted(RequestNotPermitted.java:43)
    at io.github.resilience4j.ratelimiter.RateLimiter.waitForPermission(RateLimiter.java:591)
    at io.github.resilience4j.ratelimiter.RateLimiter.lambda$decorateCheckedRunnable$3(RateLimiter.java:246)
    at io.vavr.CheckedRunnable.lambda$unchecked$0(CheckedRunnable.java:68)
    at io.github.resilience4j.ratelimiter.RateLimiter.executeRunnable(RateLimiter.java:872)
    at io.github.resilience4j.ratelimiter.RateLimiter.executeRunnable(RateLimiter.java:862)
    at io.ysh.eurekacientconsumer.service.ExternalLowRateServiceTest.lambda$callService$0(ExternalLowRateServiceTest.java:29)
    at java.base/java.lang.Thread.run(Thread.java:834)
Exception in thread "service-call-17" io.github.resilience4j.ratelimiter.RequestNotPermitted: RateLimiter 'externalLowRateService' does not permit further calls
    at io.github.resilience4j.ratelimiter.RequestNotPermitted.createRequestNotPermitted(RequestNotPermitted.java:43)
    at io.github.resilience4j.ratelimiter.RateLimiter.waitForPermission(RateLimiter.java:591)
    at io.github.resilience4j.ratelimiter.RateLimiter.lambda$decorateCheckedRunnable$3(RateLimiter.java:246)
    at io.vavr.CheckedRunnable.lambda$unchecked$0(CheckedRunnable.java:68)
    at io.github.resilience4j.ratelimiter.RateLimiter.executeRunnable(RateLimiter.java:872)
    at io.github.resilience4j.ratelimiter.RateLimiter.executeRunnable(RateLimiter.java:862)
    at io.ysh.eurekacientconsumer.service.ExternalLowRateServiceTest.lambda$callService$0(ExternalLowRateServiceTest.java:29)
    at java.base/java.lang.Thread.run(Thread.java:834)
Exception in thread "service-call-18" io.github.resilience4j.ratelimiter.RequestNotPermitted: RateLimiter 'externalLowRateService' does not permit further calls
    at io.github.resilience4j.ratelimiter.RequestNotPermitted.createRequestNotPermitted(RequestNotPermitted.java:43)
    at io.github.resilience4j.ratelimiter.RateLimiter.waitForPermission(RateLimiter.java:591)
    at io.github.resilience4j.ratelimiter.RateLimiter.lambda$decorateCheckedRunnable$3(RateLimiter.java:246)
    at io.vavr.CheckedRunnable.lambda$unchecked$0(CheckedRunnable.java:68)
    at io.github.resilience4j.ratelimiter.RateLimiter.executeRunnable(RateLimiter.java:872)
    at io.github.resilience4j.ratelimiter.RateLimiter.executeRunnable(RateLimiter.java:862)
    at io.ysh.eurekacientconsumer.service.ExternalLowRateServiceTest.lambda$callService$0(ExternalLowRateServiceTest.java:29)
    at java.base/java.lang.Thread.run(Thread.java:834)
Exception in thread "service-call-19" io.github.resilience4j.ratelimiter.RequestNotPermitted: RateLimiter 'externalLowRateService' does not permit further calls
    at io.github.resilience4j.ratelimiter.RequestNotPermitted.createRequestNotPermitted(RequestNotPermitted.java:43)
    at io.github.resilience4j.ratelimiter.RateLimiter.waitForPermission(RateLimiter.java:591)
    at io.github.resilience4j.ratelimiter.RateLimiter.lambda$decorateCheckedRunnable$3(RateLimiter.java:246)
    at io.vavr.CheckedRunnable.lambda$unchecked$0(CheckedRunnable.java:68)
    at io.github.resilience4j.ratelimiter.RateLimiter.executeRunnable(RateLimiter.java:872)
    at io.github.resilience4j.ratelimiter.RateLimiter.executeRunnable(RateLimiter.java:862)
    at io.ysh.eurekacientconsumer.service.ExternalLowRateServiceTest.lambda$callService$0(ExternalLowRateServiceTest.java:29)
    at java.base/java.lang.Thread.run(Thread.java:834)
Exception in thread "service-call-20" io.github.resilience4j.ratelimiter.RequestNotPermitted: RateLimiter 'externalLowRateService' does not permit further calls
    at io.github.resilience4j.ratelimiter.RequestNotPermitted.createRequestNotPermitted(RequestNotPermitted.java:43)
    at io.github.resilience4j.ratelimiter.RateLimiter.waitForPermission(RateLimiter.java:591)
    at io.github.resilience4j.ratelimiter.RateLimiter.lambda$decorateCheckedRunnable$3(RateLimiter.java:246)
    at io.vavr.CheckedRunnable.lambda$unchecked$0(CheckedRunnable.java:68)
    at io.github.resilience4j.ratelimiter.RateLimiter.executeRunnable(RateLimiter.java:872)
    at io.github.resilience4j.ratelimiter.RateLimiter.executeRunnable(RateLimiter.java:862)
    at io.ysh.eurekacientconsumer.service.ExternalLowRateServiceTest.lambda$callService$0(ExternalLowRateServiceTest.java:29)
    at java.base/java.lang.Thread.run(Thread.java:834)
04:41:24.863851800 Call processing finished = service-call-12
04:41:24.863851800 Call processing finished = service-call-15
04:41:24.863851800 Call processing finished = service-call-11
04:41:24.863851800 Call processing finished = service-call-14
04:41:24.863851800 Call processing finished = service-call-13

```

## springboot自动配置

配置在application.yml中

* configs：ratelimiter配置，可以配置多个
* instances：ratelimiter实例名称，可以配置baseConfig指定具体的ratelimiter配置，也可以自行配置

``` yml
resilience4j.ratelimiter:
  configs:
    default:
      limitForPeriod: 10
      limitRefreshPeriod: 1s
  instances:
    externalLowRateService:
      limitForPeriod: 5
      limitRefreshPeriod: 5s
      timeoutDuration: 10s
```

注解`@Bulkhead`有两个属性：

* name：ratelimiter的名称，对应yml文件中instances中的配置
* fallbackMethod：方法名，超出等待时间的服务调用会执行该方法并返回

``` java
@Service
public class ExternalLowRateService {
   @RateLimiter(name = "externalLowRateService")
    public void callService() {
        try {
            Thread.sleep(2000);
            System.out.println(LocalTime.now() + " Call processing finished = " + Thread.currentThread().getName());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

测试方法，没有显式的bulkhead配置和创建，直接执行服务调用即可，结果返回跟上面的测试一样

```java
@SpringBootTest
class ExternalLowRateServiceTest {

    @Autowired
    ExternalLowRateService service;

    @Test
    void callService() throws InterruptedException {
        for (int i = 1; i <= 20; i++) {
            System.out.println(LocalTime.now() + " Starting service call = " + i);
            new Thread(service::callService,"service-call-" + i).start();
            Thread.sleep(50);
        }
        Thread.sleep(12000);
    }
}
```
