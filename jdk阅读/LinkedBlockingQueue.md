# 基本原理

`LinkedBlockingQueue`是阻塞队列使用链表的实现，使用读写分离的锁。虽然基于链表在存取时有一定的Node对象创建、销毁的开销，但是读写分裂的锁让LinkedBlockingQueue在大多数并发的场景下吞吐量比ArrayBlockingQueue大。

## 方法分析

存储Node结构,

``` java
static class Node<E> {
    E item;

    /**
        * One of:
        * - the real successor Node
        * - this Node, meaning the successor is head.next
        * - null, meaning there is no successor (this is the last node)
        */
    Node<E> next;

    Node(E x) { item = x; }
}
```
