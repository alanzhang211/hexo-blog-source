---
title: 《Redis入门指南》读书笔记
date: 2016-11-17 17:06:04
tags: [2016,Redis]
category: [Redis]
---
![Redis入门指南](http://of7369y0i.bkt.clouddn.com//book/2016/redis%E5%85%A5%E9%97%A8%E6%8C%87%E5%8D%97/%E3%80%8ARedis%E5%85%A5%E9%97%A8%E6%8C%87%E5%8D%97%E3%80%8B.jpg)

+ [百度网盘资源下载](http://pan.baidu.com/s/1eR96S6Y)

<!--more-->

# 简介
+ [官方网站](http://redis.io/)
+ [官方下载](http://redis.io/download) 可以根据需要下载不同版本。
[windows版](https://github.com/mythz/redis-windows)
+ [可视化工具RedisClient](http://www.oschina.net/news/55634/redisclient-2-0)

## 特性
### 存储结构
+ remote dictionary server的简称。字典结构存储数据。
+ 通过tcp协议获取内容
+ 支持键值的数据类型
>1. 字符串
>2. 散列类型
>3. 列表类型
>4. 集合类型
>5. 有序集合类型

+ 对不同的数据类型提供了方便的操作。

### 内存存储与持久化
将存储在内存中的数据持久化到硬盘中

### 功能
**缓存，队列系统。**
#### 缓存
+ 设置键的生存时间，过期的会自动删除。此特性实现redis缓存效果。（另一个缓存系统[Memcached](http://memcached.org/)）
> 性能上:redis是单线程模式；Memcached支持多线程。

+ 作为缓存，设置最大内存空间，超过空间后会按照一定的规则淘汰不需要的键。

#### 实现队列
+ redis的列表类型实现优先级队列；支持阻塞读取。
如：[redis实现任务队列](http://blog.csdn.net/zuoanyinxiang/article/details/50263945)
+ 支持“发布/订阅”消息模式。如：[发布/订阅](http://outofmemory.cn/code-snippet/3866/redis-dingyue-publish-example)

### 简单稳定

## 准备
### 安装
（待完善...）

## 启动停止

![](http://of7369y0i.bkt.clouddn.com//book/2016/redis%E5%85%A5%E9%97%A8%E6%8C%87%E5%8D%97/2.2.JPG)

### 启动
#### 命令行
+ 直接启动，默认端口6379
```
redis-server
```
+ ，自定义端口启动
```
redis-server -- port 6380
```

#### 批处理
##### windows
+ windows下新建startup.bat
```
redis-server  redis.windows.conf
```

+ linux
*待续...*

### 停止
+ 命令行
```
redis-cli shutdown
```
>强行停止会导致数据丢失。

## Redis命令行
#### 发送命令
通过redis-cli 向Redis发送命令有两种方式
+ 将命令作为redis-cli的参数
>ping:测试客户端与redis的连接是否正常，连接正常返回pong
+ 不带参数运行redis-cli，进入交互模式。

#### 命令返回值
1. 状态恢复
2. 错误回复：回复以(error)开头
3. 整数回复
4. 字符串回复
5. 多行字符回复

## 配置
+ 通过配置文件进行参数配置
```
redis-server /path/to/redis.conf
```
 + 通过启动参数传递同名的配置选项覆盖文件中相应的参数
```
redis-server /path/to/redis.conf --loglevel warning
```

+ 使用config set命令在不重启redis的情况下动态修改redis配置。
```
config set loglevel warning
```
> 并非所有，命令都支持config set

+ 使用config get 获取配置信息

```
config get loglevel
```

## 多数据库
为每个词典建立独立的数据库。

+ 每个数据库都是以一个0开始递增的数字命名。
+ Redis默认支持16个数据库，可以通过配置参数database修改。
+ 客户端默认选择0号数据库进行连接，可以使用select命令更换。
+ Redis不支持自定义数据库名称
+ Redis不支持为每个数据库设置不同的访问密码。
+ 多个数据库并不是完全隔离的，使用flushall命令清空Redis所有的数据库数据。
+ 建议不同的应用使用不同的Redis实例而非同一个Reids的不同数据库。

# 入门
## 字符串类型
### 命令
#### 赋值/取值
```
set key value
get key
```

#### 递增数字
```
incr key
```
使整型的key的value递增

> 当操作的key不存在时，会默认值为0，每次递增1.

+ Redis命令都是原子的

## 散列类型
+ 字段值只能是字符串
+ 不能嵌套其他数据类型
+ 散列类型适合存储对象
+ 数据是以二维表的形式存储的
+ 可以自由的为任何key增加字段

### 命令
#### 赋值/取值
```
hset key field vaule
hget key field
```
+ hset 插入数据返回1；更新返回0

##### 设置多个值
```
hmset key field1 vaule1 field2 vaule2
```
##### 获取多个值
```
hmget key field1 field2
```


##### 获取整个对象属性
```
hgetall key
```

#### 判断字段是否存在
```
hexists key field
```
存在返回1；否则返回0（key不存在返回0）。


#### 当字段不存在时赋值
+ hsetnx不支执行任何操作

#### 增加数字
```
hincrby key field increment
```
使字段值增加指定的整数；如果不存在会默认增加指定field字段。

#### 删除字段
```
hdel key field [field ...]
```
返回值为被删除的字段个数

## 列表类型
+ Redis借助列表可以作为队列

### 命令
#### 向列表两端增加元素
```
lpush key value[value...]
rpush key value[value...]
```

#### 从列表两端弹出元素
```
lpop key
rpop key
```
#### 获取类表中的元素个数
```
llen key
```

> llen的时间复杂度是O(1);使用InnoDB存储引擎的mysql会遍历整个数据表赶回条目信息。

#### 获的列表片段
```
lrange key start stop
```
+ 不会删除片段
+ 支持负索引

#### 删除列表指定的值
```
lren key count value
```
+ 删除列表前count个数值的value的元素。返回值是实际删除的个数。


## 集合类型
### 命令
#### 增加/删除元素
```
sadd key member [member...]
srem key member [member...]
```
#### 获取集合中的元素
```
smembers key
```
#### 判断元素是否在集合中
```
sismember key member
```
+ 判断一个元素是否在集合中时间复杂度O(1)

#### 集合间运算
```
sdiff key [key...]
sinter key [key...]
sunion key [key...]
```
## 有序集合
每个元素都关联一个分数。
### 命令
#### 增加元素
```
zadd key score member [score member ...]
```
#### 获取元素的分数
```
zscore key member
```

#### 获取排名在指定范围的元素
```
zrange key start stop [withscore]
```
+ 时间复杂度O(logn + m)，其中n为有序集合的基数，m为返回元素的个数。
+ 分数相同，Redis按照字典顺序排序

# 进阶
## 事务

### 错误处理
```
multi
```
+ Redis不支持事务回滚

### watch命令
监控的一个或多个key，一但其中一个key被修改或删除，之后的事务将不会执行。监控一致执行到exec

+ 不想执行事务中的命令可以使用unwatch取消监控。

## 生存时间
### 命令
```
expire key seconds
```
单位:秒。

+ expire单位是秒，pexpire是毫秒

### 实现访问频率限制
eg：限制每个ip每分钟访问次数
+ 增加事务处理
+ 使用列表记录每次的访问时间，访问次数超过限制次数时，将列表中 时间差=最晚时间-最早时间
如果时间差<1min,表示超过限制；否则将当前时间加入，同时删除最早时间。

### 实现缓存
+ 服务器内存有限，缓存时间过长会导致Redis内存会占满。
+ 如果缓存时间过段，命中率过低，起不到缓存的效果。
>解决方案：限制最大缓存；使用一定的规则进行淘汰。（Redis只作为缓存的时候使用。）

## 排序
### 有序结合
+ 常见的场景是大数据排序

### sort命令
+ 可以对列表、集合、有序集合的key进行排序

### by参数
依据by参数进行排序

### store参数
+ 使用store，结果保存在sort.result中。

### 性能优化
+ 减少键排序中的元素个数
+ 使用limit限制返回个数
+ 如果排序数量较大，使用store进行缓存

## 消息通知
### 任务队列
+ 松耦合
+ 易扩展消费者

### Redis实现队列
+ 列表实现队列
+ BRPOP 命令实现阻塞模式

### 优先队列
```
 BRPOP key [key...] timeout
```

### 发布/订阅
#### 发布消息
```
publish channel message
```
+ 返回值表示接收到这条消息的订阅者数量
+ 发不出去的消息不会被持久化

#### 同时订阅多个通道
```
subscribe channel [channel...]
```
+ subscribe：表示订阅成功返回信息，第二个值是通道的名称，第三个值是当前客户订阅的通道数量
+ message：表示接收到的消息。第二个值表示通道名称，第三个值表示消息内容
+ unsubscribe：表示成功取消订阅某个通道；第二个值表示通道名称，第三个值表示当天订阅的通道数量，此值为0表示客户端退出订阅状态。

![](http://of7369y0i.bkt.clouddn.com//book/2016/redis%E5%85%A5%E9%97%A8%E6%8C%87%E5%8D%97/1.JPG)

## 管道
+ 通道减少了客户端和Redis的通信次数
+ Redis提供了对通道的支持；可以一次性返回结果

## 节省空间
### 精简键名和键值
### 内部编码优化
+ 查看键的编码
```
object encoding key
```

## 实践
（略）
