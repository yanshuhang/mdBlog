# resilience4j-circuit breaker 熔断器

## 简介

微服务结构中，服务调用其他服务，如果服务提供者不可用，会引起服务调用者不可用，进而引起整个调用链上所有的服务可用，熔断器提供了服务提供者不可用时，服务调用者能够快速返回的机制  
这里简单介绍resilience4j-cricuit breaker的使用，包括在sping boot中使用yml文件配置熔断器和注解`@CircuitBreaker`使用熔断器

## 基本功能

resilence4j-circuit breaker 熔断器包括三个状态：关闭、开启、半开。

* 服务正常情况下熔断器处于关闭状态，一切正常，
* 在熔断器判断服务不可达时进入开启状态，服务将拒绝请求，转而执行fallback方法返回，
* 在开启状态固定时间之后，熔断器进入半开状态，服务将接受固定个数的请求，熔断器根据这些请求执行的失败概率来决定是进入开启状态还是关闭状态

### 熔断器统计失败率

resilience4j 默认是使用基于请求数量的滑动窗口统计失败率，具体数量可配置，默认配置为100，resilience4j只统计最新的100个请求的成功失败情况，如果有新的请求最老的请求会被踢出

## 配置详解

|配置属性 |默认值 |描述 |
|:----|:----|:----|
|failureRateThreshold |50 |百分比,当失败率等于或大于阈值时，熔断器状态从关闭变为开启|
|slowCallRateThreshold |100 |百分比，熔断器把调用时间大于slowCallDurationThreshold的调用视为慢调用，当慢调用比例大于等于阈值时，熔断器开启|
|slowCallDurationThreshold |60000 [ms] |长于该时间的调用被视为慢调用|
|permittedNumberOfCallsInHalfOpenState| 10 |熔断器在半开状态下允许通过的调用次数，通过计算这些调用的失败比例来判断切换为关闭还是开启状态
|maxWaitDurationInHalfOpenState |0 |熔断器在半开状态下的最长等待时间，超过该配置值的话，熔断器会从半开状态恢复为开启状态。配置是0时表示熔断器会一直处于半开状态，直到所有允许通过的访问结束。|
|slidingWindowType |COUNT_BASED |滑动窗口的类型，当熔断器关闭时，将调用的结果记录在滑动窗口中。滑动窗口的类型可以是count-based或time-based。如果滑动窗口类型是COUNT_BASED，将会统计记录最近slidingWindowSize次调用的结果。如果是TIME_BASED，将会统计记录最近slidingWindowSize秒的调用结果。|
|slidingWindowSize| 100 |滑动窗口的大小。|
|minimumNumberOfCalls| 100 |熔断器计算失败率或慢调用率之前所需的最小调用数（每个滑动窗口周期）。例如，如果minimumNumberOfCalls为10，则必须至少记录10个调用，然后才能计算失败率。如果只记录了9次调用，即使所有9次调用都失败，熔断器也不会开启。|
|waitDurationInOpenState |60000 [ms] |熔断器从开启过渡到半开应等待的时间。|
|automaticTransitionFromOpenToHalfOpenEnabled |false |设置为true，则意味着熔断器在waitDurationInOpenState时间后将自动从开启状态过渡到半开状态。设置为false，则只有在waitDurationInOpenState时间后发出调用时才会转换到半开|
|recordExceptions |empty |异常列表，除非通过ignoreExceptions显式忽略，否则与列表中某个匹配或继承的异常都将被视为失败。如果指定异常列表，则所有其他异常均视为成功，除非它们被ignoreExceptions显式忽略。
|ignoreExceptions |empty |异常列表，任何与列表之一匹配或继承的异常既不会被视为失败也不会被视为成功，即使异常是recordExceptions的一部分。|
|recordException |throwable -> true | 一个自定义的Predicate，用于评估异常是否应记录为失败。如果异常应计为失败，则断言必须返回true。如果出断言返回false，应算作成功，除非ignoreExceptions显式忽略异常。|
|ignoreException |throwable -> false |自定义Predicate来判断一个异常是否应该被忽略，如果应忽略异常，则谓词必须返回true。如果异常应算作失败，则断言必须返回false。|

## 演示使用

1. 创建一个模拟的外部服务，前10次调用成功，后面的调用返回异常
2. 自定义CircuitBreakerConfig，修改滑动窗口大小为10，默认情况下50%的调用失败会进入开启状态，修改熔断器从开启到半开启时间为10s
3. 每隔1秒使用熔断器调用一次模拟服务

需要准备的依赖包，resilience4j的spring boot包，里面包括了resilience4j所有功能的包和自动配置功能

``` xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot2</artifactId>
    <version>1.7.0</version>
</dependency>
```

### 模拟服务

服务和自定义的异常：`BadProcessingException`

``` java
@Service
public class ExternalService {
    private int count = 0;

    public String callService() {
        count++;
        if (count <= 10) {
            return "success";
        } else {
            throw new BadProcessingException("request fail.");
        }
    }

}

class BadProcessingException extends RuntimeException {
    public BadProcessingException(String message) {
        super(message);
    }
}
```

### 熔断器配置和使用

在test方法中配置熔断器和调用外部服务

``` java
@SpringBootTest
class CircuitBreakerBasicsTest {
    @Autowired
    ExternalService externalService;

    @Test
    void cbService() {
        // 创建熔断器的自定义配置
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            // 修改滑动窗口大小
            .slidingWindowSize(10)
            // 修改从开启状态到半开的时间
            .waitDurationInOpenState(Duration.ofSeconds(10)).build();
        // 创建熔断器使用自定义配置
        CircuitBreaker circuitBreaker = CircuitBreaker.of("circuitBreaker", config);
        for (int i = 0; i < 50; i++) {
            try {
                Thread.sleep(1000);
                System.out.println("count = " + (i + 1));
                // 熔断器执行外部服务, 记录下服务返回值
                String status = circuitBreaker.executeSupplier(externalService::callService);
                System.out.println(status);
            } catch (Exception e) {
                // 打印异常的全限定名称
                System.err.println(e.getClass().getName());
            } finally {
                // 打印执行过程中熔断器具体的信息
                System.out.println("Successful call count: " + circuitBreaker.getMetrics().getNumberOfSuccessfulCalls()
                    + " | failed call count: " + circuitBreaker.getMetrics().getNumberOfFailedCalls()
                    + " | failure rate %:" + circuitBreaker.getMetrics().getFailureRate()
                    + " | state: " + circuitBreaker.getState());
                System.out.println("-----------------------------------------------------------------------------");
            }
        }
    }
}
```

### 输出结果

* 前10次调用返回成功，熔断器处于关闭状态，滑动窗口中成功的调用次数累积到10次
* 11-15次调用返回自定义的异常BadProcessingException，滑动窗口中失败的调用次数开始增加，成功的次数减少，在第15次调用时达到了50%的失败阈值，熔断器开启
* 16-24次调用返回的是熔断器的异常CallNotPermittedException，调用被拒绝，滑动窗口中的调用次数不会变化
* 第25次调用，此时经过了10s熔断器从开启转为了半开状态，熔断器将接受10次调用，可以看到返回的异常是BadProcessingException
* 之后10次调用全部失败，熔断器又转为开启状态，之后一直在开启和半开之间转换

输出结果太长，只贴出其中的一部分

```text
count = 10
success
Successful call count: 10 | failed call count: 0 | failure rate %:0.0 | state: CLOSED
-----------------------------------------------------------------------------
count = 11
io.ysh.eurekacientconsumer.service.BadProcessingException
Successful call count: 9 | failed call count: 1 | failure rate %:10.0 | state: CLOSED
-----------------------------------------------------------------------------
count = 12
io.ysh.eurekacientconsumer.service.BadProcessingException
Successful call count: 8 | failed call count: 2 | failure rate %:20.0 | state: CLOSED
-----------------------------------------------------------------------------
count = 13
io.ysh.eurekacientconsumer.service.BadProcessingException
Successful call count: 7 | failed call count: 3 | failure rate %:30.0 | state: CLOSED
-----------------------------------------------------------------------------
count = 14
io.ysh.eurekacientconsumer.service.BadProcessingException
Successful call count: 6 | failed call count: 4 | failure rate %:40.0 | state: CLOSED
-----------------------------------------------------------------------------
count = 15
io.ysh.eurekacientconsumer.service.BadProcessingException
Successful call count: 5 | failed call count: 5 | failure rate %:50.0 | state: OPEN
-----------------------------------------------------------------------------
count = 16
io.github.resilience4j.circuitbreaker.CallNotPermittedException
Successful call count: 5 | failed call count: 5 | failure rate %:50.0 | state: OPEN
-----------------------------------------------------------------------------
count = 17
io.github.resilience4j.circuitbreaker.CallNotPermittedException
Successful call count: 5 | failed call count: 5 | failure rate %:50.0 | state: OPEN
-----------------------------------------------------------------------------
count = 18
io.github.resilience4j.circuitbreaker.CallNotPermittedException
Successful call count: 5 | failed call count: 5 | failure rate %:50.0 | state: OPEN
-----------------------------------------------------------------------------
count = 19
io.github.resilience4j.circuitbreaker.CallNotPermittedException
Successful call count: 5 | failed call count: 5 | failure rate %:50.0 | state: OPEN
-----------------------------------------------------------------------------
count = 20
io.github.resilience4j.circuitbreaker.CallNotPermittedException
Successful call count: 5 | failed call count: 5 | failure rate %:50.0 | state: OPEN
-----------------------------------------------------------------------------
count = 21
io.github.resilience4j.circuitbreaker.CallNotPermittedException
Successful call count: 5 | failed call count: 5 | failure rate %:50.0 | state: OPEN
-----------------------------------------------------------------------------
count = 22
io.github.resilience4j.circuitbreaker.CallNotPermittedException
Successful call count: 5 | failed call count: 5 | failure rate %:50.0 | state: OPEN
-----------------------------------------------------------------------------
count = 23
io.github.resilience4j.circuitbreaker.CallNotPermittedException
Successful call count: 5 | failed call count: 5 | failure rate %:50.0 | state: OPEN
-----------------------------------------------------------------------------
count = 24
io.github.resilience4j.circuitbreaker.CallNotPermittedException
Successful call count: 5 | failed call count: 5 | failure rate %:50.0 | state: OPEN
-----------------------------------------------------------------------------
count = 25
io.ysh.eurekacientconsumer.service.BadProcessingException
Successful call count: 0 | failed call count: 1 | failure rate %:-1.0 | state: HALF_OPEN
-----------------------------------------------------------------------------
count = 26
io.ysh.eurekacientconsumer.service.BadProcessingException
Successful call count: 0 | failed call count: 2 | failure rate %:-1.0 | state: HALF_OPEN
-----------------------------------------------------------------------------
```

## springboot 自动配置

配置文件+注解使用熔断器

在application.yml文件中加入如下配置

* configs中是熔断器配置列表，可以配置多个熔断器配置，default是其中一个
* instans中是熔断器的名称，可以配置多个，用于注解中标注service使用的是哪个熔断器，baseConfig指定了使用的配置名称，就是configs中的；也可以指定配置项，会覆盖configs中的同名配置项
* Predicate和Exception需要配置为全限定名称，Predicate只能配置一个，Exceptions可以配置多个，按列表的形式配置；测试代码中没有使用到，这里只是展示下怎样配置

``` yml
resilience4j.circuitbreaker:
  configs:
    default:
      slidingWindowSize: 10
      waitDurationInOpenState: 10s
      failureRateThreshold: 50
  instances:
    externalService:
      baseConfig: default
      permittedNumberOfCallsInHalfOpenState: 10
      recordFailurePredicate: io.github.robwin.exception.RecordFailurePredicate
      recordExceptions:
        - org.springframework.web.client.HttpServerErrorException
        - java.util.concurrent.TimeoutException
        - java.io.IOException
```

在service方法上使用`@CircuitBreaker`注解

* name：指定熔断器的名称，就是配置文件中instances下的`externalService`
* fallbackMethod：熔断器开启时，service拒绝调用，会调用fallback方法返回信息

``` java
@CircuitBreaker(name = "externalService", fallbackMethod = "fallback")
public String callService() {
    count++;
    if (count <= 10) {
        return "success";
    } else {
        throw new BadProcessingException("request fail.");
    }
}

public String fallback(Throwable throwable) {
    return "fallback";
}
```

在下面的测试代码中就不需要自行创建熔断器了，需要打印熔断器具体信息的话需要注入 `CircuitBreakerRegistry` Bean，使用`circuitBreaker`方法传入熔断器名称即可  
下面的测试代码中没有显示的使用熔断器，输出的结果跟之前是一样的

``` java
@SpringBootTest
class CircuitBreakerBasicsTest {
    @Autowired
    ExternalService externalService;

    @Autowired
    CircuitBreakerRegistry circuitBreakerRegistry;

    @Test
    void cbService() {
        // 获取熔断器
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker
        for (int i = 0; i < 50; i++) {
            try {
                Thread.sleep(1000);
                System.out.println("count = " + (i + 1));
                String status = externalService.callService();
                System.out.println(status);
            } catch (Exception e) {
                // 打印异常的全限定名称
                System.err.println(e.getClass().getName());
            } finally {
                ("externalService");
                // 打印执行过程中熔断器具体的信息
                System.out.println("Successful call count: " + circuitBreaker.getMetrics().getNumberOfSuccessfulCalls()
                    + " | failed call count: " + circuitBreaker.getMetrics().getNumberOfFailedCalls()
                    + " | failure rate %:" + circuitBreaker.getMetrics().getFailureRate()
                    + " | state: " + circuitBreaker.getState());
                System.out.println("-----------------------------------------------------------------------------");
            }
        }
    }
}
```
