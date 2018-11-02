---
title: GUAVA初探
date: 2016-07-03 21:05:42
tags: [2016,guava,开源]
category: 开源
---
# 概述
*Guava 中文是石榴的意思，该项目是 Google 的一个开源项目，包含许多 Google 核心的 Java 常用库。
https://github.com/google/guava*

<!--more-->

# 背景
项目中对于数据的操作离不开CRUD基本操作；这四种基本操作中，修改操作相对其他三者考虑的东西要多。
# 举例
有个权限管理系统，负责相关角色的权限管理。如：角色A拥有功能f1和f2，后续进行权限修改，取消f1，授予f3权限。
# 思路
从request重去除角色A最新的权限集合setB = {f1,f3}；然后和之前的比较setA = {f1,f2}；两个集合setA和setB的交集intersectionSet = {f1}就是要删除的；setB相对于setA的差集differenceSet = {f3}就是需要新增的。

![](https://github.com/alanzhang211/blog-image/raw/master/2016/07/guava.JPG)

# 实现
将两个set通关过迭代器进行轮询比较，分别所处差集和并集。这是一个重复造轮子的过程，本文简单使用guava这个google的开源工具集（https://github.com/google/guava，
Guava 中文是石榴的意思，该项目是 Google 的一个开源项目，包含许多 Google 核心的 Java 常用库）与jdk的实现进行对比。

## 代码
```
public class GuavaSet {
public static void jdkSetMethod(Set<Object> setA,Set<Object> setB) {
        System.out.println("======jdkSetMethod==========");
        Iterator<Object> itr = setA.iterator();
        System.out.println("setA:" + setA);
        System.out.println("setB:" + setB);
//交集
Set<Object> intersectionSet= new HashSet<Object>();
while (itr.hasNext()) {
            Object object = itr.next();
if (setB.contains(object)) {
                intersectionSet.add(object);
            }
        }
        System.out.println("intersectionSet:" + intersectionSet);

//差集（setA-intersectionSet）
Set<Object> differenceSet = new HashSet<Object>();
        differenceSet.addAll(setA);
        differenceSet.removeAll(intersectionSet);
        System.out.println("differenceSet:" + differenceSet);

//并集(setA+differenceSet)
Set<Object> unionSet = new HashSet<Object>();
        unionSet.addAll(setA);
        unionSet.addAll(differenceSet);
        System.out.println("union:" + unionSet);
    }

public static void guavaSetMethod(Set<Object> setA,Set<Object> setB) {
        System.out.println("======guavaSetMethod==========");
        System.out.println("setA:" + setA);
        System.out.println("setB:" + setB);

        SetView<Object> unionSet = Sets.union(setA, setB);
        System.out.println("union:"+unionSet);

        SetView<Object> differenceSet = Sets.difference(setA, setB);
        System.out.println("difference:"+differenceSet);

        SetView<Object> intersectionSet = Sets.intersection(setA, setB);
        System.out.println("intersection:"+intersectionSet);
    }

public static void main(String[] args) {
        Set<Object> setA = new HashSet<Object>();
        setA.add(1);
        setA.add(2);
        setA.add(3);
        setA.add(4);

        Set<Object> setB = new HashSet<Object>();
        setB.add(1);
        setB.add(2);
        setB.add(5);

guavaSetMethod(setB, setA);
jdkSetMethod(setB, setA);
    }
}
```
此文简单引入Guava，其他组件及源码层次对比待续。
