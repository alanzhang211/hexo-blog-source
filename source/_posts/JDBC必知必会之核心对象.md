---
title: JDBC必知必会之核心对象
date: 2016-12-14 21:32:46
tags: [java,JDBC]
category: [2016,数据库]
---

# JDBC知识点
## 核心对象简介
> Driver、DriverManager、Connection、Statement、ResultSet。

![时序图](http://of7369y0i.bkt.clouddn.com//2016/12/%E6%95%B0%E6%8D%AE%E5%BA%93/JDBC.JPG)

<!--more-->

### Driver
驱动程序对象的接口

### DriverManager 对象
针对具体的数据库厂商按JDBC规范设计的数据库驱动程序包提供具体实现。
#### 获取
通过反射加载驱动对象
Class.forName("oracle.jdbc.driver.OracleDriver");

### Connection 对象
#### 获取
Class.forName("oracle.jdbc.driver.OracleDriver");
Connection con = DriverManager.getConnection(url, "xxx","xxx");

#### 关闭
即时关闭释放资源，否则超过数据库的连接数限制，就会报错。

### Statement 对象
#### 获取
Statement stmt = con.createStatement();

获取结果集后，对结果集的处理必须确保Statement未关闭。

#### 主要实现
##### Statement
> createStatement()

+ 每次执行sql语句，数据库都要执行sql语句的编译
+ 最好用于仅执行一次查询并返回结果的情形，效率高于PreparedStatement

##### PreparedStatement
> prepareStatement(sql) 提供sql预编译过程

+ 避免SQL注入的问题。
+ Statement会使数据库频繁编译SQL，可能造成数据库缓冲区溢出。PreparedStatement 可对SQL进行预编译，从而提高数据库的执行效率。
+ PreperedStatement对于sql中的参数，允许使用占位符的形式进行替换，简化sql语句的编写。
+ batch操作

##### CallableStatement
> prepareCall(sql) 创建执行存储过程

+ 扩展 PreparedStatement，用来调用存储过程。
+ 它提供了对输出和输入/输出参数的支持。
+ CallableStatement 接口还具有对 PreparedStatement 接口提供的输入参数的支持。


#### 关闭
Statement 对象将由 Java 垃圾收集程序自动关闭。而作为一种好的编程风格，应在不需要 Statement对象时显式地关闭它们。这将立即释放 DBMS 资源，有助于避免潜在的内存问题。

### ResultSet 对象
#### 获取
+ 代表Sql语句的执行结果。Resultset封装执行结果时，采用的类似于表格的方式。
+ 维护了一个指向表格数据行的游标cursor，初始的时候，游标在第一行之前，调用ResultSet.next() 方法，可以使游标指向具体的数据行，进而调用方法获取该行的数据。

ResultSet rs = stmt.executeQuery("select * from t_table");

#### 关闭
未即时关闭，会导致游标超出。
