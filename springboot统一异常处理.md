# spring统一异常处理

使用spring的统一异常处理，我们就不再需要在业务代码中就行显式的捕获异常处理，在dao、service、controller层的异常都由spring进行统一的异常捕获处理

## 基本使用示例

### 异常处理类

使用`@ControllerAdvice`注解定义异常处理类，使用`@ExceptionHandler`匹配具体异常，定义异常处理方法  
使用`@ResponseBody`注解将返回的数据转为json格式，也可以直接使用`@RestControllerAdvice`代替`@ControllerAdvice`和`@ResponseBody`  
当捕捉到`UnknownAccountException`异常时返回一个`ResultEntity`对象，包含错误信息

``` java
@ControllerAdvice
@ResponseBody
public class GlobalExceptionHandler {
    @ExceptionHandler(UnknownAccountException.class)
    public ResultEntity unknownAccountExceptionHandler(HttpServletRequest request, Exception e) {
        return ResultEntity.error(-1, e.getMessage());
    }
}
```

### 自定义异常

简单的自定义异常，只包含异常信息

``` java
public class UnknownAccountException extends RuntimeException {
    public UnknownAccountException(String message) {
        super(message);
    }
}
```

### 简单的service

servie只简单的抛出自定的异常

``` java
@Service
public class LoginService {
    public void login() {
            throw new UnknownAccountException("用户不存在");
    }
}
```

### 自定义返回结构

只包含了返回code和信息，用来测试异常处理返回的数据

``` java
@Data
public class ResultEntity {
    private int code;
    private String message;

    public static ResultEntity success() {
        ResultEntity resultEntity = new ResultEntity();
        resultEntity.setCode(0);
        resultEntity.setMessage("success");
        return resultEntity;
    }

    public static ResultEntity error(int code, String message) {
        ResultEntity resultEntity = new ResultEntity();
        resultEntity.setCode(code);
        resultEntity.setMessage(message);
        return resultEntity;
    }
}
```

### controller

直接调用service方法抛出异常

``` java
@RestController
@RequestMapping("user")
public class UserController {

    private final LoginService loginService;

    public UserController(LoginService loginService) {
        this.loginService = loginService;
    }

    @GetMapping
    public ResultEntity login() {
        loginService.login();
        return ResultEntity.success();
    }
}
```

### 返回数据

可以看到返回的是异常捕捉后处理的对象：`ResultEntity.error`生成的对象

``` json
GET http://localhost:8080/user
Content-Type: application/json

{
  "code": -1,
  "message": "用户不存在"
}
```

如果注释掉controller中抛出异常的方法，返回结果：`ResultEntity.sucess`生成的对象

``` json
GET http://localhost:8080/user
Content-Type: application/json

{
  "code": 0,
  "message": "success"
}
```

### 异常匹配的范围

同一个异常被小范围的异常匹配处理方法和大范围的异常匹配处理方法同时覆盖，会选择小范围的异常处理方法  
在异常类中增加一个新的异常处理方法，自定义的`UnknownAccountException`是`RuntimeException`的子类，异常范围更小、更具体，在抛出`UnknownAccountException`异常时会直接匹配`UnknownAccountException`的处理方法，忽略掉范围更广的异常处理方法  
  
``` java
@ExceptionHandler(RuntimeException.class)
public ResultEntity runtimeExceptionHandler(HttpServletRequest request, Exception e) {
    return ResultEntity.error(-2, "RuntimeException的异常");
}

@ExceptionHandler(Exception.class)
public ResultEntity exceptionHandler(HttpServletRequest request, Exception e) {
    return ResultEntity.error(-2, "Exception的异常");
}
```

当异常匹配不到具体的异常处理方法时，会去匹配在继承树上离该异常最近的异常的处理方法  
注释掉`UnknownAccountException`的异常处理方法，匹配到了继承树上更近的`RuntimeException`的处理方法返回的结果

``` json
GET http://localhost:8080/user
Content-Type: application/json

{
  "code": -2,
  "message": "RuntimeException的异常"
}
```
