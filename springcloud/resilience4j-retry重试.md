# resilience4j-retry 重试

## 简介

resilience4j-retry在服务调用返回失败时提供了额外尝试调用的功能

## 配置介绍

|配置属性 |默认值 |描述|
|:----|:----|:----|
|maxAttempts |3 |最大的尝试服务调用次数，包括第一次的调用|
|waitDuration |500ms |两次重试的时间间隔|
|intervalFunction |numOfAttempts -> waitDuration |自定义的IntervalFunction，可以根据当前尝试的次数动态的修改重试的时间间隔|
|intervalBiFunction |(numOfAttempts, Either<throwable, result) -> waitDuration| 自定义的IntervalBiFunction，根据当前尝试的次数和返回的结果或异常动态的修改重试的时间间隔|
|retryOnResultPredicate |result -> false |自定义的Predicate，根据服务返回的结果判断是否应该重试。如果需要重试Predicate应返回true，否则返回false|
|retryOnExceptionPredicate |throwable -> true |自定义的Predicate，根据服务返回的异常判断是否应该重试。如果需要重试Predicate应返回true，否则返回false|
|retryExceptions |empty |异常列表，遇到列表中的异常或其子类则重试|
|ignoreExceptions |empty |异常列表，遇到列表中的异常或其子类则不重试|

## 演示使用

1. 创建一个模拟的外部服务，60%的几率返回成功，20%的几率返回失败，20%的几率返回自定义异常
2. 自定义RetryConfig，
    * 最多尝试2次(即重试1次)，重试间隔1s
    * 遇到返回失败和自定义异常时重试
3. 依次进行20次服务调用

需要准备的依赖包，resilience4j的spring boot包，里面包括了resilience4j所有功能的包和自动配置功能

``` xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot2</artifactId>
    <version>1.7.0</version>
</dependency>
```

### 模拟服务

60%的几率返回成功："SUCCESS"，20%的几率返回失败："FAILURE"，20%的几率返回自定义异常BadProcessingException

```java
@Service
public class ExternalRandomFailService {

    public String callService() {
        double random = Math.random();
        if (random < 0.6) {
            System.out.println("\t" + LocalTime.now().truncatedTo(ChronoUnit.MILLIS) + " : Processing finished. Status = SUCCESS");
            return "SUCCESS";
        } else if (random < 0.8) {
            System.out.println("\t" + LocalTime.now().truncatedTo(ChronoUnit.MILLIS) + " : Processing finished. Status = FAILURE");
            return "FAILURE";
        } else {
            System.out.println("\t" + LocalTime.now().truncatedTo(ChronoUnit.MILLIS) + " : Processing finished. Status = BadProcessingException");
            throw new BadProcessingException("Bad processing");
        }
    }
}
```

### retry配置和使用

```java
@SpringBootTest
class ExternalRandomFailServiceTest {

    @Autowired
    ExternalRandomFailService service;

    @Test
    void callService() {
        /*
         * RetryConfig: 总共尝试调用服务2次，间隔1s，返回结果是FAILURE或异常时进行重试
         */
        RetryConfig config = RetryConfig.custom()
            .maxAttempts(2)
            .waitDuration(Duration.ofSeconds(1))
            .retryOnResult(response -> response.equals("FAILURE"))
            .retryOnException(e -> e instanceof BadProcessingException)
            .build();
        Retry retry = Retry.of("externalRandomFailService", config);
        for (int i = 1; i <= 20; i++) {
            System.out.println(">> Call count = " + i);
            try {
                String result = retry.executeSupplier(service::callService);
                System.out.println("\tFinal Result = " + result);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```

### 输出结果

* 第一次调用就返回SUCCESS，例如count = 5
* 第一次返回FAILURE或异常，重试返回SUCCESS，例如count=2和count=13
* 第一次返回FAILURE或异常，重试仍返回FAILURE或异常，例如count=1和count=4
* 最大尝试次数为2次，所有重试最多只有1次，重试的间隔为1s

```text
>> Call count = 1
    20:04:14.445 : Processing finished. Status = BadProcessingException
    20:04:15.465 : Processing finished. Status = BadProcessingException
    io.ysh.eurekacientconsumer.service.BadProcessingException: Bad processing
    at io.ysh.eurekacientconsumer.service.ExternalRandomFailService.callService(ExternalRandomFailService.java:21)
    at io.github.resilience4j.retry.Retry.lambda$decorateSupplier$2(Retry.java:213)
    at io.github.resilience4j.retry.Retry.executeSupplier(Retry.java:430)
    at io.ysh.eurekacientconsumer.service.ExternalRandomFailServiceTest.callService(ExternalRandomFailServiceTest.java:33)
>> Call count = 2
    20:04:15.473 : Processing finished. Status = BadProcessingException
    20:04:16.484 : Processing finished. Status = SUCCESS
    Final Result = SUCCESS
>> Call count = 3
    20:04:16.492 : Processing finished. Status = BadProcessingException
    20:04:17.498 : Processing finished. Status = FAILURE
    Final Result = FAILURE
>> Call count = 4
    20:04:17.504 : Processing finished. Status = FAILURE
    20:04:18.519 : Processing finished. Status = FAILURE
    Final Result = FAILURE
>> Call count = 5
    20:04:18.519 : Processing finished. Status = SUCCESS
    Final Result = SUCCESS
>> Call count = 6
    20:04:18.520 : Processing finished. Status = SUCCESS
    Final Result = SUCCESS
>> Call count = 7
    20:04:18.520 : Processing finished. Status = SUCCESS
    Final Result = SUCCESS
>> Call count = 8
    20:04:18.520 : Processing finished. Status = SUCCESS
    Final Result = SUCCESS
>> Call count = 9
    20:04:18.520 : Processing finished. Status = SUCCESS
    Final Result = SUCCESS
>> Call count = 10
    20:04:18.520 : Processing finished. Status = SUCCESS
    Final Result = SUCCESS
>> Call count = 11
    20:04:18.521 : Processing finished. Status = FAILURE
    20:04:19.527 : Processing finished. Status = FAILURE
    Final Result = FAILURE
>> Call count = 12
    20:04:19.527 : Processing finished. Status = BadProcessingException
    20:04:20.537 : Processing finished. Status = SUCCESS
    Final Result = SUCCESS
>> Call count = 13
    20:04:20.537 : Processing finished. Status = FAILURE
    20:04:21.542 : Processing finished. Status = SUCCESS
    Final Result = SUCCESS
>> Call count = 14
    20:04:21.542 : Processing finished. Status = SUCCESS
    Final Result = SUCCESS
>> Call count = 15
    20:04:21.543 : Processing finished. Status = BadProcessingException
    20:04:22.550 : Processing finished. Status = SUCCESS
    Final Result = SUCCESS
>> Call count = 16
    20:04:22.550 : Processing finished. Status = FAILURE
    20:04:23.556 : Processing finished. Status = FAILURE
    Final Result = FAILURE
>> Call count = 17
    20:04:23.557 : Processing finished. Status = SUCCESS
    Final Result = SUCCESS
>> Call count = 18
    20:04:23.557 : Processing finished. Status = BadProcessingException
    20:04:24.561 : Processing finished. Status = SUCCESS
    Final Result = SUCCESS
>> Call count = 19
    20:04:24.561 : Processing finished. Status = SUCCESS
    Final Result = SUCCESS
>> Call count = 20
    20:04:24.561 : Processing finished. Status = SUCCESS
    Final Result = SUCCESS

```

## springboot自动配置

配置在application.yml中

* configs：retry配置，可以配置多个
  * retryExceptions：以列表的形式配置，需要全限定名称，ignoreExceptions一样配置
  * resultPredicate：自定义的Predicate，需要全限定名称
* instances：retry实例名称，可以配置baseConfig指定具体的retry配置，也可以自行配置

``` yml
resilience4j.retry:
  configs:
    default:
      maxRetryAttempts: 2
      waitDuration: 1s
      resultPredicate: io.ysh.eurekacientconsumer.RetryPredicate
      retryExceptions:
        - io.ysh.eurekacientconsumer.service.BadProcessingException
  instances:
    externalRandomFailService:
      baseConfig: default
      waitDuration: 1s
```

自定义的retryOnResultPredicate：如果返回结果为FAILURE，进行重试

```java
public class RetryPredicate implements Predicate<String> {
    @Override
    public boolean test(String s) {
        return s.equals("FAILURE");
    }
}
```

注解`@Retry`有两个属性：

* name：指定配置文件中retry instance
* fallbackMethod：方法名，如果最大尝试次数后仍返回失败，执行该方法

```java
@Service
public class ExternalRandomFailService {

    @Retry(name = "externalRandomFailService")
    public String callService() {
        double random = Math.random();
        if (random < 0.6) {
            System.out.println("\t" + LocalTime.now().truncatedTo(ChronoUnit.MILLIS) + " : Processing finished. Status = SUCCESS");
            return "SUCCESS";
        } else if (random < 0.8) {
            System.out.println("\t" + LocalTime.now().truncatedTo(ChronoUnit.MILLIS) + " : Processing finished. Status = FAILURE");
            return "FAILURE";
        } else {
            System.out.println("\t" + LocalTime.now().truncatedTo(ChronoUnit.MILLIS) + " : Processing finished. Status = BadProcessingException");
            throw new BadProcessingException("Bad processing");
        }
    }
}
```

测试方法，去掉retryconfig，之间执行服务调用即可，输出结果跟之前的一样

```java
@SpringBootTest
class ExternalRandomFailServiceTest {

    @Autowired
    ExternalRandomFailService service;

    @Test
    void callService() {
        for (int i = 1; i <= 20; i++) {
            System.out.println(">> Call count = " + i);
            try {
                String result = service.callService();
                System.out.println("\tFinal Result = " + result);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```

intervalBiFunction的使用，根据返回修改重试的间隔时间，可以在输出结果中观察到：

* 第一次返回FAILURE，间隔2s后重试
* 第一次返回异常，间隔3s后重试

在config.default中加入配置：

```yml
intervalBiFunction: io.ysh.eurekacientconsumer.CustomIntervalFunction
```

```java
public class CustomIntervalFunction implements IntervalBiFunction<String> {

    @Override
    public Long apply(Integer integer, Either<Throwable, String> either) {
        long duration = 1000;
        if (either.isRight() && either.get().equals("FAILURE")) {
            duration = 2000;
        } else if (either.isLeft() && either.getLeft() instanceof BadProcessingException) {
            duration = 3000;
        }
        return duration;
    }
}
```
