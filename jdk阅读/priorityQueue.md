# 优先级队列-priorityQueue

## 基本介绍

* 基于最小堆的优先级队列，优先级队列中的元素根据其Comparable接口方法或Comparator提供的方法进行比较排序，具体取决于使用的构造器。
* 不能存储`null`
* 头部是指定排序的最小元素
* 优先级队列是无界的，自动扩容，其内部实现是数组，最大容量为数组的最大容量

## 字段

内部存储结构是数组，`queue[n]`的左右字节点是`queue[2n+1]`和`queue[2n+2]`，每个节点都小于或等于它的字节点，最小值是`queue[0]`

``` java
transient Object[] queue;
```

## 构造器

* 默认构造器：初始大小为11，其元素必须实现Comparable接口
* 可以指定初始大小和比较器
* 可以从其他队列接受初始元素，然后构建堆

``` java
public PriorityQueue()

public PriorityQueue(int initialCapacity)

public PriorityQueue(Comparator<? super E> comparator)

public PriorityQueue(int initialCapacity, Comparator<? super E> comparator)

public PriorityQueue(Collection<? extends E> c)

public PriorityQueue(PriorityQueue<? extends E> c)

public PriorityQueue(SortedSet<? extends E> c)
```

## 主要操作

**add、offer**：向队列中添加元素  
元素被添加到队列的尾部，此时最小堆的结构被破坏需要调用`siftUp`方法调整最小堆

``` java
public boolean add(E e) {
    return offer(e);
}

public boolean offer(E e) {
    // 不能添加null
    if (e == null)
        throw new NullPointerException();
    modCount++;
    int i = size;
    // 元素数量大于等于数组大小，扩容
    if (i >= queue.length)
        grow(i + 1);
    size = i + 1;
    // 队列为空，直接放到头部
    if (i == 0)
        queue[0] = e;
    // 否则添加到队列尾部，然后调整使其成为一个最小堆
    else
        siftUp(i, e);
    return true;
}
```

**siftuup**：自下而上的调整堆使其成为一个最小堆  
根据`Comparator`或`Comparable`比较有两个版本的方法，逻辑是一样的：

* 跟其父节点进行比较
* 如果小于父节点，跟父节点交换
* 直到大于等于父节点或者到到堆顶(即k==0),结束循环

``` java
private void siftUp(int k, E x) {
    if (comparator != null)
        siftUpUsingComparator(k, x);
    else
        siftUpComparable(k, x);
}

private void siftUpUsingComparator(int k, E x) {
    while (k > 0) {
        int parent = (k - 1) >>> 1;
        Object e = queue[parent];
        // 大于等于父节点，结束循环
        if (comparator.compare(x, (E) e) >= 0)
            break;
        // 父节点下移
        queue[k] = e;
        // x的插入位置上移
        k = parent;
    }
    // 循环结束找到了插入的位置，x插入到数组
    queue[k] = x;
}
```

**poll**：删除并返回头部元素  
头部被删除之后，由于失去了堆顶，最小堆结构被破坏，需要调整堆结构：

* 将最后一个元素从堆顶自上而下的寻找合适的位置插入

``` java
public E poll() {
    if (size == 0)
        return null;
    int s = --size;
    modCount++;
    E result = (E) queue[0];
    E x = (E) queue[s];
    queue[s] = null;
    if (s != 0)
        siftDown(0, x);
    return result;
}
```

**siftDown**：自上而下的调整堆
跟siftUp一样有两个版本但逻辑是一样的：

* 跟左右节点中的较小的比较
* 如果比子节点大，交换位置
* 直到比子节点小，或者没有子节点

由于向下调整到没有子节点即可结束循环，所有只需要遍历一半即可(由堆的特性决定，有一半的节点为叶节点，没有子节点)

``` java
private void siftDown(int k, E x) {
    if (comparator != null)
        siftDownUsingComparator(k, x);
    else
        siftDownComparable(k, x);
}

private void siftDownUsingComparator(int k, E x) {
    int half = size >>> 1;
    // 只需遍历非叶节点
    while (k < half) {
        // 左节点的索引
        int child = (k << 1) + 1;
        Object c = queue[child];
        int right = child + 1;
        // 比较去的左右节点中的较小的一个
        if (right < size &&
            comparator.compare((E) c, (E) queue[right]) > 0)
            c = queue[child = right];
        // 比子节点小，符合最小堆结构，结束循环
        if (comparator.compare(x, (E) c) <= 0)
            break;
        // 不符合，则较小的节点上移
        queue[k] = c;
        // x的插入位置下移
        k = child;
    }
    // 循环结束找到位置，x插入到数组
    queue[k] = x;
}
```

## 构建堆

在构建优先级队列时，可以通过构造器传入队列元素，这些元素是不符合最小堆的结构的，需要调整堆：

* 从后向前遍历堆的非叶节点，执行siftDown操作
* 这样每个以非叶节点为堆顶的堆都被调整为符合结构的最小堆
* 直到堆顶，循环完成

``` java
private void heapify() {
    for (int i = (size >>> 1) - 1; i >= 0; i--)
        siftDown(i, (E) queue[i]);
}
```

## 扩容

优先级队列中的元素数量大于数组容量时，进行扩容：

* 旧容量小于64时，每次容量翻倍
* 大于64时，每次容量1.5倍

``` java
private void grow(int minCapacity) {
    int oldCapacity = queue.length;
    // Double size if small; else grow by 50%
    int newCapacity = oldCapacity + ((oldCapacity < 64) ?
                                        (oldCapacity + 2) :
                                        (oldCapacity >> 1));
    // overflow-conscious code
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    queue = Arrays.copyOf(queue, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```
