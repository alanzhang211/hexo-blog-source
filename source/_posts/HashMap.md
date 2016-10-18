---
title: HashMap
date: 2016-03-20 22:09:57
tags: [java,map,HashMap]
category: java
description: HashMap是生产中使用频率很大高的集合类，多用于缓存数据处理，维护key-value的键值对。
---
# HashMap
## 定义
```
public class HashMap<K,V>  extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable
```
基于哈希表的 Map 接口的实现，以key-value的形式存在。在HashMap中，key-value总是会当做一个整体（Entry<k,v>对象）来处理，系统会根据hash算法来来计算key-value的存储位置，我们总是可以通过key快速地存、取value。是一种支持快速存取的数据结构。


默认大小是16，支持自定负载因子（默认0.75）。当当前容量大于initialCapacity*loadFactor时，map就进行扩容处理，大小扩展到原来的2倍。

### 源码
```
void addEntry(int hash, K key, V value, int bucketIndex) {

if ((size >= threshold) && (null != table[bucketIndex])) {

resize(2 * table.length);

hash = (null != key) ? hash(key) : 0;

 bucketIndex = indexFor(hash, table.length);

}

createEntry(hash, key, value, bucketIndex);

}
```
其中threshold是map容纳的数量。

**threshold = (int)Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);**
### 结构图
![结构图](http://of7369y0i.bkt.clouddn.com/HashMap1.jpg)

**HashMap底层实现还是数组，只是数组的每一项都是一条链（元素节点为Entry）。其中参数initialCapacity就代表了该数组的长度。**

## 三个视图

map的keys使用Set存储，values使用Collection存储； key-value使用内部Entry实现封装，使用Set存储。

## 内部类
### Entry
```
static class Entry<K,V> implements Map.Entry<K,V>
```
实现Map接口，维护一个key，value的链表结构。也就是每个key数组对应的table链表对象。

### HashIterator
```
private abstract class HashIterator<E> implements Iterator<E> {

Entry<K,V> next;        // next entry to return

int expectedModCount;   // For fast-fail

int index;              // current slot

Entry<K,V> current;

}
```
实现map内部的迭代器处理。同时，支持fast-fail机制。（Colllection是在AbstractList定义了expectedModCount）

### ValueIterator
实现value的迭代器。

如：Iterator values = aMap.values().iterator();

### KeyIterator
实现key的迭代器

如：Iterator keys = aMap.keySet().iterator();

### put(K key, V value)
```
public V put(K key, V value) {

if (key == null)

return putForNullKey(value);

int hash = hash(key);

int i = indexFor(hash, table.length);

for (Entry<K,V> e = table[i]; e != null; e = e.next) {

Object k;

if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {

V oldValue = e.value;

e.value = value;

e.recordAccess(this);

return oldValue;

}

}

modCount++;

addEntry(hash, key, value, i);

return null;

}
```
1. 如果key为null，则放在数组第一位置。

2. 如果key不为null，则通过计算key的hash值找到数据索引；如果对应的table内有元素（hash碰撞），则计算key是否相等（即通过equals比较）。如果相等，则替换老元素，并返回老元素；如果不相等，怎进行addEntry处理，将新元素添加到table的链首。

**计算元key的索引**
```
int i = indexFor(hash, table.length)
```

```
static int indexFor(int h, int length) {

return h & (length-1);

}
```
h&(length – 1)就相当于对length取模，而且速度比直接取模快得多，这是HashMap在速度上的一个优化。作用：均匀分布table数据和充分利用空间。

```
void createEntry(int hash, K key, V value, int bucketIndex) {

Entry<K,V> e = table[bucketIndex];

table[bucketIndex] = new Entry<>(hash, key, value, e);

size++;

}
```
获取bucketIndex元素e，将新创建的Entry指向老元素e。


*参考*

1. http://blog.csdn.net/chenssy/article/details/18323767

2. http://www.oracle.com/technetwork/cn/articles/maps1-100947-zhs.html
