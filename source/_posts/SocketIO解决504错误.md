---
title: 'SocketIO解决504错误'
date: 2017-04-10 21:44:49
tags: [2017,架构设计]
category: 软件设计
---
![](http://of7369y0i.bkt.clouddn.com//2017/04/%E6%9E%B6%E6%9E%84%E8%AE%BE%E8%AE%A1/galaxy.jpg)
# 问题描述
[即席查询](http://baike.baidu.com/link?url=Hz5i4YUx3rO2whqAOcMmxtv4tikbBLlb_zbxjR8jEfYrLijJ4ZSZgwP-dbRe0jDBmjiITRe4QHxLz4mBfz2nNIHJEB4zFl_rMbA41q4X4_j4is_kcz6S3N-bzoB8iw1y)需要在前台展示出查询的结果集。由于sql语句的复杂度以及hive等计算引擎。无法在给定的时间内查询出结果集（超出连接超时时间），导致前端页面在未收到返回之前断开连接**出现[504错误](http://baike.baidu.com/link?url=g3QpkgRxr0OJkkLbVd-xmhltCjCefO75ma9GPLUQOSTnyDFJ5dGB9tw2XIXz7TRt_soCO9xhS-OUkfNg2m97WeBWh9RaZ2NveguXOfQahmO)。**

<!--more-->

# 分析
原系统采用http通信(客户端一直等待服务器返回)，如何处理实时性，避免504的发生？

# 方法
## 方法1
修改服务器的请求响应时间。如：系统使用ngnix，修改ngnix.conf中的配置项。

```
#调整为300s
keepalive_timeout 300
```
> 不足：生产环境的nginx配置是个通用配置，改一处会牵连其他应用；过长时间占用服务器资源（如：上面配置，最长会有5min的占用）。

## 方法2
后端改异步查询，前端进行轮询获取结果（短轮询或长轮询）。

> 不足：轮询消耗服务器资源，过多无效的请求处理。

## 方法3
采用WebSocket进行结果集推送（SocketIO）。
前端采用node+[socket.io](https://socket.io/)，后端使用[netty-SocketIO](https://github.com/mrniko/netty-socketio),实现事件监听推送。

> 不足：目前浏览器版本支持不普遍，需要支持HTML5。但socketIO会对不支持的进行降级处理。

# 设计
最终，决定使用方法3实现异步改造。


## 面临的问题
### 如何识别某个客户端？
记录客户端的sessionId，用Redis的zset存储。

### 如何向指定窗口发送执行结果？
目前项目需求需要支持对指定客户端推送消息。浏览器不同的tab页，表示不同的客户端。针对不同的tab页进行执行结果推送。方法如下：

```
server.getClient(uuid).sendEvent("getResultSetEvent", data);
```

> 引申：如何实现消息的“私聊”？
给出两个思路：
+ 创建room
+ 使用namespace

## 方案实施
### scoket监听事件
+ addConnectListener：socket监听器，缓存客户端sessionId
+ addDisconnectListener：socket断开监听，清除客户端缓存sessionId

### 业务事件
+ execSqlEvent：执行sql事件
+ getStatusEvent：获取执行状态事件
+ getResultSetEvent：获取结果集事件

### 时序图
![即时查询异步时序图](http://of7369y0i.bkt.clouddn.com//2017/04/%E6%9E%B6%E6%9E%84%E8%AE%BE%E8%AE%A1/%E5%8D%B3%E5%B8%AD%E6%9F%A5%E8%AF%A2.png)

> 流程简要说明：
+ execSqlEvent事件注册后，客户端提交执行事件execSqlEvent。服务器先获取客户端可用锁（一个客户端只能支持一个sql执行计划），然后将执行记录先落地到DB，之后推送getStatusEvent事件，推送状态信息。
+ 服务端缓存相关数据信息，之后启动执行线程异步执行。
+ 执行线程持续的进行状态推送（getStatusEvent），并将中间结果落地到DB。
+ 推送getStatusEvent事件，如果失败，客户端展示失败信息；如果成功，推动getResultSetEvent，客户端展示结果集。

## 数据结构
### 缓存结构
+ zset（sessionId，status）：使用Redis记录客户端状态，保持同一个客户端只能有一个sql执行计划。（status：0-可用；1-不可用）**增加Redis锁**
+ map(sqlKey,resultset):记录某个执行结果,粒度以执行窗口为单位（一个窗口支持多条sql语句执行）。
+ map(sessionId,sqlKey):记录客户端和执行sql的对应关系，将指定结果集推送给指定客户端。

### 响应数据结构
#### socket响应数据结构
SocketDataBo
```
public class SocketDataBo implements Serializable {
    /**
     * 错误码：0-失败；1-成功
     */
    private Integer code = 1;
    /**
     * 错误信息
     */
    private String message;
    /**
     * 执行sql任务结果
     */
    private Result result;

    /**
     *执行sql任务结果
     */
    public class Result implements Serializable {
        /**
         * 任务id
         */
        private Integer resultId;
        /**
         * 状态（执行状态：0-未执行, 1-执行中，2-执行成功 3-执行失败）
         */
        private Byte execStatus;
        /**
         * 执行结果集
         */
        private ResultSetBo resultSetBo;
    }
}

```
#### 结果集结构
ResultSetBo
```
public class ResultSetBo implements Serializable {
    /**
     *列名称
     */
    private List<String> columnNames;
    /**
     * 行
     */
    private List<ResultRowBo> rows;
}

```

# 结语
类似的，网上有对webim的实现原理的讨论。

如：58到家的沈剑的一篇推文[http如何像tcp一样实时的收消息](http://mp.weixin.qq.com/s/6BCucq6QsH8lfDGLtQCl2A)可去围观。

---
参考：
+ [ websocket 和 socket.io 之间的区别](http://blog.csdn.net/lb7758zx/article/details/51513470)

+ [基于socket.io的实时消息推送](http://blog.xiayf.cn/2014/09/06/socket.io-push-server/)
