---
title: LINKEDHASHMAP
date: 2016-03-20 10:42:10
tags: [2016,java,LinkedHashMap]
category: java
---
*LinkedHashMap基于HashMap实现了双向链表的处理.*

# 定义
```
public class LinkedHashMap<K,V>

extends HashMap<K,V>

implements Map<K,V>
```
继承于HashMap，底层使用哈希表与双向链表来保存所有元素。其基本操作与父类HashMap相似，它通过重写父类相关的方法，来实现自己的链接列表特性（维护一个运行于所有条目的双重链接列表。此链接列表定义了迭代顺序，该迭代顺序通常就是将键插入到映射中的顺序（插入顺序）。））

<!--more-->

# 核心方法介绍
## init
```
void init() {

header = new Entry<>(-1, null, null, null);

//双向指针处理

header.before = header.after = header;

}
```
## 构造方法
```
public LinkedHashMap(int initialCapacity, float loadFactor) {

super(initialCapacity, loadFactor);

accessOrder = false;

}
```
accessOrder：是LinkedHashMap的排序模式；true表示按照访问顺序迭代，false时表示按照插入顺序 。一般情况下，不必指定排序模式，其迭代顺序即为默认为插入顺序。

如果想按照从近期访问最少到近期访问最多的顺序（即访问顺序）来保存元素，那么请使用下面的构造方法构造LinkedHashMap：

```
public LinkedHashMap(int initialCapacity,float loadFactor,boolean accessOrder) {

super(initialCapacity, loadFactor);

this.accessOrder = accessOrder;

}
```
这种迭代访问顺序，适合构建LRU（Least Recently Used）缓存。LinkedHashMap在添加方法中使用了removeEldestEntry方法：

```
void addEntry(int hash, K key, V value, int bucketIndex) {

super.addEntry(hash, key, value, bucketIndex);

// Remove eldest entry if instructed

Entry<K,V> eldest = header.after;

if (removeEldestEntry(eldest)) {

removeEntryForKey(eldest.key);

}

}
```
默认返回false，即不进行近期访问最少处理元素，进行删除处理。

```
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {

return false;

}
```
如果想实现LRU缓存，则需要重写removeEldestEntry方法，如维持大小我为100的集合：

```
private static final int MAX_ENTRIES = 100;

protected boolean removeEldestEntry(Map.Entry eldest) {

return size() > MAX_ENTRIES;

}
```
超过100，则继续删除最早的元素。
