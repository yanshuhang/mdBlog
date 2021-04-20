# resilience4j bulkhead

## 简介

bulkhead限制了同时执行中的服务调用数量，避免一个服务的大量请求占用完机器的全部资源

## 配置介绍

|配置属性 |默认值 |描述|
|:----|:----|:----|
|maxConcurrentCalls |25 |服务同时执行的最大数量
|maxWaitDuration |0 |超过最大数量时，其他未执行的请求的最大等待时间

## 演示使用

1. 创建一个模拟的外部服务，服务执行完成需要2s的时间
2. 自定义BulkheadConfig，修改最大同时执行的数量为5，等待时间为5s
3. 每隔10ms开启一个新线程调用外部服务

需要准备的依赖包，resilience4j的spring boot包，里面包括了resilience4j所有功能的包和自动配置功能

``` xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot2</artifactId>
    <version>1.7.0</version>
</dependency>
```

### 模拟服务

服务执行2s，然后打印时间和当前线程名称

``` java
@Service
public class ExternalConcurrentService {

    public void callService() {
        try {
            Thread.sleep(2000);
            System.out.println(LocalTime.now() + " Call processing finished = " + Thread.currentThread().getName());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### bulkhead配置和使用

``` java
@SpringBootTest
class ExternalConcurrentServiceTest {

    @Autowired
    ExternalConcurrentService service;

    @Test
    void callService() throws InterruptedException {
        /**
         * 创建bulkheadconfig，最大同时访问个数5，最大等待时间5s
         */
        BulkheadConfig config = BulkheadConfig.custom()
            .maxConcurrentCalls(5)
            .maxWaitDuration(Duration.ofSeconds(5))
            .build();
        Bulkhead bulkhead = Bulkhead.of("externalConcurrentService", config);

        for (int i = 1; i <= 20; i++) {
            System.out.println(LocalTime.now() + " Starting service call = " + i);
            // 使用bulkhead调用外部服务
            new Thread(() -> bulkhead.executeRunnable(service::callService), "service-call-" + (i)).start();
            Thread.sleep(10);
        }

        Thread.sleep(10000);
    }
}
```

### 输出结果

* 20次调用依次开始
* 1-5次调用同时执行然后返回，其他调用都在等待
* 然后6-10次也依次执行完成
* 16-20次调用返回了异常，这些调用等待执行的时间超过了配置的5s
* 最后11-15次调用结束

``` text
04:11:02.036855 Starting service call = 1
04:11:02.061813200 Starting service call = 2
04:11:02.077744200 Starting service call = 3
04:11:02.092730500 Starting service call = 4
04:11:02.108695500 Starting service call = 5
04:11:02.123658400 Starting service call = 6
04:11:02.139610100 Starting service call = 7
04:11:02.154876 Starting service call = 8
04:11:02.170141700 Starting service call = 9
04:11:02.185673400 Starting service call = 10
04:11:02.201139600 Starting service call = 11
04:11:02.216462900 Starting service call = 12
04:11:02.231764700 Starting service call = 13
04:11:02.246983100 Starting service call = 14
04:11:02.262306100 Starting service call = 15
04:11:02.277676500 Starting service call = 16
04:11:02.293432400 Starting service call = 17
04:11:02.308953 Starting service call = 18
04:11:02.324277300 Starting service call = 19
04:11:02.339946600 Starting service call = 20
04:11:04.045773500 Call processing finished = service-call-1
04:11:04.076742 Call processing finished = service-call-2
04:11:04.091805400 Call processing finished = service-call-3
04:11:04.107228700 Call processing finished = service-call-4
04:11:04.122704300 Call processing finished = service-call-5
04:11:06.059598600 Call processing finished = service-call-6
04:11:06.090279400 Call processing finished = service-call-7
04:11:06.106207100 Call processing finished = service-call-8
04:11:06.121352900 Call processing finished = service-call-9
04:11:06.136645500 Call processing finished = service-call-10
Exception in thread "service-call-16" io.github.resilience4j.bulkhead.BulkheadFullException: Bulkhead 'externalConcurrentService' is full and does not permit further calls
    at io.github.resilience4j.bulkhead.BulkheadFullException.createBulkheadFullException(BulkheadFullException.java:49)`
    at io.github.resilience4j.bulkhead.internal.SemaphoreBulkhead.acquirePermission(SemaphoreBulkhead.java:164)
    at io.github.resilience4j.bulkhead.Bulkhead.lambda$decorateRunnable$10(Bulkhead.java:297)
    at io.github.resilience4j.bulkhead.Bulkhead.executeRunnable(Bulkhead.java:534)
    at io.ysh.eurekacientconsumer.service.ExternalConcurrentServiceTest.lambda$callService$0(ExternalConcurrentServiceTest.java:31)
    at java.base/java.lang.Thread.run(Thread.java:834)
Exception in thread "service-call-17" io.github.resilience4j.bulkhead.BulkheadFullException: Bulkhead 'externalConcurrentService' is full and does not permit further calls
    at io.github.resilience4j.bulkhead.BulkheadFullException.createBulkheadFullException(BulkheadFullException.java:49)
    at io.github.resilience4j.bulkhead.internal.SemaphoreBulkhead.acquirePermission(SemaphoreBulkhead.java:164)
    at io.github.resilience4j.bulkhead.Bulkhead.lambda$decorateRunnable$10(Bulkhead.java:297)
    at io.github.resilience4j.bulkhead.Bulkhead.executeRunnable(Bulkhead.java:534)
    at io.ysh.eurekacientconsumer.service.ExternalConcurrentServiceTest.lambda$callService$0(ExternalConcurrentServiceTest.java:31)
    at java.base/java.lang.Thread.run(Thread.java:834)
Exception in thread "service-call-18" io.github.resilience4j.bulkhead.BulkheadFullException: Bulkhead 'externalConcurrentService' is full and does not permit further calls
    at io.github.resilience4j.bulkhead.BulkheadFullException.createBulkheadFullException(BulkheadFullException.java:49)
    at io.github.resilience4j.bulkhead.internal.SemaphoreBulkhead.acquirePermission(SemaphoreBulkhead.java:164)
    at io.github.resilience4j.bulkhead.Bulkhead.lambda$decorateRunnable$10(Bulkhead.java:297)
    at io.github.resilience4j.bulkhead.Bulkhead.executeRunnable(Bulkhead.java:534)
    at io.ysh.eurekacientconsumer.service.ExternalConcurrentServiceTest.lambda$callService$0(ExternalConcurrentServiceTest.java:31)
    at java.base/java.lang.Thread.run(Thread.java:834)
Exception in thread "service-call-19" io.github.resilience4j.bulkhead.BulkheadFullException: Bulkhead 'externalConcurrentService' is full and does not permit further calls
    at io.github.resilience4j.bulkhead.BulkheadFullException.createBulkheadFullException(BulkheadFullException.java:49)
    at io.github.resilience4j.bulkhead.internal.SemaphoreBulkhead.acquirePermission(SemaphoreBulkhead.java:164)
    at io.github.resilience4j.bulkhead.Bulkhead.lambda$decorateRunnable$10(Bulkhead.java:297)
    at io.github.resilience4j.bulkhead.Bulkhead.executeRunnable(Bulkhead.java:534)
    at io.ysh.eurekacientconsumer.service.ExternalConcurrentServiceTest.lambda$callService$0(ExternalConcurrentServiceTest.java:31)
    at java.base/java.lang.Thread.run(Thread.java:834)
Exception in thread "service-call-20" io.github.resilience4j.bulkhead.BulkheadFullException: Bulkhead 'externalConcurrentService' is full and does not permit further calls
    at io.github.resilience4j.bulkhead.BulkheadFullException.createBulkheadFullException(BulkheadFullException.java:49)
    at io.github.resilience4j.bulkhead.internal.SemaphoreBulkhead.acquirePermission(SemaphoreBulkhead.java:164)
    at io.github.resilience4j.bulkhead.Bulkhead.lambda$decorateRunnable$10(Bulkhead.java:297)
    at io.github.resilience4j.bulkhead.Bulkhead.executeRunnable(Bulkhead.java:534)
    at io.ysh.eurekacientconsumer.service.ExternalConcurrentServiceTest.lambda$callService$0(ExternalConcurrentServiceTest.java:31)
    at java.base/java.lang.Thread.run(Thread.java:834)
04:11:08.071709400 Call processing finished = service-call-11
04:11:08.102768900 Call processing finished = service-call-12
04:11:08.118110700 Call processing finished = service-call-13
04:11:08.133213700 Call processing finished = service-call-14
04:11:08.148714400 Call processing finished = service-call-15
```

## springboot自动配置

配置在application.yml中

* configs：bulkhead配置，可以配置多个
* instances：bulkhead实例名称，可以配置baseConfig指定具体的bulkhead配置，也可以自行配置

```yml
resilience4j.bulkhead:
  configs:
    default:
      maxConcurrentCalls: 10
  instances:
    externalConcurrentService:
      baseConfig: default
      maxConcurrentCalls: 5
      maxWaitDuration: 5s
```

注解`@Bulkhead`有三个属性：

* name：bulkhead的名称，对应yml文件中instances中的配置
* type：bulkhead类型，默认为基于信号量SEMAPHORE的实现，可以指定为基于线程池THREADPOOL的实现
* fallbackMethod：方法名，超出等待时间的服务调用会执行该方法并返回

``` java
@Service
public class ExternalConcurrentService {

    @Bulkhead(name = "externalConcurrentService")
    public void callService() {
        try {
            Thread.sleep(2000);
            System.out.println(LocalTime.now() + " Call processing finished = " + Thread.currentThread().getName());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

测试方法，没有显式的bulkhead配置和创建，直接执行服务调用即可，结果返回跟上面的测试一样

``` java
@SpringBootTest
class ExternalConcurrentServiceTest {

    @Autowired
    ExternalConcurrentService service;

    @Test
    void callService() throws InterruptedException {
        for (int i = 1; i <= 20; i++) {
            System.out.println(LocalTime.now() + " Starting service call = " + i);
            new Thread(service::callService, "service-call-" + (i)).start();
            Thread.sleep(10);
        }
        Thread.sleep(10000);
    }
}
```
