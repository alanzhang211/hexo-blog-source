---
title: java集合梳理-概览
date: 2016-03-12 11:32:02
tags: [2016,集合]
category: [java]
---
工作，学习，面试等遇到最多的就是Java集合知识。遂，进行简单梳理（其实，这些工作应该在接触java时就该输出的）。首先，来个集合类的总览效果图。

![](http://of7369y0i.bkt.clouddn.com/2016/03/p11.JPG)

<!--more-->

接下来进行第一层剖析。

# 类图
![](http://of7369y0i.bkt.clouddn.com/2016/03/p2.JPG)

# 组件
## Iterator
1、迭代接口，实现集合的迭代处理。

2、用途：它可以把访问逻辑从不同类型的集合类中抽象出来，从而避免向客户端暴露集合的内部结构。

3、与Enumeration的区别

3.1、Iterator可以删除元素，但是Enumration却不能

3.2、 还有一点要注意的就是，使用Iterator来遍历集合时，应使用Iterator的remove()方法来删除集合中的元素，使用集合的remove()方法将抛出ConncurrentModificationException异常。

>引申1：如何处理ConncurrentModificationException异常（其实就是fail-fast机制）

### 原因
使用了集合自身的remove方法移除元素；只要删除成功modCount++；导致expectedModCount != modCount，抛出ConncurrentModificationException异常。

### 源码分析
expectedModCount和modCount定义与AbstractList中。每当集合被修改一次（结构上面的修改，内部update不算），如add、remove等方法，modCount + 1。都会进行checkForComodification()检测。

```
final void checkForComodification() {

if (modCount != expectedModCount)

throw new ConcurrentModificationException();

}

}
```
### 处理方式
>（内容引自：http://www.2cto.com/kf/201403/286536.html）

#### remove方法
使用Iterator提供的remove方法，用于删除当前元素
```
for (Iterator<string> it = myList.iterator(); it.hasNext();) {

String value = it.next();

if (value.equals( “3”)) {

it.remove();  // ok

}

}

System. out.println( “List Value:” + myList.toString());

```
#### 建一个集合，记录需要删除的元素，之后统一删除
```
List<string> templist = new ArrayList<string>();

for (String value : myList) {

if (value.equals( “3”)) {

templist.remove(value);

}

}

// removeAll其中使用Iterator进行遍历

myList.removeAll(templist);

System. out.println( “List Value:” + myList.toString());
```
#### CopyOnWriteArrayList进行删除操作
**使用线程安全CopyOnWriteArrayList进行删除操作**
```
List<string> myList = new CopyOnWriteArrayList<string>();

myList.add( “1”);

myList.add( “2”);

myList.add( “3”);

myList.add( “4”);

myList.add( “5”);

Iterator<string> it = myList.iterator();

while (it.hasNext()) {

String value = it.next();

if (value.equals( “3”)) {

myList.remove( “4”);

myList.add( “6”);

myList.add( “7”);

}

}

System. out.println( “List Value:” + myList.toString());
```

#### 不使用Iterator进行遍历
**不使用Iterator进行遍历，需要注意的是自己保证索引正常**
```
for ( int i = 0; i < myList.size(); i++) {

String value = myList.get(i);

System. out.println( “List Value:” + value);

if (value.equals( “3”)) {

myList.remove(value);  // ok

   i–; // 因为位置发生改变，所以必须修改i的位置

}

}
```
>备注：removeAll源码
> //抽象类：AbstractCollection实现removeAll方法，使用了iterator的remove()方法处理。此方法没有进行checkForComodification检测。


```
public boolean removeAll(Collection<?> c) {

    boolean modified = false;

    Iterator<?> e = iterator();

    while (e.hasNext()) {

        if (c.contains(e.next())) {

       e.remove();

       modified = true;

        }

    }

    return modified;

}
```
>引申2:多线程下如何处理ConncurrentModificationException

### 处理方式
1. 所有遍历增删地方都加上synchronized或者使用Collections.synchronizedList，虽然能解决问题但是并不推荐，因为增删造成的同步锁可能会阻塞遍历操作

2. 推荐使用ConcurrentHashMap、CopyOnWriteArrayList、CopyOnWriteArraySet。（两个集合实现map、list、Set的线程安全，详细分析见后文.

>注意：1. CopyOnWriteArrayList不能使用Iterator.remove()进行删除。2. CopyOnWriteArrayList使用Iterator且使用List.remove(Object);

# ListIterator
1、继承与Iterator，除了Iterator定义的方法（next()、hasNext()、remove()）外，扩展了适合于List操作元素的方法（

hasPrevious()、previous()：实现逆向（顺序向前）遍历。

nextIndex()、previousIndex()：定位索引位置。ListIterator没有获取当前元素索引的方法。一个长度为 n 的列表的迭代器有 n+1 个可能的指针位置，如下图示：

Element(0)   Element(1)   Element(2)   … Element(n-1)

^           ^           ^             ^               ^

set()、add()：修改、添加元素到列表中。

1. ListIterator在List、ArrayList、LinkedList和Vector使用

2. 与Iterator的对比
> 相同点：
> 都是迭代器，当需要对集合中元素进行遍历不需要干涉其遍历过程时，这两种迭代器都可以使用.  
>不同点:
> 1.使用范围不同，Iterator可以应用于所有的集合，Set、List和Map和这些集合的子类型。
而ListIterator只能用于List及其子类型。
> 2.ListIterator有add方法，可以向List中添加对象，而Iterator不能。
3.ListIterator和Iterator都有hasNext()和next()方法，可以实现顺序向后遍历，
但是ListIterator有hasPrevious()和previous()方法，可以实现逆向（顺序向前）遍历。Iterator不可以。
 5.都可实现删除操作，但是ListIterator可以实现对象的修改，
set()方法可以实现。Iterator仅能遍历，不能修改.

# Iterable
1. 实现这个接口允许对象成为 “foreach” 语句的目标。

2. Iterable的主要作用为：实现Iterable接口来实现适用于foreach遍历的自定义类。

>引申：为什么一定要实现Iterable接口，为什么不直接实现Iterator接口呢？

因为Iterator接口的核心方法next()或者hasNext() 是依赖于迭代器的当前迭代位置的。 如果Collection直接实现Iterator接口，势必导致集合对象中包含当前迭代位置的数据(指针)。 当集合在不同方法间被传递时，由于当前迭代位置不可预置，那么next()方法的结果会变成不可预知。 **除非再为Iterator接口添加一个reset()方法，用来重置当前迭代位置。 但即时这样，Collection也只能同时存在一个当前迭代位置。 而Iterable则不然，每次调用都会返回一个从头开始计数的迭代器。 多个迭代器是互不干扰的。**

# Collection
1. List、Set的父接口

2. 实现中List允许元素重复，Set不允许重复。元素允许为null。

3. 实现Iterable。

4. Collection没有get()方法来取得某个元素。只能通过iterator()遍历元素。

>引申：Collection 和Collections的区别

**java.util.Collection 是一个集合接口。** 它提供了对集合对象进行基本操作的通用接口方法。Collection接口在Java 类库中有很多具体的实现。Collection接口的意义是为各种具体的集合提供了最大化的统一操作方式。

**java.util.Collections 是一个包装类。** 它包含有各种有关集合操作的静态多态方法。此类不能实例化，就像一个工具类，服务于Java的Collection框架。

# Map
1. 存储key-value键值对。其中，map的keys使用Set存储，values使用Collection存储；key-value使用内部Entry实现封装。

2. key不能重复，value允许重复。

3. 要进行迭代，您必须获得一个 Iterator 对象。因此，要迭代 Map 的元素，必须进行比较烦琐的编码。

Iterator keyValuePairs = aMap.entrySet().iterator();

Iterator keys = aMap.keySet().iterator();

Iterator values = aMap.values().iterator();

详细介绍参见：[Java Map 集合类简介](http://www.oracle.com/technetwork/cn/articles/maps1-100947-zhs.html)

待续……
