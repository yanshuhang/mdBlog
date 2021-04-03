# springboot + mysql router + 拦截器实现多数据源切换

## 环境

准备工作:

* `mysql group replication`搭建:xxx, 创建一主两从的mysql集群
* `mysql router`搭建:xxx, 创建读写两个路由
* 提前创建test数据库和user表,表结构只需要自增的主键id和一个name列即可

## springboot应用

创建springboot应用时需要`mysql driver`, `mybatis`, `spring-retry`, `aop`的包

``` xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.18</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
```

配置mybatis的扫描路径和开启spring-retry功能

``` java
@MapperScan(basePackages="com.ysh.mysql.dao")
@EnableRetry
public class MysqlApplication {
```

### 配置文件

数据相关的配置放到了独立的文件`db.yml`中, 通过`application.yml`导入

application.yml导入db.yml

``` yml
spring:
  config:
    import: db.yml
```

db.yml: 配置两个数据源, `primary`用于写入, `secondary`用于读取, 数据库地址是`mysql router`的路由地址端口  
为了方便创建`hikaridatasource`, 把数据库的url driver-class-name username等配置都放到了`hikari`中

``` yml
datasource:
  primary:
    hikari:
      jdbc-url: jdbc:mysql://192.168.184.128:3300/test
      driver-class-name: com.mysql.cj.jdbc.Driver
      username: root
      maximum-pool-size: 12
      pool-name: primary
  secondary:
    hikari:
      jdbc-url: jdbc:mysql://192.168.184.128:3301/test
      driver-class-name: com.mysql.cj.jdbc.Driver
      username: root
      maximum-pool-size: 12
      pool-name: seconday
```

### datasource的创建

创建枚举类`Db`对应`db.yml`中的primary和secondary, 用于对应数据源类型

``` java
public enum Db {

    PRIMARY(0, "primary"),
    SECONDARY(1, "secondary");

    private int code;
    private String db_name;

    Db(int code, String name) {
        this.db_name = name;
        this.code = code;
    }

    public int getCode() {
        return code;
    }

    public void setCode(int code) {
        this.code = code;
    }

    public String getDb_name() {
        return db_name;
    }

    public void setDb_name(String db_name) {
        this.db_name = db_name;
    }
}
```

线程本地变量保存数据源类型,用于切换数据源

``` java
public class DbContextHolder {
    private static final ThreadLocal<Db> DB_ENUM_THREAD_LOCAL = new ThreadLocal<>();

    public static void setDb(Db db) {
        for (Db dbType : Db.values()) {
            if (dbType.equals(db)) {
                DB_ENUM_THREAD_LOCAL.set(dbType);
                return;
            }
        }
        throw new IllegalArgumentException("错误的数据库类型名称");
    }

    public static Db getDb() {
        return DB_ENUM_THREAD_LOCAL.get();
    }

    public static void clear() {
        DB_ENUM_THREAD_LOCAL.remove();
    }
}
```

可以在多数据源中切换的`AbstractRoutingDataSource`这是spring提供的抽象类,我们只需继承并实现`determineCurrentLookupKey`方法即可  
这里根据当前线程的局部变量返回数据源类型, 默认为primary,既可以读也可以写

``` java
public class DbRouting extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        Db db = DbContextHolder.getDb();
        if (db == null) {
            return Db.PRIMARY;
        }
        return db;
    }
}
```

创建多数据源:

``` java
@Configuration
public class DbConfig {

    @Bean(name = "dbRouting")
    public DataSource dbRouting(Environment environment) {
        DbRouting dbRouting = new DbRouting();
        // 保存数据源类型和HikariDataSource的映射关系
        Map<Object, Object> datasources = new HashMap<>(Db.values().length);
        dbRouting.setTargetDataSources(datasources);
        // 根据配置的数据库类型, 循环创建多个HikariDataSource
        for (Db dbType : Db.values()) {
            datasources.put(dbType, buildDataSource(environment, dbType));
        }
        return dbRouting;
    }

    // 根据db.yml中的配置创建HikariDataSource
    private DataSource buildDataSource(Environment environment, Db db) {
        // 通过Binder创建HikariDataSource, 只需要传入db.yml中对应的配置名称即可
        Binder binder = Binder.get(environment);
        String sourceProperty = "datasource." + db.getDb_name() + ".hikari";
        BindResult<HikariDataSource> result = binder.bind(sourceProperty, Bindable.of(HikariDataSource.class));
        return result.get();
    }
}
```

mybatis的`SqlSessionFactory`配置  
注入我们自定义的Datasource: dbRouting

``` java
@Configuration
public class MybatisConfig {
    @Autowired
    @Qualifier("dbRouting")
    DataSource dataSource;

    @Bean
    public SqlSessionFactory sqlSessionFactory() throws Exception {
        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(dataSource);
        return sqlSessionFactoryBean.getObject();
    }
}
```

### 注解+拦截器

自定义注解, 用于dao层的方法上

``` java
@Documented
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DbType {
    Db value();
}
```

拦截器, 用于拦截配置了`DbType`注解的dao层方法, 在方法执行前根据注解内容设置数据源的类型, 在执行完之后清理掉

``` java
@Component
@Aspect
public class DbInterceptor {

    @Pointcut(value = "execution( * com.ysh.mysql.dao..*.*(..)) && @annotation(dbType)")
    public void userDao(DbType dbType) {}

    @Before("userDao(dbType)")
    public void before(DbType dbType) {
        DbContextHolder.setDb(dbType.value());
    }

    @After("userDao(dbType)")
    public void after(DbType dbType) {
        DbContextHolder.clear();
    }
}
```

### 实体类+dao+service层

实体类: getter和setter还有构造器省略

``` java
public class User {
    private int id;
    private String name;
}
```

dao层接口: 由于只是简单的sql,这里没有使用mybatis的xml配置了,简单的mybatis注解功能就可以了, 另外方法上还有自定义的`@DbType`注解, 写方法使用了`Db.PRIMARY`数据源, 读方法使用了`Db.SECONDARY`数据源

``` java
@Repository
public interface UserDao {
    @DbType(Db.SECONDARY)
    @Select("select * from user")
    List<User> selectAll();

    @DbType(Db.SECONDARY)
    @Select("select * from user where id = #{id}")
    User selectById(int id);

    @DbType(Db.PRIMARY)
    @Insert("insert into user values (0, #{name})")
    void insertUser(User user);

    @DbType(Db.PRIMARY)
    @Update("update user set name = #{name} where id = #{id}")
    void update(User user);

    @DbType(Db.PRIMARY)
    @Delete("delete from user where id = #{id}")
    void deleteById(int id);

    @DbType(Db.SECONDARY)
    @Select("select @@hostname")
    String hostname();
}
```

service层: `hostname`方法使用了spring-retry包的`@Retryable`注解  

* value = Exception.class: 指明了方法发生Exception异常时进行重试
* maxAttempts = 2: 最大重试次数, 超过次数不再重试
* backoff = @Backoff(delay = 500)):delay指定了延迟多少时间后进行重试

``` java
@Service
public class UserService {
    @Autowired
    UserDao userDao;

    public List<User> selectAll() {
        return userDao.selectAll();
    }

    public User selectById(int id) {
        return userDao.selectById(id);
    }

    @Retryable(value = Exception.class, maxAttempts = 2, backoff = @Backoff(delay = 500, multiplier = 1, maxDelay = 1000)))
    public String hostname() {
        return userDao.hostname();
    }

    public void insertUser(User user) {
        userDao.insertUser(user);
    }

    public void update(User user) {
        userDao.update(user);
    }

    public void deleteById(int id) {
        userDao.deleteById(id);
    }
}
```

### 测试

* 在数据库中随意插入一些数据

``` java
@Test
public void insert() {
    for (int i = 1; i < 10; i++) {
        User user = new User(0, "dummy #" + i);
        userService.insertUser(user);
    }
}
```

可以观察到使用的时`primary`数据源

``` text
[main] com.zaxxer.hikari.HikariDataSource       : primary - Starting...
[main] com.zaxxer.hikari.HikariDataSource       : primary - Start completed.
```

* 测试数据源的切换

``` java
@Test
public void testMutipleDb() {
    User user = userService.selectById(1);
    user.setName("newName");
    userService.update(user);
    System.out.println(userService.selectById(1));
}
```

可以看到先启用了`secondary`进行查询, 后启用的`primary`进行了更新,最后打印了更新后的user信息

``` java
[main] com.zaxxer.hikari.HikariDataSource       : secondary - Starting...
[main] com.zaxxer.hikari.HikariDataSource       : secondary - Start completed.
使用secondary进行查询
[main] com.zaxxer.hikari.HikariDataSource       : primary - Starting...
[main] com.zaxxer.hikari.HikariDataSource       : primary - Start completed.
使用primary进行更新
User(id=1, name=newName)
```

* 测试`mysql router`的负载均衡  
使用`ConcurrentHashMap`记录每次连接的mysql服务器hostname

``` java
@Test
public void hostname() throws InterruptedException {
    int count = 5000;
    ExecutorService executorService = Executors.newFixedThreadPool(12);
    CountDownLatch countDownLatch = new CountDownLatch(count);
    ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
    for (int i = 0; i < count; i++) {
        executorService.execute(() -> {
            String hostname = userService.hostname();
            map.merge(hostname, 1, Integer::sum);
            countDownLatch.countDown();
        });
    }
    countDownLatch.await();
    System.out.println(map);
}
```

打印map,显示结个mysql服务器使用次数基本还算均衡

``` java
{0015160df212=1610, 96b6ea95827a=1667, 52f27d7c6fb3=1723}
```

* 继续使用上面的hostname方法测试重试功能  
在方法执行时, 手动关闭一个docker容器, 测试结果显示关闭掉的那个连接减少, 但总次数还是正确的, 这里为了有时间关闭容器, 把count改为了15000, 下面的总数是正确的, 说明宕机时执行的连接
经过重试在其他mysql服务器上得到了完成

``` java
{0015160df212=7681, 96b6ea95827a=1538, 52f27d7c6fb3=5781}
```

启动刚才关闭得容器, 然后手动命令将其加入到`group replication`中, 再次测试显示负载均衡没有问题

``` java
{0015160df212=5167, 52f27d7c6fb3=5106, 96b6ea95827a=4727}
```
