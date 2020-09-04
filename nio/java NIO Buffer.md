# Buffer基本用途

一个Buffer就是一块内存，用于写入、读取数据。实际上的实现就是各种基本类型的数组包装到Buffer对象中，提供了一系列的方法来操作这个数组。

基本读写:

 1. 向Buffer写入数据
 2. 调用buffer.flip()
 3. 读取Buffer中的数据
 4. 调用buffer.clear()

``` java
// 将字符串转换位byte数组放入ByteBuffer
String s = "hello world";
// 创建一个ByteBuffer，容量64个字节
ByteBuffer byteBuffer = ByteBuffer.allocate(64);
// 将字符串byte数组放入ByteBuffer中
byteBuffer.put(s.getBytes());
// 在读取之前需要使用flip()方法
byteBuffer.flip();
// 读取一个byte
System.out.println((char) byteBuffer.get());
// 返回Buffer存储元素的数组
byte[] bytes = byteBuffer.array();
// limit()返回实际存储的最后一个元素的索引
System.out.println(new String(bytes, 0, byteBuffer.limit()));
```

## Buffer内部概念

buffer的字段：Buffer实际上是包装了一个数组，使用以下字段和方法可以更容易的操作数组，这些个字段的具体数值可以通过同名的方法返回

``` java
private int position = 0;
private int limit;
private int capacity;
```

* capacity是Buffer的容量，即数组的大小
* 在写入时，position是下一个写入的索引，limit是Buffer的容量
* 在读取之后调用flip()，position被重置为0，limit被重置为之前的position，这样就可以读取数据了

## Buffer方法

**分配一个Buffer**：调用具体实现类的static方法：allocate(int)，传入的时Buffer的大小

Buffer有多种类型，其中ByteBuffer是最常用的，下面创建了个64字节大小的ByteBuffer

``` java
ByteBuffer byteBuffer = ByteBuffer.allocate(64);
```

**写入数据**：使用put方法写入字节或字节数组

``` java
byteBuffer.put(s.getBytes());
```

**读取数据**：使用get读取单个字节，可以传入byte数组，将数据写入数组；或者使用array方法返回Buffer封装的byte数组

``` java
System.out.println((char) byteBuffer.get());
byte[] bytes = byteBuffer.array();
```

更常用的时从一个channel将数据写入Buffer

``` java
// 数据写入buffer
channel.read(bytebuffer);
// 从buffer读取数据
channel.write(bytebuffer);
```

**flip()**：在写入数据后调用，重置position和limit，然后可以读取Buffer中的数据

``` java
public final Buffer flip() {
    limit = position;
    position = 0;
    mark = -1;
    return this;
}
```

**clear()**：在读取完数据后调用，重置position和limit，然后可以继续写入数据

``` java
public final Buffer clear() {
    position = 0;
    limit = capacity;
    mark = -1;
    return this;
}
```

## Buffer类型

java NIO 有以下几类Buffer：都是对基本类型数组的包装

* ByteBuffer
* MappedByteBuffer
* CharBuffer
* ShortBuffer
* IntBuffer
* LongBuffer
* DoubleBuffer
* FloatBuffer
