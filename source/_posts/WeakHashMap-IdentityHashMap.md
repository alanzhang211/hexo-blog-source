---
title: WeakHashMap&IdentityHashMap
date: 2016-03-20 21:33:40
tags: [java,map,WeakHashMap,IdentityHashMap]
category: java
description: 简述WeakHashMap和IdentityHashMap
---
# WeakHashMap
WeakHashMap实现Map的弱引用；在内存不足时，jvm优先处理弱引用所占据的堆内存空间

## 定义
public class WeakHashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>

继承AbstractMap，实现Map接口，是HashMap的一种实现。使用弱引用作为内部数据的存储方案，WeakHashMap可以作为简单缓存表的解决方案，当系统内存不够的时候，垃圾收集器会自动的清除没有在其他任何地方被引用的键值对。


WeakHashMap依赖于java.lang.ref包。

## 成员变量
private final ReferenceQueue<Object> queue = new ReferenceQueue<>();

引用队列ReferenceQueue：

一旦弱引用返回null值，那么其指向的对象（即Widget）就变成了垃圾，这个弱引用对象(即weakWidget)也就没有用了。这通常意味着要进行一定方式的清理（cleanup）。例如，WeakHashmap将会移除一些死的（dread）的entry，避免持有过多死的弱引用。

ReferenceQuene能够轻易的追踪这些死掉的弱引用。可以讲ReferenceQuene传入WeakHashmap的构造方法（constructor）中，这样，一旦这个弱引用指向的对象成为垃圾，这个弱引用将加入ReferenceQuene中。

## 内部类
### Entry
private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V>

继承弱引用WeakReference，添加元素到集合中时，使用弱引用处理对象。使得对象在内存不足时，进行回收。

源码：
```
Entry(Object key, V value,

ReferenceQueue<Object> queue,

int hash, Entry<K,V> next) {

super(key, queue);

this.value = value;

this.hash  = hash;

this.next  = next;

}
```
将Entry的key指向引用队列，key值为弱引用。如果一个WeakHashmap的key变成垃圾，那么它对应用的value也自动的被移除。

## 内部方法
expungeStaleEntries这个函数的来实现移除其内部不用的条目从而达到的自动释放内存的目的的.基本上只要对WeakHashMap的内容进行访问就会调用这个函数，从而达到清除其内部不在为外部引用的条目。
```
private void expungeStaleEntries() {

//遍历key引用队列，queue存储的是key。返回需要回收的key对象。

for (Object x; (x = queue.poll()) != null; ) {

synchronized (queue) {

@SuppressWarnings(“unchecked”)

Entry<K,V> e = (Entry<K,V>) x;

int i = indexFor(e.hash, table.length);

Entry<K,V> prev = table[i];

Entry<K,V> p = prev;

while (p != null) {

Entry<K,V> next = p.next;

if (p == e) {

if (prev == e)

table[i] = next;

else

prev.next = next;

//value置为null，便于GC

e.value = null;

size–;

break;

}

prev = p;

p = next;
}
}
}
}
```
# IdentityHashMap
## 定义
public class IdentityHashMap<K,V>extends AbstractMap<K,V> implements Map<K,V>, java.io.Serializable, Cloneable

继承AbstractMap，实现Map接口。比较键（和值）时使用引用相等性代替对象相等性。换句话说，在 IdentityHashMap 中，当且仅当 (k1==k2) 时，才认为两个键 k1 和 k2 相等（在正常 Map 实现（如 HashMap）中，当且仅当满足下列条件时才认为两个键 k1 和 k2 相等：(k1==null ? k2==null : e1.equals(e2))）。

此类的典型用法是拓扑保留对象图形转换，如序列化或深层复制。要执行这样的转换，程序必须维护用于跟踪所有已处理对象引用的“节点表”。节点表一定不等于不同对象，即使它们偶然相等也如此。

此类的另一种典型用法是维护代理对象。例如，调试设施可能希望为正在调试程序中的每个对象维护代理对象。

此类提供所有的可选映射操作，并且允许 null 值和 null 键。此类对映射的顺序不提供任何保证；特别是不保证顺序随时间的推移保持不变。
## 构造函数
`public IdentityHashMap() {

// DEFAULT_CAPACITY = 32

init(DEFAULT_CAPACITY);

}

private void init(int initCapacity) {

threshold = (initCapacity * 2)/3;

table = new Object[2 * initCapacity];

}`

** 此类用的比较少，一般工作中很少用到。可以理解为key可重复（key的应用相等）的map **

`IdentityHashMap<String,Object> map = new IdentityHashMap<String,Object>();
map.put(new String(“张三”), “first”);
map.put(new String(“张三”), “second”);
`

map都会进行存放。
