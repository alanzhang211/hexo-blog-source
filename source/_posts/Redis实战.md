---
title: Redis实战
date: 2016-12-17 15:59:09
tags: [Redis,缓存]
category: [Redis]
---
继上篇[《Redis入门指南》读书笔记](http://alanzhang.me/2016/11/17/%E3%80%8ARedis%E5%85%A5%E9%97%A8%E6%8C%87%E5%8D%97%E3%80%8B%E8%AF%BB%E4%B9%A6%E7%AC%94%E8%AE%B0/#more)之后，整理了部分Redis相关实战场景。

*(来源于:个人工作经验,网络分享)*

<!--more-->

<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width=330 height=86 src="//music.163.com/outchain/player?type=2&id=441138551&auto=1&height=66"></iframe>

# Jedis
java 的 Redis的客户端开发实现
+ [GitHub](https://github.com/xetorthio/jedis)
+ [官方文档](http://redis.io/documentation)
+ [Redis命令参考简体中文版](http://redis.readthedocs.io/en/2.4/index.html)
+ [maven地址](http://mvnrepository.com/artifact/redis.clients/jedis)

# Redis特性
Redis的优点可以总结为以下几点：

 + 性能极高，Redis能支持超过 100K+ 每秒的读写频率；、
 + 丰富的数据类型，Redis支持二进制案例的 Strings、Lists、Hashes、Sets 及 Ordered Sets 数据类型操作
+ 原子性，Redis的所有操作都是原子性的，同时Redis还支持对几个操作全并后的原子性执行；
+ Redis还支持 Publish/Subscribe、通知、Key过期等等特性。

# 应用场景
## 用户信息管理
用户有很多信息需要管理，如登录信息、注册信息等。传统的方式是采用关系型数据库存储用户信息，定义一张用户表，用户的属性对应表的列，这种方式的可扩展性很差，当用户增加新属性时，需要修改数据库中的用户表、数据订正等操作；采用Redis数据库进行用户信息管理时，通过采用Hashes数据结构。

key使用用户id，用户信息是Hashes内的Field。

## 关注列表
可以使用sets实现这类关注链。

调用sadd方法，添加value。

## 排行
实时更新积分排行榜，每个用户都有自己的积分和姓名信息。通过定义rank这个Key对应每个用户的积分，通过调用zadd增加对应用户的积分值，如 zadd rank 1000 Super；最后通过zrangebyscore zrank -inf +inf 遍历用户的key，得到指定范围内用户的积分，从而得到积分排行榜。

## 最新评论
可以通过Lists实现最新评论。

当用户有评论时，调用Ipush增加用户的评论，如Ipush latest.comment “今天天气很好”；用户也可以调用 Irange latest.comment 0 2 获得最新评论。

**以上信息来源于[Redis实践及在直播行业的应用](https://yq.aliyun.com/articles/62559?spm=5176.8112568.421684.11.bm4YpN)**

---

## session信息
实现集群间的session共享

![](http://of7369y0i.bkt.clouddn.com//redis/session.png)

> session的几个关键点：过期时间，SessionId，一个SessionId里面会存在多组key/value数据。基于这个特性我将采用Hash结构来存储。


## 数据库缓存加速
最容易理解的，Redis作为缓存服务。最初的就是实现数据缓存，用于减轻数据库端的负载。



## 点赞数、评论数目
通过Incr/decr进行数目增加或减少


## Subscribe/Publish订阅模式实现
用于构建即时通信应用，比如网络聊天室(chatroom)和实时广播、实时提醒等
![](http://of7369y0i.bkt.clouddn.com//redis/SubscribePublish.JPG)


## 分布式锁机制
### 案例
目前，从事开发的数据开发平台，应用部署多个节点。涉及到元数据抽取处理。用户手动点击抽取，如果多个客户端并发去点击。就会出数据抽取重复的问题。
### 措施
使用Redis缓存抽取源资源。如：以db为粒度，将数据库id以key/value的形式进行缓存。同时，设置过期时间。

 > 参考资料[谈谈Redis的SETNX](http://huoding.com/2015/09/14/463?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)


## 去重统计
[漫谈redis在运维数据分析中的去重统计方式](https://yq.aliyun.com/articles/57153?from=groupmessage&hmsr=toutiao.io&isappinstalled=0&utm_medium=toutiao.io&utm_source=toutiao.io)
