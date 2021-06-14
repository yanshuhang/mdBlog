# 概要

Object类是所有类的父类，所有对象、数组都实现了此类的方法。

## 源码

**registerNatives()**:

```java
private static native void registerNatives();
static {
    registerNatives();
}
```

* native方法：没有方法体，方法的实现是C或C++编写的，简而言就是java中声明的可调用的使用C/C++实现的方法

* Object类是所有类的父类，是最新初始化的类，在初始化时会执行上面的静态域和方法，registerNatives()用于将java的native方法与C/C++函数关联起来

**getclass()**:

```java
@HotSpotIntrinsicCandidate
public final native Class<?> getClass();
```

* 返回对象的运行时类，即Class对象
* @HotSpotIntrinsicCandidate注解：Java 9引入的新特性，JDK的源码中，被@HotSpotIntrinsicCandidate标注的方法，在HotSpot中都有一套高效的实现，该高效实现基于CPU指令，运行时，HotSpot维护的高效实现会替代JDK的源码实现，从而获得更高的效率。
* final方法：final修饰的方法不能被子类重写
  * final修饰符的其他作用：1. 修饰的类不能被继承 2. 修饰的变量不能被修改：基本类型值不能被修改，引用类型不能指向其他对象(其指向的对象本身可修改)

**hashcode()**:

```java
@HotSpotIntrinsicCandidate
public native int hashCode();
```

返回对象的hash码，哈希码主要用于哈希表数据结构，例如hashmap

* 在程序执行期间，同一个对象多次调用，必须返回相同的数值，两次不同的程序执行期间不必返回相同的值
* 如果equals方法返回相同，则两个对象必须返回相同的hash值
* 如果equals方法返回不同，两个对象hash值可以是相同的，不同的对象返回不同的hash值可以提高哈希表的性能

**equals()**:

```java
public boolean equals(Object obj) {
    return (this == obj);
}
```

equals方法默认比较对象的引用，即判断是否是同一个对象。如果需要比较对象字段的值需要重写equals方法，同时也必须重写hashcode方法

**clone()**:

```java
@HotSpotIntrinsicCandidate
protected native Object clone() throws CloneNotSupportedException;
```

创建并返回对象的浅拷贝，返回的对象跟当前对象是独立的两个对象，对象的字段如果是基本类型会直接复制值，但如果是引用类型会复制对象的地址值，所以clone返回的对象跟当前对象的引用类型字段指向相同的对象  
深拷贝需要重写clone方法，处理引用类型的对象值的复制

* 对象的类必须实现Cloneable接口，在未实现接口时调用clone方法会抛出CloneNotSupportedException异常

**toString()**:

```java
public String toString() {
    return getClass().getName() + "@" + Integer.toHexString(hashCode());
}
```

返回对象的字符串表示，容易阅读，默认返回由对象的实例的类的名称、符号字符' @ '以及该对象的哈希码的十六进制表示组成，建议所有的子类都重写该方法

**notify(),notifyAll()**:

```java
@HotSpotIntrinsicCandidate
public final native void notify();

@HotSpotIntrinsicCandidate
public final native void notifyAll();
```

唤醒在对象监视器上等待的线程，notify方法选择一个线程唤醒，notifyAll方法唤醒所有等待中的线程，最终只会有一个线程获得对象的锁

**wait()方法**：

```java
public final void wait() throws InterruptedException {
    wait(0L);
}

public final void wait(long timeoutMillis, int nanos) throws InterruptedException {
    if (timeoutMillis < 0) {
        throw new IllegalArgumentException("timeoutMillis value is negative");
    }

    if (nanos < 0 || nanos > 999999) {
        throw new IllegalArgumentException(
                            "nanosecond timeout value out of range");
    }

    if (nanos > 0) {
        timeoutMillis++;
    }

    wait(timeoutMillis);
}

public final native void wait(long timeoutMillis) throws InterruptedException;
```

使当前线程等待，直到被唤醒或者在等待timeoutMillis之后自行唤醒

**finalize()**:

```java
@Deprecated(since="9")
protected void finalize() throws Throwable { }
```

在对象被垃圾回收时会调用该方法，默认返回为空，需要重写，由于可能导致的性能和死锁等问题，该方法在java9中已被不推荐使用

* @Deprecated注解：不推荐使用
