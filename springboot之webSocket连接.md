# springboot之webSocket连接

## 服务端

### 开启功能

功能开启需要在pom.xml中导入包，并配置Configuration创建`ServerEndpointExporter`bean

``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

``` java
@Configuration
public class WebSocketConfig {
    @Bean
    public ServerEndpointExporter serverEndpointExporter() {
        return new ServerEndpointExporter();
    }
}
```

### 配置WebScoket端点

使用`@ServerEndpoint`注解，与controller差不多一样配上匹配路径，方法使用`@OnOpen @OnMessage @OnClose`等注解对应WebSocket开启连接、接受信息、关闭连接的操作，这里使用了自定义的`WebSocketSessionManager`类来完成具体的操作  
**需要注意的是在向WebSocket类注入bean时需要将字段设置为static成为类变量，因为spring管理的bean默认是单例的，而WebSocket是多对象的**

``` java
@ServerEndpoint("/ws/{username}")
@Slf4j
@Component
public class WebSocketController {

    // 具体工作交由 WebSocketSessionManager
    private static WebSocketSessionManager manager;

    @Autowired
    public void setManager(WebSocketSessionManager manager) {
        WebSocketController.manager = manager;
    }

    // 匹配uri上的参数使用的是@PathParam注解，跟controller的注解有区别
    @OnOpen
    public void onOpen(Session session, @PathParam("username") String username) {
        manager.addSession(session, username);
    }

    @OnMessage
    public void onMessage(Session session, String msg) {
        log.info("session {} send msg {}", session.getId(), msg);
    }

    @OnClose
    public void onClose(@PathParam("username") String username) {
        manager.removeSession(username);
    }
}
```

这里就简单的使用用户名来对应session保存在map中，注意需要使用同步的容器

``` java
@Slf4j
@Component
public class WebSocketSessionManager {
    private ConcurrentHashMap<String, Session> sessions = new ConcurrentHashMap<>();

    // 必须是同步的容器
    public ConcurrentHashMap<String, Session> getSessions() {
        return sessions;
    }

    // 注入 ObjectMapper 序列化对象成 json 字符串
    private final ObjectMapper objectMapper;

    public WebSocketSessionManager(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    public void addSession(Session session, String username) {
        if (!sessions.containsKey(username)) {
            sessions.put(username, session);
            log.info("用户 {} 加入", username);
            refeshCount(sessions.size());
        }
    }

    public void removeSession(String username) {
        if (sessions.containsKey(username)) {
            sessions.remove(username);
            log.info("用户 {} 退出", username);
            refeshCount(sessions.size());
        }
    }

    public void sendMessageToUser(String username, String message) {
        Session session = sessions.get(username);
        try {
            session.getBasicRemote().sendText(message);
        } catch (IOException e) {
            log.error("webSocket信息发送异常", e);
        }
    }

    public void sendMessageToAll(String username, String message) {
        HashMap<String, Object> data = new HashMap<>();
        data.put("code", 1);
        data.put("username", username);
        data.put("message", message);
        String dataJson = "";
        try {
            dataJson = objectMapper.writeValueAsString(data);
        } catch (JsonProcessingException e) {
            e.printStackTrace();
        }
        for (Session session : sessions.values()) {
            try {
                session.getBasicRemote().sendText(dataJson);
            } catch (IOException e) {
                log.error("webSocket信息发送异常", e);
            }
        }
    }

    public void refeshCount(int count) {
        HashMap<String, Object> data = new HashMap<>();
        data.put("code", 2);
        data.put("count", count);
        String dataJson = "";
        try {
            dataJson = objectMapper.writeValueAsString(data);
        } catch (JsonProcessingException e) {
            e.printStackTrace();
        }
        for (Session session : sessions.values()) {
            try {
                session.getBasicRemote().sendText(dataJson);
            } catch (IOException e) {
                log.error("webSocket信息发送异常", e);
            }
        }
    }
}

```

### 实体类、controller

简单的实体，其实只用到了username

``` java
@Data
public class User {
    private String username;
    private String password;
}
```

controller包含2个方法：

1. 在第一次访问页面时返回当前人数
2. 接收聊天信息，转交webSocketSessionManager发送给所有人

``` java
@Slf4j
@RestController
public class ChatRoomController {
    private final WebSocketSessionManager webSocketSessionManager;

    public ChatRoomController(WebSocketSessionManager webSocketSessionManager) {
        this.webSocketSessionManager = webSocketSessionManager;
    }

    @GetMapping("count")
    public int count() {
        return webSocketSessionManager.getSessions().size();
    }

    @PostMapping("message")
    public void sendMessage(@RequestBody Map<String, Object> map) {
        log.info("user: {}, message: {}", map.get("username"), map.get("message"));
        String message = (String) map.get("message");
        String username = (String) map.get("username");
        webSocketSessionManager.sendMessageToAll(username, message);
    }
}
```

## 客户端

客户端文件index.html放在resources目录下的static文件夹中，可以通过`http://localhost:8080/index.html`直接访问  
主要引入使用到了：

1. [vue](https://cn.vuejs.org/v2/guide/)：框架
2. [element-ui](https://element.eleme.cn/#/zh-CN/component/installation)：ui库
3. [axios](https://github.com/axios/axios)：发送http请求

``` html
<html>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8"/>
<title>index</title>
<body>
<!-- 引入vue -->
<script src="https://cdn.jsdelivr.net/npm/vue/dist/vue.js"></script>
<!-- 引入样式 -->
<link rel="stylesheet" href="https://unpkg.com/element-ui/lib/theme-chalk/index.css">
<!-- 引入组件库 -->
<script src="https://unpkg.com/element-ui/lib/index.js"></script>
<!-- 引入axios -->
<script src="https://unpkg.com/axios/dist/axios.min.js"></script>
<div id="app">
    <el-container>
        <el-header></el-header>
        <el-main>
            <el-row :gutter="20">
                <el-col :span="2" :offset="6">
                    <el-input v-model="username" placeholder="输入姓名" v-bind:disabled="enterDisabled"></el-input>
                </el-col>
                <el-col :span="2">
                    <el-button type="primary" v-on:click="connectWs" v-bind:disabled="enterDisabled">进入聊天室</el-button>
                </el-col>
                <el-col :span="4" style="padding-top: 18px">
                    当前人数：{{count}}
                </el-col>
            </el-row>
            <el-row :gutter="20">
                <el-col :span="10" :offset="6">
                    <el-card id="chat" shadow="never">
                        <div v-for="text in textList" :key="text.id">
                            <el-row>
                                <el-col :span="2">
                                    <div style="color: deepskyblue">{{text.username}} :</div>
                                </el-col>
                                <el-col :span="22">
                                    <div>{{text.message}}</div>
                                </el-col>
                            </el-row>
                        </div>
                    </el-card>

                </el-col>
            </el-row>

            <el-row :gutter="20">
                <el-col :span="8" :offset="6">
                    <el-input v-model="message" placeholder="请输入内容"></el-input>
                </el-col>

                <el-col :span="2">
                    <el-button type="primary" v-on:click="sendMessage" v-bind:disabled="sendDisabled">发送消息</el-button>
                </el-col>

            </el-row>
        </el-main>
    </el-container>
</div>
</body>
<script>
    let app = new Vue({
        el: '#app',
        data: {
            username: "",
            message: "",
            count: 0,
            enterDisabled: false,
            sendDisabled: true,
            textList: [],
            websocket: null
        },
        methods: {
            connectWs() {
                this.websocket = new WebSocket(
                    "ws://localhost:8080/ws/" +
                    this.username
                );
                this.websocket.onopen = this.websocketonopen;
                this.websocket.onmessage = this.websocketonmessage;
                this.websocket.onerror = this.websocketonerror;
                this.websocket.onclose = this.websocketclose;
                this.sendDisabled = false
                this.enterDisabled = true
            },
            websocketonopen() {
                //连接建立之后执行send方法发送数据
            },
            websocketonerror() {
                //连接建立失败重连
                this.connectWs();
            },
            websocketonmessage(e) {
                //数据接收
                let data = JSON.parse(e.data);
                switch (data.code) {
                    case 1:
                        this.textList.push(data)
                        this.scrollToBottom();
                        console.log(data)
                        break
                    case 2:
                        console.log(data)
                        this.count = data.count
                        break
                }
            },
            websocketsend(Data) {
                //数据发送
                this.websocket.send(Data);
            },
            websocketclose(e) {
                //关闭
                console.log("断开连接", e);
            },
            sendMessage: function () {
                axios.post('message', {
                    username: this.username,
                    message: this.message
                }).then((response) => {
                    this.message = ""
                })
            },
            scrollToBottom: function () {
                let div = document.getElementById('chat')
                div.scrollTop = div.scrollHeight
            },
        },
        created: function () {
            axios.get("/count").then((response) => {
                this.count = response.data
                console.log(this.count)
                console.log(response.data)
            })
        }

    });
</script>
<style>
    .el-row {
        margin-bottom: 20px;

    }

    body {
        margin: 0px;
        margin-top: 100px;
    }

    .el-card {
        height: 400px;
        overflow-y: auto;
    }

    .el-card__body {
        padding-top: 5px;
        padding-right: 2px;
    }
</style>
</html>
```

## 简单演示
