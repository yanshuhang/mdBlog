# 基本使用

Thread执行任务一般是没有返回值的，如果在线程执行完后需要获取返回值，要用到Callable接口，有两种使用方式

1. 创建FutureTask，交给Thread对象执行，futureTask对象会保存返回值
2. 使用线程池，ExecutorService.submit方法，返回一个Future对象

``` java
// 显示的Thread执行
FutureTask<String> futureTask = new FutureTask<>(() -> "futureTask message");
new Thread(futureTask).start();
System.out.println(futureTask.get());

// 线程池执行
ExecutorService service = Executors.newCachedThreadPool();
Future<String> future = service.submit(() -> "just future");
System.out.println(future.get());
```

实际上线程池的submit方法也是通过创建FutureTask对象交由线程执行

``` java
// 实际调用的是AbstractExecutorService类中的sumit方法
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    // newTaskFor方法返回一个FutureTask对象
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}

protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    return new FutureTask<T>(callable);
}
```

## FutureTask执行原理

Thread构造器只接收Runnable对象，无法接收Callable，而FutureTask就是中间的一层，FutureTask实现了RunnableFuture接口，该接口又继承了Runnable、Future接口，所以FutureTask就是一个Runnable对象，在包装了Callable之后可以交与Thread执行，像是适配器模式。

既然FutureTask是一个Runnable对象，终点就在于它的run方法，在callable执行之后获取返回值result，然后set(result)方法将返回值保存到对象的outcome字段了，这样就可以通过get()获取都返回值了。

``` java
public void run() {
    if (state != NEW ||
        !RUNNER.compareAndSet(this, null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                // 获得返回值
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                // 保存到对象字段
                set(result);
        }
    } finally {
        runner = null;
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}

protected void set(V v) {
    if (STATE.compareAndSet(this, NEW, COMPLETING)) {
        outcome = v;
        STATE.setRelease(this, NORMAL); // final state
        finishCompletion();
    }
}
```

FutureTask还可以接收Runnable对象，但是必须要手动指定返回值

``` java
 public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;
}
```

这里的runnable通过Executors.callable方法包装为一个RunnableAdapter对象，这是一个典型的适配器模式

``` java
public static <T> Callable<T> callable(Runnable task, T result) {
    if (task == null)
        throw new NullPointerException();
    return new RunnableAdapter<T>(task, result);
}
```

RunnableAdapter是Executors的内部类，实现了Callable接口，在执行call方法时就是调用task的run方法，然后返回我们手动指定的返回值

``` java
public T call() {
    task.run();
    return result;
}
```
