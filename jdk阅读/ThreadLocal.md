# ThreadLocal

ThreadLocal提供了线程本地变量，每个使用该变量的线程都会初始化一个完全独立的实例副本。ThreadLocal变量通常被`private static`修饰。当一个线程结束，它所使用的所有ThreadLocal变量相对的实例副本都会被回收。

## 基础使用

``` java
public class UserInfoHandler {
    private static ThreadLocal<UserInfo> threadLocal = ThreadLocal.withInitial(() -> new UserInfo());

    private static void test() {
        for (int i = 0; i < 5; i++) {
            int j = i;
            new Thread(() -> {
                UserInfo userInfo = threadLocal.get();
                userInfo.setName("user" + j);
                System.out.println("[" + Thread.currentThread().getName() + "]: " + userInfo);
            }).start();
        }
    }

    public static void main(String[] args) {
        test();
    }
}

@Data
class UserInfo {
    String id;
    String name;

    public UserInfo() {
        this.id = UUID.randomUUID().toString();
    }
}
```

每个线程都有自己独立的本地变量，相互之间不会影响

``` java
[Thread-4]: UserInfo(id=a456e9df-7267-41e3-bfc4-09c835dc2131, name=user4)
[Thread-0]: UserInfo(id=b69aa145-c879-4c55-8190-85148f839571, name=user0)
[Thread-2]: UserInfo(id=d4ac0367-3077-488b-86f0-5c481dd51c8b, name=user2)
[Thread-3]: UserInfo(id=a3f1a000-165a-4c61-b521-c4c4500e3862, name=user3)
[Thread-1]: UserInfo(id=9771ad8c-7c7b-4d23-b59d-92880b3406ad, name=user1)
```

## 主要方法

Thread对象持有`ThreadLocal.ThreadLocalMap threadLocals`默认为null，ThreadLocalMap是ThreadLocal类内部定义一个哈希表，只用于维护线程局部值，保存ThreadLocal对象和局部值的键值对。

**set方法**：设置当前线程中ThreadLocal对应的value值

``` java
public void set(T value) {
    // 获取当前执行线程
    Thread t = Thread.currentThread();
    // 获取线程t中的ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        map.set(this, value);
    } else {
        createMap(t, value);
    }
}
```

**getMap，createMap方法**：直接返回线程t的字段，创建ThredLocalMap赋值给线程t的threadLocals

``` java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

**get方法**：如果ThreadLocalMap中有当前ThreadLocal对象的key则返回value，否则调用`setInitialValue`方法设置初始value值

``` java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

**setInitialValue方法**：设置初始值，不需要使用set方法也可以get到value。调用`initialValue()`获取value，然后有map就设置，没有就创建并设置

``` java
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        map.set(this, value);
    } else {
        createMap(t, value);
    }
    if (this instanceof TerminatingThreadLocal) {
        TerminatingThreadLocal.register((TerminatingThreadLocal<?>) this);
    }
    return value;
}
```

**initialValue**是ThreadLocal的protected方法，默认返回null，重写该方法设置初始值

``` java
protected T initialValue() {
    return null;
}
```

重写initialValue方法示例

``` java
ThreadLocal<String> mThreadLocal = new ThreadLocal<String>() {
    @Override
    protected String initialValue() {
      return Thread.currentThread().getName();
    }
};
```

ThreadLocal有已经定义好的重写initialValue方法的类`SuppliedThreadLocal`，使用方法`withIntial`即可，而且可以写成lambda的形式

``` java
static final class SuppliedThreadLocal<T> extends ThreadLocal<T> {

    private final Supplier<? extends T> supplier;

    SuppliedThreadLocal(Supplier<? extends T> supplier) {
        this.supplier = Objects.requireNonNull(supplier);
    }

    @Override
    protected T initialValue() {
        return supplier.get();
    }
}

public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
    return new SuppliedThreadLocal<>(supplier);
}
```

## 搭配线程池

当线程结束时，它所使用的所有的ThreadLocal变量相对的实例副本都会被回收。但使用线程池时，线程是复用的，被复用的线程会保留其上次使用的ThreadLocal变量及相对的实例。

使用线程池来执行任务：

``` java
private static void testWithPool() {
    ExecutorService service = Executors.newFixedThreadPool(2);
    for (int i = 0; i < 5; i++) {
        int j = i;
        service.execute(() -> {
            UserInfo userInfo = threadLocal.get();
            userInfo.setName("user" + j);
            System.out.println("[" + Thread.currentThread().getName() + "]: " + userInfo);
        });
    }
    try {
        Thread.sleep(100);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    service.shutdown();
}
```

执行结果，线程池中2个线程复用导致了UserInfo的id重复，遗留了上次任务的值

``` java
[pool-1-thread-2]: UserInfo(id=3836447b-e7cf-4be5-b68e-f867405124f3, name=user1)
[pool-1-thread-1]: UserInfo(id=ef42d5b5-233e-4fbc-a3a6-c14ce4dc0948, name=user0)
[pool-1-thread-2]: UserInfo(id=3836447b-e7cf-4be5-b68e-f867405124f3, name=user2)
[pool-1-thread-1]: UserInfo(id=ef42d5b5-233e-4fbc-a3a6-c14ce4dc0948, name=user3)
[pool-1-thread-2]: UserInfo(id=3836447b-e7cf-4be5-b68e-f867405124f3, name=user4)
```

如果不想出现这种情况，解决办法是在执行任务之后调用ThreadLocal的remove方法，将ThreadLocal从当前线程的ThreadLocalMap中删除

``` java
service.execute(() -> {
    try {
        UserInfo userInfo = threadLocal.get();
        userInfo.setName("user" + j);
        System.out.println("[" + Thread.currentThread().getName() + "]: " + userInfo);
    } finally {
        threadLocal.remove();
    }
});
```

有些使用场景却是需要线程池复用线程时保留上次任务的ThreadLocal，`simpleDateFormat`就是一个；`simpleDateFormat`在多线程并发时不是安全的，使用了ThreadLocal封装之后解决了多线程并发不安全的问题，但是每启用一个线程都会创建一个`simpleDateFormat`对象，这时候在使用线程池时保留上次任务中的`simpleDateFormat`对象，可以减少重复创建对象，就不能使用remove了。
