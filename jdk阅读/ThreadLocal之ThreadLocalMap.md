# ThreadLocalMap原理

ThreadLocalMap定义在ThreadLocal类内，被Thread类使用存储ThreadLocal对象和对应的值。虽然是表结构，ThreadLocalMap并没有继承Map，而是自行实现。

## 存储结构

使用Entry数组存储数据，Entry是ThreadLocal和value的映射结构，跟HashMap不同的是Entry并没有前驱、后继等字段，Entry对象之间没有关联，只是全部存储在table中。

``` java
private Entry[] table;
```

Entry对象弱引用ThreadLocal对象，意味着ThreadLocal对象在没有强引用可达时会被垃圾回收

``` java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

## 主要方法

ThreadLocalMap方法处理中有两个重要的概念：

* stale entry：废弃的条目，就是key为null的entry，需要清理掉
* run：table中两个null之间的一群连续的entry，hash值相同的key会存储在同一个run中，便于查找，不需要遍历整个table；同一个run中会有不同hash的entry，但相同hash的不会在不同的run中

**get方法**：如果在hash之后直接找到就返回，否则调用getEntryAfterMiss方法继续查找

``` java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```

**getEntryAfterMiss方法**：while循环判断entry

* key匹配到，返回entry
* key为null，说明这是一个stale entry，调用expungeStaleEntry方法清理
* key没匹配到、也不是stale entry，则索引+1循环判断下一个entry

while循环结束未找到匹配的entry，返回null

``` java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;
    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

**expungeStaleEntry方法**：注意到在`getEntryAfterMiss`方法的循环判断中如果调用了`expungeStaleEntry`方法，索引不会nextIndex，而是直接e = tab[i]，是因为在该方法调用之后当前索引后面的同一个run中的entry都被重新hash寻找存储的位置，之后会出现两种情况

1. 当前位置存储了新的entry（重新hash之后被放到这里），需要再次判断，不能跳过，所以索引没有nextIndex
2. 当前位置为null，成为了所处run的终点，也不能跳过需要重新判断

本方法返回一个索引，是删除stale entry之后的第一个null的索引，也就是该run的终点，在stale entry和该索引之间的entry都会被检查

1. 是stale entry就会被删除
2. 非stale entry如果hash到的索引跟当前存储的索引不一致，则重新分配存储位置

``` java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    // 删除stale entry
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;
    // 重新hash后面的entry直到遇到null
    // 由于删除了stale entry，其位置变成了null，本来在同一个run中的entries被分开了，所有要重新hash该run中的entries，保证hash值相同的entries在同一个run中
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        // 删除废弃的entry
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            // 如果hash得到的索引h跟当前存储的索引i不一致，则从索引h开始遍历找到一个null的位置存储entry，这样entry就跟它hash值相同的entries在一个run里了
            if (h != i) {
                tab[i] = null;
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

**set方法**：在发生碰撞时和HashMap的处理逻辑不同，HashMap是在碰撞的位置创建链表存储hash到同一个索引的节点，而这里是在数组中往后遍历找到合适的Entry存储（for循环中的内容）或者创建一个新Entry存储

``` java
private void set(ThreadLocal<?> key, Object value) {
    // 1.找到key在数组中的索引，跟HashMap中一样数组大小都是设定为2的幂次方，方便定位索引
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    // 2.从找到的索引开始遍历，如果entry为null结束循环跳转第3步，
    // entry不为null，判断Entry的key和ThreadLocal是否是同一个对象，如果是替换value，方法返回
    // 如果entry的key为null，调用replaceStaleEntry方法返回
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        if (k == key) {
            e.value = value;
            return;
        }
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    // 3.创建Entry，赋值当前索引
    tab[i] = new Entry(key, value);
    int sz = ++size;
    // 4.清理陈旧的entry，如果方法返回false即没有清理，而且size达到了阈值，扩展table
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

在for循环中方法返回有两个可能条件：

* 找到了存储ThreadLocal的Entry，此时替换value后返回
* 找一个key为null的Entry，执行方法replaceStaleEntry后返回

**replaceStaleEntry方法**：寻找key-value的存储位置并存储

``` java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    // 向前遍历直到null，找到最靠前stale entry索引
    // 也就是在该run中的第一个stale entry的索引
    // 在下面的代码中slotToExpunge都会存储该run中第一个stale entry的索引
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
            (e = tab[i]) != null;
            i = prevIndex(i, len))
        if (e.get() == null)
            slotToExpunge = i;

    // 向后遍历直到null
    for (int i = nextIndex(staleSlot, len);
            (e = tab[i]) != null;
            i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        // 如果当前entry的key匹配，则替换value
        if (k == key) {
            e.value = value;
            // 然后将该索引i的entry和staleSlot交换，之后索引i存储的是一个stale entry
            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            // 如果向前遍历没找到stale entry，即staleSlot就是该run中的第一个stale entry索引，由于交换了位置则i就是第一个stale entry的索引
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            // 第一个stale entry索引交与expungeStaleEntry方法，expungeStaleEntry方法会删除该run中此索引后面的所有stale entry并重新hash非stale entry，然后返回该run的新终点的索引交由cleanSomeSlots处理
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        // 如果在向前遍历时没有找到stale entry，staleSlot之后地第一个stale entry就是该run中的第一个stale entry，因为不论如何staleSlot处都会存储key-value的entry
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    // 如果该run中没找到匹配的key，释放staleSlot处的value，创建entry存储在staleSlot处
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    // 如果for循环中没有匹配key，也就没有执行其中的删除stale entry的代码。
    // 所以判断是否staleSlot之外的stale entry，有就执行清理的代码
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

**cleanSomeSlots方法**：寻找stale entry并删除，返回是否有stale entry被删除，该方法只会在两个地方执行

1. replaceStaleEntry方法中，参数i是某个run的终点
2. set方法中，参数i是新entry的位置

不管哪种情况，索引i存储的都不是一个stale entry，方法每次遍历n都会减半，所以在没有找stale entry的情况下只会查找log2(n)次之后结束循环，如果找到stale entry就再查找log2(len)-1次。

``` java
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);
    return removed;
}
```

**rehash方法**：在resize之前执行清除所有stale entry的方法expungeStaleEntries，如果清理之后size仍大于threshold的3/4则resize

``` java
private void rehash() {
    expungeStaleEntries();
    // Use lower threshold for doubling to avoid hysteresis
    if (size >= threshold - threshold / 4)
        resize();
}
```

**expungeStaleEntries方法**：遍历一遍数组table清理所有stale entry，expungeStaleEntry方法同时会对非stale entry重新hash

``` java
private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null)
            expungeStaleEntry(j);
    }
}
```

**resize方法**：新建数组长度*2，遍历旧table将非stale entry放入新数组

``` java
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;
    for (Entry e : oldTab) {
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null; // Help the GC
            } else {
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++;
            }
        }
    }
    setThreshold(newLen);
    size = count;
    table = newTab;
}
```
