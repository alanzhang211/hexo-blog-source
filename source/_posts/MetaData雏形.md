---
title: MetaData雏形
date: 2017-01-22 18:35:13
tags: [2017,metadata,元数据管理,架构]
category: [软件设计]
---
# 背景
随着数仓团队的壮大，以及数据运营分析师的扩张。数据处理的多样性及维护成本不多递增。为了实现数据规范化管理，降低维护成本。Metadata元数据管理平台应运而生。

# 系统分析
## 功能
主体功能
+ 实现元数据的管理，从源表抽取业务数据。便于数仓和数据运营查看管理元数据。
+ 任务的开发维护，内部结合调度系统执行kettle脚本，完成数仓工程师日常的抽取和开发工作。
+ 即时查询处理，实现数仓和分析师对业务数据表的查询和验证。


## 架构
![MetaData架构](http://of7369y0i.bkt.clouddn.com//2017/01/%E6%9E%B6%E6%9E%84%E8%AE%BE%E8%AE%A1/MetaData%E6%9E%B6%E6%9E%84.jpg)

依旧是分层结构，SpringMVC+MyBatis+Vue构成整体应用框架。

<!--more-->

## 架构讲解
### Web层
采用vue框架,这是一个符合MVVM模式的前端模板引擎（公司的项目中普遍使用Freemark模板引擎），响应式开发组件（数据和DOM绑定）。更详细内容参见[vue官网](http://cn.vuejs.org/)
![MVVM](http://of7369y0i.bkt.clouddn.com//2017/01/%E6%9E%B6%E6%9E%84%E8%AE%BE%E8%AE%A1/mvvm.png)

### API层
对外提供两种接口形式。
+ RPC API使用 Dubbo发布服务。
+ HTTP 接口提供符合RESTful API规范接口。

### Metadata核心层
#### authorization
实现权限的认证处理，依赖于公司账户权限认证服务。通过Redis处理用户session等登录信息。

#### Metadata-Manager
Metadata业务核心模块，实现相关业务逻辑处理。

#### Metadata-Query
元数据查询组件处理，主要负责读操作处理。隔离于Metadata-Manager方便后续进行读写分离。

#### Log
记录用户操作日志，元数据更改日志。后续可能会做一些审计等处理。目前，只是日志记录组件。

#### Metadata-dao
数据持久化层，底层使用MyBatis实现数据库操作,连接池采用[Druid](https://github.com/alibaba/druid)。

#### Message
消息队列处理，主要对接MQ异步消息处理。

#### Third-Manager
其他系统对接入口，实现接口的二次封装处理，消除差异化对Metadata-Manager的影响。

#### Scheduler
任务调度组件，支撑抽取任务周期执行，缓存过期轮询处理。

#### 抽取引擎
+ Metadata-Extractor：核心抽取逻辑，对接Metadata-Manager。
+ Sql-Crawler：主要负责元数据的抽取处理。目前系统支持关系型数据库，分布式数据库的抽取。抽取组件借鉴[db-meta](https://github.com/wukenaihe/db-meta)并进行定制化，支持hive，impala等。
+ File-Crawler：对于文件抽取目前还不支持（File-Crawler后续规划开发中）。

#### 执行引擎
主要负责即时查询业务逻辑。采用JDBC直连数据库操作，支持缓存select查询数据处理，可以下载的缓存结果等。
+ Metadata-SqlExec：sql执行核心组件，封装一系列Sql执行逻辑。
+ Metadata-SqlParser：sql语句解析组件。借鉴于[JSqlParser](https://github.com/JSQLParser/JSqlParser),并做了定制化处理。

### DB层
#### Metadata-store
Metadata系统数据持久化（存储于Mysql）

#### Redis
分布式缓存，支撑用户登录信息缓存，分布式锁等。

#### 数据源
元数据抽取来源，支持关系数据库，分布式数据库，HDFS文件系统等。

### 中间件
+ MQ：实现分布式异步任务处理。
+ Dubbo：支撑rpc服务。
+ 其他

### Other-System
其他系统对接，主要对接一下系统
+ Schedule-System：调度系统，支撑数仓人员任务运行。是一个任务执行系统。
+ User-System：账户系统，实现权限处理。
+ 其他

# 结语
算是半年下来蕴蓄的一个初生“儿”吧。细细看来，还有很多组件耦合需要分离。业务线不完善，元数据开发平台的闭环还有很多缺口。如：元数据分类管理，血缘关系追踪等。

目前在看Linkin开源的元数据管理[WhereHows](https://github.com/linkedin/WhereHows)，希望可以吸取更多精华。
