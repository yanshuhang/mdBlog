# 基本介绍

基于数组、固定大小的BlockingQueue，创建时必须要指定其队列容量，线程安全。所有的操作使用同一个锁。

## 源码细节

**构造器**：必须指定队列大小，可以指定锁是否是公平的(默认不公平，较之公平锁效率更高)，可以在队列创建时从指定容器导入元素

``` java
public ArrayBlockingQueue(int capacity)
public ArrayBlockingQueue(int capacity, boolean fair)
public ArrayBlockingQueue(int capacity, boolean fair, Collection<? extends E> c)
```

**主要字段**：对象数组、存取位置、大小、锁相关变量、保存所有迭代其的变量(允许队列的操作更新迭代器)

``` java
/** 存储元素的数组 */
final Object[] items;

/** 取和存时的索引、队列中元素数量 */
int takeIndex;
int putIndex;
int count;

/** 所有访问使用的锁 */
final ReentrantLock lock;

/** 取出操作的等待条件 */
private final Condition notEmpty;

/** 放入操作的等待条件 */
private final Condition notFull;

/**
 * Shared state for currently active iterators, or null if there
 * are known not to be any.  Allows queue operations to update
 * iterator state.
 */
transient Itrs itrs;
```

**主要方法**：  

* 入队、出队，在数组items中使用putIndex、takeIndex循环存取元素，结束后通知在condition上等待的线程。  

``` java
private void enqueue(E e) {
    final Object[] items = this.items;
    items[putIndex] = e;
    if (++putIndex == items.length) putIndex = 0;
    count++;
    notEmpty.signal();
}
private E dequeue() {
    final Object[] items = this.items;
    @SuppressWarnings("unchecked")
    E e = (E) items[takeIndex];
    items[takeIndex] = null;
    if (++takeIndex == items.length) takeIndex = 0;
    count--;
    if (itrs != null)
        itrs.elementDequeued();
    notFull.signal();
    return e;
}
```

出入队列api方法都是加锁，然后调用enqueue、dequeue方法
|方法|操作|异同|
|----|----|----|----|
|add  |入队    |队列满返回fasle   |
|put  |入队    |队列满等待    |
|offer|入队    |队列满返回false，可传入等待时间    |
|poll |出队    |队列空返回null，可传入等待时间    |
|take |出队    |队列空等待    |

* 基于索引的删除，有3个使用到的地方：两个是迭代器的remove方法中，一个是类的remove指定对象的方法(在遍历找到对象的index后调用)

``` java
void removeAt(final int removeIndex) {
    final Object[] items = this.items;
    // 删除的索引刚好是takeIndex，简单置为null即可
    if (removeIndex == takeIndex) {
        items[takeIndex] = null;
        if (++takeIndex == items.length) takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
    } else {
        // 将removeIndex之后的全都向前移一位，直到putIndex为止
        for (int i = removeIndex, putIndex = this.putIndex;;) {
            int pred = i;
            // 到达数组尾部，循环
            if (++i == items.length) i = 0;
            // 遍历到putIndex前一位，置为null，跳出循环
            if (i == putIndex) {
                items[pred] = null;
                this.putIndex = pred;
                break;
            }
            // 前移一位
            items[pred] = items[i];
        }
        count--;
        if (itrs != null)
            itrs.removedAt(removeIndex);
    }
    notFull.signal();
}

public boolean remove(Object o) {
        if (o == null) return false;
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            if (count > 0) {
                final Object[] items = this.items;
                // 遍历查找o的index
                // 外围的for循环用来确定遍历起点、终点，因为putIndex会小于takeIndex
                // 内部的for循环用来匹配对象
                for (int i = takeIndex, end = putIndex,
                         to = (i < end) ? end : items.length;
                     ; i = 0, to = end) {
                    for (; i < to; i++)
                        if (o.equals(items[i])) {
                            removeAt(i);
                            return true;
                        }
                    if (to == end) break;
                }
            }
            return false;
        } finally {
            lock.unlock();
        }
    }
```

### 迭代器

迭代器构造函数，主要是初始化索引有关的变量，保存当前迭代器对象到itrs单链表

``` java
Itr() {
    lastRet = NONE;
    final ReentrantLock lock = ArrayBlockingQueue.this.lock;
    lock.lock();
    try {
        // 队列为空
        if (count == 0) {
            // assert itrs == null;
            cursor = NONE;
            nextIndex = NONE;
            prevTakeIndex = DETACHED;
        } else {
            // 队列不空，nextItem保存takeIndex处元素，用于next方法第一次返回
            final int takeIndex = ArrayBlockingQueue.this.takeIndex;
            prevTakeIndex = takeIndex;
            nextItem = itemAt(nextIndex = takeIndex);
            cursor = incCursor(takeIndex);
            // 将迭代器对象加到itrs单链表上
            if (itrs == null) {
                itrs = new Itrs(this);
            } else {
                itrs.register(this); // in this order
                itrs.doSomeSweeping(false);
            }
            prevCycles = itrs.cycles;
        }
    } finally {
        lock.unlock();
    }
}
```

由于ArrayBlockingQueue的实现是一个循环数组，在nextItem

``` java
public boolean hasNext() {
    if (nextItem != null)
        return true;
    noNext();
    return false;
}

private void noNext() {
    final ReentrantLock lock = ArrayBlockingQueue.this.lock;
    lock.lock();
    try {
        // assert cursor == NONE;
        // assert nextIndex == NONE;
        if (!isDetached()) {
            // assert lastRet >= 0;
            incorporateDequeues(); // might update lastRet
            if (lastRet >= 0) {
                lastItem = itemAt(lastRet);
                // assert lastItem != null;
                detach();
            }
        }
        // assert isDetached();
        // assert lastRet < 0 ^ lastItem != null;
    } finally {
        lock.unlock();
    }
}
```
