# springboot2.4版本，多配置文件

## yml文件中多配置加载顺序

使用`---`在一个yml文件中分割多个配置，如果启用多个配置中有一样的配置项会相互覆盖，在2.4版本中声明在最后面的会覆盖前面的配置。在2.4之前的版本中取决于`spring.profiles.active`中声明的顺序

## 声明配置

2.4版本之前的用法，使用`spring.profiles`指定配置标识，使用`spring.profiles.active`开启配置

```yml
spring:
  profiles:
    active: dev
---
spring:
  profiles: dev
secret: dev-password
```

2.4版本的用法，使用`spring.config.activate.on-profile`，`spring.profiles.active`不能和它配置在同一个配置块中

```yml
spring:
  profiles:
    active: dev
---
spring:
  config:
    activate:
      on-profile: dev
secret：dev-password
```

## 配置组

2.4版本以前使用`spring.profiles`和`spring.profiles.include`的组合

如下配置中，在dev配置块中，引入了devdb和devmq配置，激活dev配置时，devdb和devmq也会激活

``` yml
spring:
  profiles:
    active:
      - dev
  # 2.4使用spring.profiles.include需要以下配置
  config:
    use-legacy-processing: true
---
spring:
  profiles: dev
  profiles.include: devdb, devmq
secret: dev-password
---
spring:
  profiles: devdb
db: devdb
---
spring:
  profiles: devmq
mq: devmq
```

2.4版本之后，使用`spring.profiles.group`来配置组合，active中可以选择激活dev组或者test组

``` yml
spring:
  profiles:
    active:
      - dev
    group:
      dev:
        - devdb
        - devmq
      test:
        - testdb
        - testmq
---
spring:
  config:
    activate:
      on-profile: dev
secret: dev-password
---
spring:
  config:
    activate:
      on-profile: devdb
db: devdb
---
spring:
  config:
    activate:
      on-profile: devmq
mq: devmq
```
