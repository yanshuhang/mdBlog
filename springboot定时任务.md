# spring定时任务  

## 注解开启定时任务

在主类上使用`@EnableScheduling`注解开启功能

## 简单的定时任务

任务类需要使用`@Component`注解，对象交与spring管理  
定时任务较多时需要使用多线程执行，使用`@Async`注解，类中的方法成为异步任务，每个任务都会在不同的线程中

``` java
@Slf4j
@Async
@Component
public class SchedulingTask {

    @Scheduled(cron = "0/5 * * * * *")
    public void cron() {
        log.info("使用cron {}", System.currentTimeMillis()/1000);
    }

    @Scheduled(fixedDelay = 5000)
    public void fixDelay() {
        log.info("使用fixDelay {}", System.currentTimeMillis()/1000);
    }

    @Scheduled(fixedRate = 5000, initialDelay = 1000)
    public void fixedRate() {
        log.info("使用fixedRate {}", System.currentTimeMillis()/1000);
    }
}
```

### 代码执行结果

直接启动springboot应用，定时任务便开始执行了

``` java
[cTaskExecutor-2] c.e.demo01.anything.SchedulingTask       : 使用fixedRate 1582878398
[cTaskExecutor-3] c.e.demo01.anything.SchedulingTask       : 使用cron 1582878400
[cTaskExecutor-4] c.e.demo01.anything.SchedulingTask       : 使用fixDelay 1582878401
[cTaskExecutor-5] c.e.demo01.anything.SchedulingTask       : 使用fixedRate 1582878402
[cTaskExecutor-6] c.e.demo01.anything.SchedulingTask       : 使用cron 1582878405
[cTaskExecutor-7] c.e.demo01.anything.SchedulingTask       : 使用fixDelay 1582878407
[cTaskExecutor-8] c.e.demo01.anything.SchedulingTask       : 使用fixedRate 1582878407
```

### 定时配置

在上面的定时任务中，我们在方法上使用@Scheduled注解来设置任务的执行时间，并且使用三种属性配置方式：

1. fixedRate：定义一个按一定频率执行的定时任务
2. fixedDelay：定义一个按一定频率执行的定时任务，与上面不同的是，改属性可以配合`initialDelay`， 定义该任务延迟执行时间。
3. cron：通过表达式来配置任务执行时间

## cron表达式

cron表达式由7个部分组成，使用空格隔开，7个部分分别代表的含义：  
`秒 分钟 小时 月中的第几日 月份 星期几 年`  
其中年是可选项

### cron表达式范例

``` java
*/5 * * * * ?  每隔5秒执行一次
0 */1 * * * ?  每隔1分钟执行一次
0 0 23 * * ?  每天23点执行一次
0 0 1 * * ?  每天凌晨1点执行一次：
0 0 1 1 * ?  每月1号凌晨1点执行一次
0 0 23 L * ?  每月最后一天23点执行一次
0 0 1 ? * L  每周星期天凌晨1点实行一次
0 26,29,33 * * * ?  在26分、29分、33分执行一次
0 0 0,13,18,21 * * ? 每天的0点、13点、18点、21点都执行一次
```

### 表达式中值的范围

``` java
字段名                 允许的值                        允许的特殊字符  
秒                     0-59                             , - * /  
分                     0-59                             , - * /  
小时                   0-23                             , - * /  
日                     1-31                             , - * ? / L W C  
月                     1-12 or JAN-DEC                  , - * /  
周几                   1-7 or SUN-SAT                   , - * ? / L C #  
年 (可选字段)           empty, 1970-2099                 , - * /
```

### 表达式中的符号

“*”字符：代表所有可能的值，`0 0 23 * * ?` 每天23点执行一次,即所有可能的天数里都执行一次

“?”字符：表示不确定的值，只用在代表‘日’的两项上：月中的第几日、星期中的第几日，由于这两项的值可能存在冲突的情况，所以在一项使用了具体的值时，另一项需要使用`?`表示不确定

“,”字符：在一个项中指定数个值，在多个指定值触发

“-”字符：指定一个值的范围，相当于多个连续的值使用逗号隔开

“/”字符：指定一个值的增加幅度。n/m表示从n开始，每次增加m，3/20表示每隔20分钟执行一次，“3”表示从第3分钟开始执行

“L”字符：用在日表示一个月中的最后一天，`2L`表示月中的倒数第二天；用在星期表示该月最后一个星期几，`2L`表示该月的最后一个星期2

“W”字符：指定离给定日期最近的工作日(周一到周五)，如“15W”放在每月（day-of-month）字段上表示为“到本月15日最近的工作日”，LW连用表示某个月的最后一个工作日

“#”字符：表示该月第几个周几。6#3表示该月第3个周五  
