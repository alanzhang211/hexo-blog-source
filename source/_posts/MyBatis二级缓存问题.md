---
title: 'MyBatis二级缓存问题'
date: 2017-06-01 22:16:15
tags: [2017,MyBatis]
category: [MyBatis]
---
# 描述
前段时间解决socket异步通信问题后。好好的，无缘无故出现诡异问题。socket推送结果状态后，前端页面进行结果展示。发现获取的结果并不是最新的。


# 分析
## 分析
交互过程参见[SocketIO解决504错误](http://alanzhang.me/2017/04/10/SocketIO%E8%A7%A3%E5%86%B3504%E9%94%99%E8%AF%AF/#more)。


然后看看后台日志和浏览器的请求过程,发送多次的请求。

![浏览器请求](http://of7369y0i.bkt.clouddn.com//2017/06/q1.JPG)

后台日志
![后台日志查看查询](http://of7369y0i.bkt.clouddn.com//2017/06/log.png)

<!--more-->

socket中间推送的一些中间结果，前端触发的查询没有落到数据库。这让人想到是不是缓存的问题。于是，网上搜了搜。果真是Mybatis的配置了二级缓存导致的。

![mybatis配置](http://of7369y0i.bkt.clouddn.com//2017/06/mybatis%E9%85%8D%E7%BD%AE.png)

## 二级缓存
Mybatis二级缓存是多个SqlSession共享的，其作用域是mapper的同一个namespace，不同的sqlSession两次执行相同namespace下的sql语句且向sql中传递参数也相同即最终执行相同的sql语句，第一次执行完毕会将数据库中查询的数据写到缓存（内存），第二次会从缓存中获取数据将不再从数据库查询，从而提高查询效率。Mybatis默认没有开启二级缓存需要在setting全局参数中配置开启二级缓存。

再看，项目中的mybatis的xml文件。对于基础的save、update、delete操作存放在mybatis.mapper.meta.gen文件夹下。而扩展的复杂业务存放在mybatis.mapper.meta目录下。

![mybatis](http://of7369y0i.bkt.clouddn.com//2017/06/xml.JPG)

这将导致两个包下的select操作会共用二级缓存。所以，socket不断的去update的时候（mybatis.mapper.meta.gen包下面的），之前的二级缓存依旧有效。导致，前端页面的查询没有落到db层，而是从缓存中获取。看到的现象是就前端展示的结果没有更新。

## 处理方式
将mybatis的二级缓存配置禁用。
```
<setting name="cacheEnabled" value="false" />
```

网上讨论禁用二级缓存的话题，以及如何规避。这里不做描述。有兴趣的同学可以参考文章结尾的两篇。

## 再谈mybatis问题
或许有人会问，项目中为什么会出现两份类似的mybatis的xml配置，为什么不写在一起？这样就不会有之前的问题(同一个mapper使用同一缓存)。

这里额外说明一下：这里的分包处理的原因，来源于mybatis的生成插件。“gen”命名就是这么来的。为了每次修改数据库表，一键生成相关实体、mapper、xml文件，及相关基础的CRUD操作。如果有复杂业务需要，则扩展mapper配置。这样不会影响原始的mapper文件。

至此，带来了便利的同时也失去了一些。**世界万物，有得就有失；在恰当的时机选择合适的方式。**

---
*参考资料*
+ http://www.cnblogs.com/xujian2014/p/5478476.html
+ http://www.open-open.com/lib/view/open1477809986747.html

---
版权所有，欢迎转载，转载请标明出处！
