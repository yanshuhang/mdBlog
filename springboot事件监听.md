# springboot事件监听

## 自定义事件

自定义事件继承至`ApplicationEvent`，事件类不能注册为spring的组件  
模拟用户注册后发布的用户注册事件：

``` java
public class UserRegisterEvent extends ApplicationEvent {

    private User user;

    public UserRegisterEvent(Object source, User user) {
        super(source);
        this.user = user;
    }

    public User getUser() {
        return user;
    }
}
```

需要的实体类User: 只需要简单的信息

``` java
@Data
public class User {
    private String username;
    private String password;
}
```

## 事件监听器

监听类需要注册为spring组件  
在方法上使用`@EventListener`注解，方法参数为需要监听的事件，可以从事件中获取一些传递过来的信息，这里就简单的打印用户名  
可以和`@Async`异步注解一起使用，这样发布事件的代码可以迅速返回不需要等待监听器的完成

``` java
@Slf4j
@Component
public class UserRegisterEventListener {
    @EventListener
    @Async("taskA")
    public void sendEmail(UserRegisterEvent userRegisterEvent) {
        log.info("发送注册邮件，用户名{} ", userRegisterEvent.getUser().getUsername());
    }
}
```

## 模拟用户注册的代码

### service 发布事件

service中注入了`ApplicationContext`bean使用`publishEvent`方法发布事件

``` java
@Service
public class RegisterService {
    private final ApplicationContext applicationContext;

    public RegisterService(ApplicationContext applicationContext) {
        this.applicationContext = applicationContext;
    }

    public void register(User user) {
        // 省略其它注册操作，发布事件
        applicationContext.publishEvent(new UserRegisterEvent(this, user));
    }
}
```

### controller

``` java
@RestController
@RequestMapping("user")
public class UserController {

    private final RegisterService registerService;

    public UserController(RegisterService registerService) {
        this.registerService = registerService;
    }

    @PostMapping("register")
    public void register(@RequestBody User user) {
        // 简单起见，直接传递用户信息，发布注册事件
        registerService.register(user);
    }
}

```

### 模拟注册

``` json
POST http://localhost:8080/user/register
Accept: */*
Cache-Control: no-cache
Content-Type: application/json

{
  "username": "jack",
  "password": "123456"
}
```

### 代码输出

可以看到监听器成功监听到事件并完成log

``` text
[        taskA-1] c.e.d.l.UserRegisterEventListener        : 发送注册邮件，用户名jack
```
