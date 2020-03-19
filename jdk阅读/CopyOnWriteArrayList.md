# CopyOnWriteArrayList

CopyONWriteArrayList是ArrayList的线程安全版本，顾名思义在发生write操作时copy数组：write操作会加锁，新建一个数组将旧数组中的内容复制到新数组。read操作没有锁，可能write操作已经创建了新数组，但read操作还在读旧数组，所以read会缺乏一些实时性。

## 方法API

write操作中使用的锁对象，使用的是synchronized关键字，注释中说明在ReentrantLock和builtin monitors两者都能完成任务时，作者更倾向于使用builtin monitors。

``` java
/**
 * The lock protecting all mutators.  (We have a mild preference
 * for builtin monitors over ReentrantLock when either will do.)
 */
final transient Object lock = new Object();
```

使用的数组存储内容，数组值通过get、set方法操作，方法是包访问权限是为了同一个包里的CopyOnWriteArraySet类

``` java
/** The array, accessed only via getArray/setArray. */
private transient volatile Object[] array;

/**
 * Gets the array.  Non-private so as to also be accessible
 * from CopyOnWriteArraySet class.
 */
final Object[] getArray() {
    return array;
}

final void setArray(Object[] a) {
    array = a;
}
```

get操作没有加锁只需要在简单的读取即可

``` java
public E get(int index) {
    return elementAt(getArray(), index);
}

static <E> E elementAt(Object[] a, int index) {
    return (E) a[index];
}
```

在添加元素的操作和基于index的删除操作中，就是简单的加锁然后一系列数组的复制等操作，得到新的数组然后setArray，都是单线程操作。

``` java
public boolean add(E e) {
    synchronized (lock) {
        Object[] es = getArray();
        int len = es.length;
        es = Arrays.copyOf(es, len + 1);
        es[len] = e;
        setArray(es);
        return true;
    }
}
```

这样的一系列api有，都是简单的加锁然后数组复制移动的操作

``` java
public E set(int index, E element)
public boolean add(E e)
public void add(int index, E element)
public E remove(int index)
void removeRange(int fromIndex, int toIndex)
public boolean addAll(Collection<? extends E> c)
public boolean addAll(int index, Collection<? extends E> c)
```

在基于对象的删除操作中，需要遍历数组找到这个对象的index，遍历比较对象消耗较大，如果全部加锁会很影响，所以这里在遍历时没有加锁，在查找到index后基于index进行remove操作。  
当在snapshot数组中查找时，有可能其他线程做了write操作，此时的current数组会跟snapshot不一致，具体有几种情况

* current跟snapshot一致，跳过if代码块，执行数组的复制操作
* current跟snapshot不一致，此时在if代码块中再次查找index，在index前后都有可能出现对象o
  * 在index之前是否出现了对象o，即for循环中的内容，如果找到o即确定新的index，没找到才会执行后面的if
  * 第一个if，index >= len 说明在for已经遍历了整个数组没找到，返回false
  * 第二个if，current[index] == o 判断index处的对象是否是o，是就index没变
  * 如果上面三个操作都没找到o，在current数组中从index开始遍历查找o

在index之前查找时使用了current[i] != snapshot[i]的短路操作比直接在整个current数组里indexOfRange要效率一些。

``` java
public boolean remove(Object o) {
    Object[] snapshot = getArray();
    int index = indexOfRange(o, snapshot, 0, snapshot.length);
    return index >= 0 && remove(o, snapshot, index);
}

private boolean remove(Object o, Object[] snapshot, int index) {
    synchronized (lock) {
        Object[] current = getArray();
        int len = current.length;
        if (snapshot != current) findIndex: {
            int prefix = Math.min(index, len);
            for (int i = 0; i < prefix; i++) {
                // 如果current[i] != snapshot[i]说明current[i]肯定不等于o
                // 直接短路不需要执行equals判断
                if (current[i] != snapshot[i]
                    && Objects.equals(o, current[i])) {
                    index = i;
                    break findIndex;
                }
            }
            if (index >= len)
                return false;
            if (current[index] == o)
                break findIndex;
            index = indexOfRange(o, current, index, len);
            if (index < 0)
                return false;
        }
        Object[] newElements = new Object[len - 1];
        System.arraycopy(current, 0, newElements, 0, index);
        System.arraycopy(current, index + 1,
                         newElements, index,
                         len - index - 1);
        setArray(newElements);
        return true;
    }
}
```

基于对象判断的addIfAbsent方法也使用了snapshot来遍历然后加锁对比current来add

``` java
public boolean addIfAbsent(E e) {
        Object[] snapshot = getArray();
        return indexOfRange(e, snapshot, 0, snapshot.length) < 0
            && addIfAbsent(e, snapshot);
    }

private boolean addIfAbsent(E e, Object[] snapshot) {
    synchronized (lock) {
        Object[] current = getArray();
        int len = current.length;
        if (snapshot != current) {
            int common = Math.min(snapshot.length, len);
            for (int i = 0; i < common; i++)
                // 如果找到 return false 添加失败
                if (current[i] != snapshot[i]
                    && Objects.equals(e, current[i]))
                    return false;
            if (indexOfRange(e, current, common, len) >= 0)
                    return false;
        }
        // 没找到执行数组的复制操作
        Object[] newElements = Arrays.copyOf(current, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    }
}
```

## CopyOnWriteArrayList iterator迭代

COWIterator是基于snapshot的，不支持write相关的操作，会抛出`UnsupportedOperationException`。所以迭代会出现实时性的问题，迭代期间对CopyOnWriteArrayList的修改都不会被体现出来。

``` java
/** Snapshot of the array */
private final Object[] snapshot;
/** Index of element to be returned by subsequent call to next.  */
private int cursor;

COWIterator(Object[] es, int initialCursor) {
    cursor = initialCursor;
    snapshot = es;
}
```
