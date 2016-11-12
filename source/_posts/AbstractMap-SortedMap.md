---
title: AbstractMap&SortedMap
date: 2016-03-20 10:55:26
tags: [2016,AbstractMap,SortedMap]
category: java
---
继上篇介绍java集合第一次层结构，主要介绍了Collection和Map的抽象层。这篇主题介绍Map接口.
![]()

<!--more-->

# AbstractMap
此类提供 Map 接口的骨干实现，以最大限度地减少实现此接口所需的工作。

要实现不可修改的映射，编程人员只需扩展此类并提供 entrySet 方法的实现即可，该方法将返回映射的映射关系 set 视图。通常，返回的 set 将依次在 AbstractSet 上实现。此 set 不支持 add 或 remove 方法，其迭代器也不支持 remove 方法。

要实现可修改的映射，编程人员必须另外重写此类的 put 方法（否则将抛出 UnsupportedOperationException），entrySet().iterator() 返回的迭代器也必须另外实现其 remove 方法。

按照 Map 接口规范中的建议，编程人员通常应该提供一个 void（无参数）构造方法和 map 构造方法。

此类中每个非抽象方法的文档详细描述了其实现。如果要实现的映射允许更有效的实现，则可以重写所有这些方法。

## 源码
```
transient volatile Set<K>        keySet = null;

transient volatile Collection<V> values = null;
```
### transient
1. transient 是表明该数据不参与序列化。因为 HashMap 中的存储数据的数组数据成员中，数组还有很多的空间没有被使用，没有被使用到的空间被序列化没有意义。所以需要手动使用 writeObject() 方法，只序列化实际存储元素的数组。
2. 由于不同的虚拟机对于相同 hashCode 产生的 Code 值可能是不一样的，如果你使用默认的序列化，那么反序列化后，元素的位置和之前的是保持一致的，可是由于 hashCode 的值不一样了，那么定位函数 indexOf（）返回的元素下标就会不同，这样不是我们所想要的结果。

### 引申
#### 序列化

**把对象转换为字节序列的过程称为对象的序列化。
把字节序列恢复为对象的过程称为对象的反序列化。**

**对象的序列化主要有两种用途：**
> 1. 把对象的字节序列永久地保存到硬盘上，通常存放在一个文件中
。2. 在网络上传送对象的字节序列。

>在很多应用中，需要对某些对象进行序列化，让它们离开内存空间，入住物理硬盘，以便长期保存。比如最常见的是Web服务器中的Session对象，当有 10万用户并发访问，就有可能出现10万个Session对象，内存可能吃不消，于是Web容器就会把一些seesion先序列化到硬盘中，等要用了，再把保存在硬盘中的对象还原到内存中。

##### serialVersionUID
serialVersionUID的取值是Java运行时环境根据类的内部细节自动生成的。如果对类的源代码作了修改，再重新编译，新生成的类文件的serialVersionUID的取值有可能也会发生变化。
类的serialVersionUID的默认值完全依赖于Java编译器的实现，对于同一个类，用不同的Java编译器编译，有可能会导致不同的 serialVersionUID，也有可能相同。为了提高serialVersionUID的独立性和确定性，强烈建议在一个可序列化类中显示的定义serialVersionUID，为它赋予明确的值。

**显式地定义serialVersionUID有两种用途：**
>1、 在某些场合，希望类的不同版本对序列化兼容，因此需要确保类的不同版本具有相同的serialVersionUID；
2、 在某些场合，不希望类的不同版本对序列化兼容，因此需要确保类的不同版本具有不同的serialVersionUID。

#### volatile
实现多线程下key和values的共享。

Volatile修饰的成员变量在每次被线程访问时，都强迫从共享内存（主存储）中重读该成员变量的值。而且，当成员变量发生变化时，强迫线程将变化值回写到共享内存。这样在任何时刻，两个不同的线程总是看到某个成员变量的同一个值。

##### containsValue
```
public boolean containsValue(Object value) {

Iterator<Entry<K,V>> i = entrySet().iterator();

if (value==null) {

while (i.hasNext()) {

Entry<K,V> e = i.next();

if (e.getValue()==null)

return true;

}

} else {

while (i.hasNext()) {

Entry<K,V> e = i.next();

if (value.equals(e.getValue()))

return true;

}

}

return false;

}
```
如果value==null，则直接遍历集合，如果元素有为null的直接返回；如果value不为null，则使用equals处理。

**“==”比“equal”运行速度快,因为“==”只是比较引用。**

 **引申**

 ==：基础类型，也称原始类型，

如：byte,short,char,int,long,float,double,Boolean。比较的是值内容。

复合对象，比较的是内存地址。

equals：复合对象继承与Object对象，并对equals方法进行重写。依据不同的重写实现有不同的判断依据。逻辑相对“==”之比较地址要耗时。如：String类先比较地址（效率快）；在不等的情况下，比较内容（先对耗时，有）

Object中的equals中定义

```
public boolean equals(Object obj) {

return (this == obj);

}
```

String中的重写equals

```
public boolean equals(Object anObject) {

if (this == anObject) {

return true;

}

if (anObject instanceof String) {

String anotherString = (String)anObject;

int n = count;

if (n == anotherString.count) {

char v1[] = value;

char v2[] = anotherString.value;

int i = offset;

int j = anotherString.offset;

while (n– != 0) {

if (v1[i++] != v2[j++])

return false;

}

return true;

}

}

return false;

}
```
#### Clone
返回此 AbstractMap 实例的浅拷贝本：不复制键和值本身，如要实现深拷贝，需要重写clone()方法。

```
protected Object clone() throws CloneNotSupportedException {

AbstractMap<K,V> result = (AbstractMap<K,V>)super.clone();

result.keySet = null;

result.values = null;

return result;

}
```
# SortedMap
保证按照键的升序排列的映射，可以按照键的自然顺序（参见 Comparable 接口）进行排序， 或者通过创建有序映射时提供的比较器进行排序。对有序映射的集合视图（由 entrySet、keySet 和 values 方法返回）进行迭代时，此顺序就会反映出来。 要采用此排序，还需要提供一些其他操作（此接口是相似于 SortedSet 接口的映射）。插入有序映射的所有键都必须实现 Comparable 接口（或者被指定的比较器所接受）。

## Comparator
返回与此有序映射关联的比较器，如果使用键的自然顺序，则返回 null。
