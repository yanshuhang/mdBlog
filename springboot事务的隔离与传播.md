# springboot之事务的隔离与传播

## 事务的隔离

多个事务是并发的访问数据库，并发是必须的但会带来几个问题：

* `脏读`：当事务A正在访问数据，并且对数据进行了修改，而这种修改还没有提交到数据库中，这时事务B也访问这个数据，然后使用了这个数据。如果事务A发生了异常，回滚了数据，那么事务B就读到了错误的数据
* `不可重复读`：事务A多次读同一数据。在事务A的两次读数据之间，事务B提交了修改，那么事务A两次读到的的数据可能是不一样的。这样在一个事务内两次读到的数据是不一样的，因此称为是不可重复读。
* `幻读`：事务A读取某个范围的记录，事务B在该范围插入了新的记录，事务A再次读取该范围的记录，会产生幻行。

多个事务并行时，事务之间处理数据的互斥程度就是事务的隔离级别  
spring使用注解`@Transactional`定义一个事务，使用`isolation`配置隔离级别  
隔离级别等级：

* `ISOLATION.DEFAULT`：使用数据库默认的隔离级别，Mysql 默认采用的 REPEATABLE_READ隔离级别
* `ISOLATION.READ_UNCOMMITTED`：最低的隔离级别，允许读取未提交的数据变更，会发生脏读、不可重复读、幻读
* `ISOLATION.READ_COMMITTED`：允许读取其他事务已提交的数据，会发生不可重复读、幻读
* `ISOLATION.REPEATABLE_READ`：对同一数据的多次读取结果是一致的，会发生幻读
* `ISOLATION.SERIALIZABLE`：事件线性执行，完全不会产生干扰

``` java
@Transactional(isolation = Isolation.DEFAULT)
```

## 事务的传播

spring中事务就是使用`@Transactional`注解的方法，一般用在service层的方法上，方法A中调用方法B,这两个方法都是事务，在事务B执行时当前已存在了事务A，怎样处理这两个事务之间的关系就是事务的传播
spring中事务传播级别：

* `PROPAGATION.REQUIRED`：当前没有事务就新建一个事务，当前存在事务就加入该事务
* `PROPAGATION.SUPPORTS`：当前没有事务就以非事务方式允许，当前存在事务就加入该事务
* `PROPAGATION.MANDATORY`：当前没有事务就抛出异常，当前存在事务就加入该事务
* `PROPAGATION.REQUIRES_NEW`：无论无何都会新建一个事务，如果当前有事务会挂起当前的事务
* `PROPAGATION.NOT_SUPPORTED`：无论如何都以非事务方式允许，如果当前有事务会挂起当前的事务
* `PROPAGATION.NEVER`：无论如何都以非事务方式允许，如果当前有事务会抛出异常
* `PROPAGATION.NESTED`：当前没有事务就新建一个事务，当前存在事务会以嵌套式事务存在

## 事务回滚机制

当事务方法发生异常时会回滚事务执行的数据，默认是 rollbackFor = runtimeException.class，发生运行时异常才回滚，可以更改需要匹配什么异常才回滚  
**注意在当前事务方法内catch了指定回滚的异常，不会发生回滚**

## 传播与回滚

事务的传播中一个事务的回滚也会影响到另外一个事务：  
`PNOT_SUPPORTED NEVER`不以事务执行，`SUPPORTS MANDATORY`跟`REQUIRED`表现一致

* `REQUIRED`：事务会合并到当前事务中成为一个事务，其中的任一个方法报错回滚整个事务都会回滚
* `REQUIRES_NEW`：事务以单独事务运行，与当前事务之间相互独立，当前事务的异常回滚不会影响事务，事务异常回滚会导致当前事务回滚，如果内部事务catch了异常不会引起外部事务回滚
* `NOT_SUPPORTED`：嵌套事务，内部事务之间相互独立，都是外部事务的子事务，外务事务异常回滚会导致所有子事务也回滚，子事务异常回滚也会导致所有事务回滚，子事务异常catch后外部事务和其他子事务正常执行

https://juejin.im/entry/5a8fe57e5188255de201062b 这篇文章对各种情况下异常的回滚做了详细的测试
