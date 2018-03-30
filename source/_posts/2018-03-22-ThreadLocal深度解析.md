---
title: ThreadLocal深度解析
date: 2018-03-22 10:20:26
tags:
---
### ThreadLocal解决什么问题

`ThreadLocal`不是用来解决共享对象的多线程访问问题的，不同的`Thread`通过`ThreadLocal`获取到的是不同的副本（实际是不同的实例）。线程内部的副本是其他线程不需要访问也是访问不到的。当某一个类不是线程安全，同时该类的实例需要在多个方法中被使用且每个线程需要自己独立的实例时，可以使用`ThreadLocal`来解决这一问题。

### ThreadLocal实现原理

#### 错误方式

既然每个Thread通过`ThreadLocal.get()`获得的都是自己的一个副本，一个通常的实现方法是`ThreadLocal`自己维护一个Map，key是Thread，value是该线程中的副本。线程通过`get()`获取实例时，可以以线程（通过`Thread.currentThread()获取当前线程`）为key，从Map中找出对应的实例即可。但是该实现方式存在以下问题：

- Map作为全局变量，增加或减少线程均需要写Map，需要通过加锁来确保Map的线程安全性
- 线程结束时，需要从该线程访问过的所有ThreadLocal中删除副本，否则可能会引起内存泄露

#### 正确方式

为了提高ThreadLocal效率，需要去掉锁机制。如果Thread维护一个自己的Map，该Map存储所有用到的本地副本，这样每个Thread只需要访问自己的Map，不存在写冲突，也就不需要锁了。

![184951-13b6596868c2dac](F:\NoteBook\picture\2184951-13b6596868c2dacd.png)

该方案虽然没有锁的问题，但是在每个线程内部都保存了该线程用到的本地副本。如果不删除这些引用，会导致这些副本无法被垃圾回收，造成内存泄露。

### ThreadLocal在JDK8中的实现

### ThreadLocalMap类

ThreadLocalMap是ThreadLocal的静态内部类，对ThreadLocal进行的get、set操作最后都将委托给该类。同时每个Thread内部都有一个ThreadLocalMap变量，用于保存本线程用到的副本。它的结构如下：

![018032116251](F:\NoteBook\picture\20180321162514.png)

可以看到ThreadLocalMap有一个常量和三个成员变量：

```java
		/**
         * The initial capacity -- MUST be a power of two.
         */
        private static final int INITIAL_CAPACITY = 16;

        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         */
        private Entry[] table;

        /**
         * The number of entries in the table.
         */
        private int size = 0;

        /**
         * The next size value at which to resize.
         */
        private int threshold; // Default to 0
```

其中INITIAL_CAPACITY代表了这个Map的初始容量，即table的初始大小；table是一个Entry类型的数组，用于存储数据；size代表table中的存储数目；threshold代表需要扩容是对应size的阀值，默认为容量的2/3。

#### Entry类

Entry类是ThreadLocalMap的静态内部类，是线程本地副本真正保存的位置，源码如下：

```java
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

可见Entry继承自`WeakReference<ThreadLocal<?>>`，即每个Entry对象都有一个ThreadLocal的弱引用。这样当线程结束，没有引用指向线程内部的ThreadLocalMap变量时，table数组可以被垃圾回收，如此便可以防止内存泄露。Object类型的成员变量value，指向Thread用到的ThreadLocal中的本地副本。

#### ThreadLocal.set()方法

设置实例的方法如下所示：

```java
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

线程首先通过`getMap(thread)`方法获取自身的ThreadLocalMap。因为ThreadLocalMap是线程私有的，只有该线程才能访问，其他线程访问不到，所有不用考虑线程安全问题。

```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

从getMap源码中可见，该ThreadLocalMap的实例是Thread类的一个字段，即由ThreadLocal对象与具体实例的映射，这一点与上文分析一致。

获取到ThreadLocalMap后，若map为null，调用`createMap`方法来创建一个ThreadLocalMap，在createMap中调用ThreadLocalMap的构造函数，返回设置了首元素的ThreadLocalMap。

```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```

`firstKey`为当前调用的ThreadLocal，firstValue为`set(T value)`中的value。在构造函数中注意一个细节，计算下标i的时候采用了`hashcode & (size - 1)`的算法，这相当于取模运算`hashCode % size`的一个更高效的实现。同时因为采用了该算法，size必须是2的指数，这可以使得hash发生冲突的次数减小。

若map不为null，则调用ThreadLocalMap的`set(ThreadLocal, Ojbect)`方法将实例设置到table数组中。源码如下：

```java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

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

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

首先计算该元素的hash值，如果冲突了，通过`nextIndex`方法再次计算hash值。`nextIndex`实际上获取数组的下一个下标，若已到数组最后一个元素则返回0，即返回到数组第一个元素。因此ThreadLocalMap解决冲突的方法是线性探测法。如果table中已经存在，则仅仅是更新Entry中的value值。如果entry里对应的key为null的话，表明该entry为`staled entry`，则调用`replaceStaleEntry`函数用于替换原有的key和value并进行一些清理操作(清理其他key为null的Entry防止内存泄露)，有兴趣的同学可以查看源码具体实现。若是经历了上面步骤没有命中hash，也没有发现无用的Entry，`set`方法就会创建一个新的Entry，并会进行**启发式的垃圾清理**，用于清理无用的Entry。主要通过`cleanSomeSlots`方法惊醒清理（清理时机通常为添加新元素或另一个无用的元素被回收时）。只要没有清理任何的stale entries并且size达到阀值的时候，就会触发`rehash`：

```java
private void rehash() {
    expungeStaleEntries();

    // Use lower threshold for doubling to avoid hysteresis
    if (size >= threshold - threshold / 4)
        resize();
}
```

`rehash`会先调用`expungeStaleEntries`函数，执行一次全表扫描，用于删除无用的entry。清理之后的size仍大于等于threshold的3/4时进行`resize`扩容（长度增加一倍）。通过`replaceStaleEntry`和`rehash`这两个方法会即时将table中无效的entry设置为null，从而使得entry可被回收，有效的防止了内存泄露。

#### ThreadLocal.get()方法

获取实例的方法如下所示

```java
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

同`set`方法，首先通过getMap方法获取线程内部的ThreadLocalMap对象。若map和entry都不为null，说明线程内部存在对应副本，直接返回即可。若不存在，调用`setInitialValue`方法获取该ThreadLocal变量在该线程中对应的具体实例的初始值。

```java
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
```

该方法为private方法，无法被重载。但是在方法中首先调用了`initialValue`方法来获取初始值。

```java
protected T initialValue() {
    return null;
}
```

`initialValue`方法是protected的，说明是用来被继承的。所以在使用ThreadLocal时通常会重载该方法。拿到该线程对应的ThreadLocalMap对象，如该对象不为null，则直接将该ThreadLocal对象与对应实例初始值的映射set到该map中，否则创建该map并将其添加其中。

若map存在，`get`操作最终会调用ThreadLocalMap的`getEntry`方法：

```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}

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

hash以后如果是ThreadLocal对应的Entry就返回，否则调用`getEntryAfterMiss`方法，根据线性探测法继续查找，直到找到或对应entry为`null`，并返回。

### ThreadLocal.remove()方法

删除实例的方法如下所示

```java
public void remove(){
    ThreadLOcalMap m = getMap(Thread.currentThread());
    if (m != null){
        m.remove(this);
    }
}
```

同理，`remove`方法也是先获取Thread中的ThreadLocalMap实例。若map不为null，这调用map的`remove`方法，从table中将entry删除。

### 适用场景

ThreadLocal适用如下场景

- 每个线程需要有自己单独的实例
- 实例需要在多个方法中共享

### 应用案例

在Web中，通常使用Session在各个页面中传递信息。每个客户端对应于Web中的一个线程，需要保证线程有自己单独的Session实例；而线程内部的各方法又需要共享Session。如不使用ThreadLocal，其实现方式如下：

```java
public class Web{
    public static class Session{
        private string username;
        private int age;
        ....
        // get/set方法
    }
    public Session createSession(){
        return new Session();
    }
    public string getUsername(Session session){
        return session.getUsername();
    }
    public int getAge(Session session){
        return session.getAge();
    }
}
```



### 总结

Entry会被清理的场景：

- Thread结束之后被垃圾回收处理
- set一个变量时，发现staled entry。进行替换并清理
- set一个变量时，`size`大于阀值时，调用`rehash`方法清理并扩容
- 调用`remove`方法时，删除该entry并清理发现的staled entry

尽管采用了弱引用的ThreadLocalMap不会造成内存泄露，但是并不能保证staled entry被及时清理。因此我们在使用完ThreadLocal后最好还是`remove`一下（使用线程池时一定要即时`remove`，线程使用后归还给了线程池，并没有销毁），保证即时回收无用的Entry。