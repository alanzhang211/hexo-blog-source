---
title: WebSocket+Ngnix负载均衡处理
date: 2017-04-26 22:48:46
tags: [2017,Ngnix,websocket]
category: 软件设计
---
# 项目实施

继上一次解决[SocketIO解决504错误](http://alanzhang.me/2017/04/10/SocketIO%E8%A7%A3%E5%86%B3504%E9%94%99%E8%AF%AF/),在项目实施中采用ngnix实现websocket的负载均衡，又采坑了。网络拓扑如下示：

![网络拓扑1](http://of7369y0i.bkt.clouddn.com//2017/04/%E6%9E%B6%E6%9E%84%E8%AE%BE%E8%AE%A1/socket1.png)

## ngnix转发飘移问题
发现服务端监听到的client连接频繁的出现重连现象。
![sessionId重连](http://of7369y0i.bkt.clouddn.com//2017/04/jvm/socket-sessionid.JPG)

<!--more-->

## 问题分析
socketIO在请求时，以 long-polling的方式进行通信。而ngnix默认的负载均衡方式是轮询（round robin），会根据请求的时间顺序去分配后端服务。就会导致是上面的建立连接（onConnect）和断开连接（onDdisConnect）的问题的出现。
![url](http://of7369y0i.bkt.clouddn.com//2017/04/%E6%9E%B6%E6%9E%84%E8%AE%BE%E8%AE%A1/polling.JPG)

## 问题解决
### ngnix负载均衡策略
#### 轮询（默认）
每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器down掉，能自动剔除。
```
upstream backserver {
    server 192.168.0.14;
    server 192.168.0.15;
}
```
#### 指定权重
指定轮询几率，weight和访问比率成正比，用于后端服务器性能不均的情况。
```
upstream backserver {
    server 192.168.0.14 weight=10;
    server 192.168.0.15 weight=10;
}
```

### IP绑定 ip_hash
每个请求按访问ip的hash结果分配，这样每个请求访问固定一个后端服务器，假如当前server不能提供服务，就会根据当前的哈希值再哈希出一个新哈希值，选择另一个服务器继续尝试，尝试的最大次是upstream中server的个数，假如server的个数超过20，也就是要最大尝试次数在20次以上，当尝试次数达到20次，仍然找不到一个合适的服务器，ip_hah策略不再尝试ip哈希值来选择server,而在剩余的尝试中，它会转而使用RR的策略，使用轮循的方法，选择新的server。**同时，可以解决session的问题**。
```
upstream backserver {
    ip_hash;
    server 192.168.0.14:88;
    server 192.168.0.15:80;
}
```

#### fair（第三方）
按后端服务器的响应时间来分配请求，响应时间短的优先分配。
```
upstream backserver {
    server server1;
    server server2;
    fair;
}
```

### url_hash（第三方）
按访问url的hash结果来分配请求，使每个url定向到同一个后端服务器，后端服务器为缓存时比较有效。
```
upstream backserver {
    server squid1:3128;
    server squid2:3128;
    hash $request_uri;
    hash_method crc32;
}
```
综上，WebSocket+Ngnix采用ip_hash的负载均衡策略，防止请求漂移问题。

![](http://of7369y0i.bkt.clouddn.com//2017/04/%E6%9E%B6%E6%9E%84%E8%AE%BE%E8%AE%A1/socket2.png)


+ **注意均衡算法要使用 ip_hash , 防止使用 long-polling 通信时请求分发到了不同的服务器导致异常。**

+ **刷新浏览器时，相当与客户端首先disconnect然后重新建立一次connect。并且此socket.io会默认重连之前断开的连接。**

## ngnix配置

```
    upstream websocket {
        ip_hash;
        server  192.168.0.1:9000;
        server  192.168.0.2:9000;
    }

    server {
        listen 9000;
        charset utf-8;
        location / {
            proxy_pass http://websocket;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $http_host;
        }
     }
```
## 参考资料
+ [nginx负载均衡基于ip_hash的session粘帖](https://www.oschina.net/question/12_24613)
+ [Nginx负载均衡_IP_HASH](http://www.360doc.com/content/14/1225/14/7635_435663893.shtml)
+ [Nginx 负载均衡配置和策略](http://outofmemory.cn/code-snippet/3040/Nginx-load-junheng-configuration-strategy)
+ [socket.io client + socketio-netty server简析](http://blog.csdn.net/zhou4700219/article/details/53518823)
+ [Nginx负载均衡时RR和ip_hash策略](http://uule.iteye.com/blog/2236475)
