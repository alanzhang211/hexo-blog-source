---
title: Redis实战问题分析
date: 2017-01-19 00:09:31
tags: [2017,Redis,共享锁]
category: Redis
---
# 描述
## 需求描述
数据库中一个表的某个字段（非主键），需要实现递增的效果。

## 实现方式
### 数据库自增机制
使用数据库auto increment机制，实现自增效果。

**问题：由于数据库不允许非主键的自增处理，排除此方案。**

<!--more-->

### 先查后写
先查询数据库，获取最大值。然后再在此基础上进行+1操作。

**问题：非原子操作，可能导致脏读**

### Redis共享锁
数据库共享资源问题。在多线程并发处理下，对数据库进行更新操作。避免出现脏读问题，使用内存数据库Redis的INCRBY实现值递增的原子性操作，解决高并发下共享数据问题。

**Redis特点：不支持异步处理，内部是串行执行每一步指令.**

## 实施
### 流程图
![](http://of7369y0i.bkt.clouddn.com//2017/01/Redisthread.JPG)


### 说明
1. main线程获取最新数据库字段值，并缓存。
2. 启用多个线程（业务抽取线程组，实现多个元数据库的抽取过程）。
3. 使用Redis的INCRBY实现增量处理。（内部需要同步抽取的数据库信息，涉及到之前提到的字段自增。）
4. 异步插入同步过来的数据库信息（其中，自增字段已有Redis处理）。

### 伪代码

```
//主线程缓存最新dbId
Integer maxDbId = extractManager.getMaxDBId();
redisCacheRepository.set(MetaConstant.DB_ID_REDIS_KEY,maxDbId.toString());
```

```
//线程并行处理
tableExtractorCallable = new TableExtractorCallable<Integer>(tableExtractor, datasourceDo, schemaInfo,extractManager,redisCacheRepository);
Future futher = taskExecutor.submit(tableExtractorCallable);
futures.add(futher);
```

```
//redis incrBy递增1
Long dbId = redisCacheRepository.incrBy(MetaConstant.DB_ID_REDIS_KEY,1);
databaseInfoDo.setDbId(Short.valueOf(dbId.toString()));
```

## 问题
> 程序处理中出现报错 **“redis.clients.jedis.exceptions.JedisDataException: ERR value is not an integer or out of range”**

### 分析
redisTemplate配置
```
<bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate">
    <property name="connectionFactory" ref="redisConnectionFactory"/>
</bean>
```


由于使用ValueOperations的set方法为点击量设置了初始值，Key的序列化器默认采用的是JdkSerializationRedisSerializer，其将初始值变成了序列化字符串存入了Redis，而Redis执行INCRBY命令时是无法识别序列化字符串为整数的。从而导致以上错误。

### 措施
修改value的序列化机制，改为StringRedisSerializer。

```
<bean id="redisTemplate" class="org.springframework.data.redis.core.RedisTemplate">
    <property name="connectionFactory" ref="redisConnectionFactory"/>
        <property name="valueSerializer">
            <bean class="org.springframework.data.redis.serializer.StringRedisSerializer" />
    </property>
</bean>
```

## 结语

<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width=330 height=86 src="//music.163.com/outchain/player?type=2&id=451126287&auto=1&height=66"></iframe>

**“道路漫漫，がんばって！”**
