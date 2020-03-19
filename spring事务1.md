# springboot之事务注解

数据库事务(`transaction`)是访问并可能操作各种数据项的一个数据库操作序列，这些操作要么全部执行,要么全部不执行，是一个不可分割的工作单位。事务由事务开始与事务结束之间执行的全部数据库操作组成。

## 开启事务

springboot中只需要在方法上加上`@Transactional`注解，该方法即可成为一个事务

## 简单示例

### 准备数据

#### 创建数据表

CREATE TABLE account
(
    id      int AUTO_INCREMENT
        PRIMARY KEY,
    name    varchar(15)    NOT NULL,
    balance decimal(10, 2) NOT NULL
);

#### 插入数据

### 实体、mybatis代码

#### 实体

``` java
@Data
public class Account implements Serializable {
    private Integer id;

    private String name;

    private BigDecimal balance;
}
```

#### dao、service

``` java
@Repository
public interface AccountDao {
    int updateAccount(Integer id, double money);
}
```

```java
public interface TransferService {
    void transfer(Integer out, Integer in, double money);
}
```

``` java
@Service
public class TransferServiceImpl implements TransferService {

    private final AccountDao accountDao;

    public TransferServiceImpl(AccountDao accountDao) {
        this.accountDao = accountDao;
    }

    @Override
    // @Transactional
    public void transfer(Integer out, Integer in, double money) {
        accountDao.updateAccount(out, money*(-1));
        // int i = 1 / 0;
        accountDao.updateAccount(in, money);
    }
}
```

### 测试

测试从id 1的账号转账 100 到 id 2 的账户

``` java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class TransferServiceImplTest {

    @Autowired
    private TransferService transferService;

    @Test
    void transfer() {
        transferService.transfer(1, 2, 100);
    }
}

```

#### 测试情况

* 屏蔽注解和有异常的代码，此时正常转账成功

* 屏蔽注解，开启有异常的代码，在id 1 转出后发生异常， id 2 的转入操作代码没有得到执行  
此时的结果是id 1 少了100 而id 2 却没有增加100

* 开启事务注解，代码发生异常后，事务会回退操作，id 1 2的账号都保持事务执行前的状态
