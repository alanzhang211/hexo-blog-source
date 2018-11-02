---
title: KEYTOOL和PORTECLE介绍
date: 2014-12-31 12:19:36
tags: [2014,KEYTOOL,PORTECLE,密钥管理]
category: 信息安全
---
# 介绍
keytool：JDK自带的密钥管理工具，目录：JDK安装bin目录下。
[JDK下载地址](http://www.oracle.com/technetwork/java/javase/downloads/index.html)

portecle：图形化界面的 JDK 中的命令行工具 keytool。可生成各种不同类型的密钥库，生成并存储相关的 X.509 证书、生成 CSRs、导入和储存信任的证书并进行维护。
[官网](http://portecle.sourceforge.net/)

本文以图解的方式介绍上述两个工具的操作过程。

<!--more-->

# 图解
## keytool
### 创建密钥库文件tomcat.keystore
```
keytool -genkey -alias alan -keypass 123456 -keyalg RSA -keystore D:\Tomcat\mykey\tomcat.keystore
```

![](https://github.com/alanzhang211/blog-image/raw/master/2014/12/31/1.JPG)

### 置密钥信息
![](https://github.com/alanzhang211/blog-image/raw/master/2014/12/31/2.JPG)

### 密钥文件
![](https://github.com/alanzhang211/blog-image/raw/master/2014/12/31/3.JPG)

控制台查看密钥信息使用keytool –list命令
```
keytool -list -v -keystore D:\Tomcat\mykey\tomcat.keystore
```

![](https://github.com/alanzhang211/blog-image/raw/master/2014/12/31/4.JPG)

### 导出密钥库证书文件
导出密钥库证书文件tomcat.cer

```
keytool -export -alias alan -keystore D:\Tomcat\mykey\tomcat.keystore -file D:\Tomcat\mykey\tomcat.cer
```
![](https://github.com/alanzhang211/blog-image/raw/master/2014/12/31/5.JPG)

### 证书文件产生

![](https://github.com/alanzhang211/blog-image/raw/master/2014/12/31/6.JPG)

将证书文件导入jssecacerts文件中，作为jre证书库
```
keytool -import -alias alan -file D:\Tomcat\mykey\tomcat.cer -keystore D:\Tomcat\mykey\jssecacerts
```
![](https://github.com/alanzhang211/blog-image/raw/master/2014/12/31/7.JPG)

### 证书库文件加入jre环境中
![](https://github.com/alanzhang211/blog-image/raw/master/2014/12/31/8.JPG)

>Jre是指JDK目录下的jre。目录：.\jre\lib\security。运行环境优先查找证书库jssecacerts文件，若不存在。或去找jre默认的证书库文件cacerts。


## portecle
### 运行portecle-1.7

![](https://github.com/alanzhang211/blog-image/raw/master/2014/12/31/9.JPG)

### 查看的证书
![](https://github.com/alanzhang211/blog-image/raw/master/2014/12/31/10.JPG)


输入密码，JRE中默认的是changeit。
![](https://github.com/alanzhang211/blog-image/raw/master/2014/12/31/11.JPG)

然后就会显示证书库中已存在的证书信息，如下图示：
![](https://github.com/alanzhang211/blog-image/raw/master/2014/12/31/12.JPG)

### 向证书文件中加入网关证书trustRoot.cer。
![](https://github.com/alanzhang211/blog-image/raw/master/2014/12/31/13.JPG)


然后，点击Import导入证书。导入时会提示给证书重命名，点击OK，导入成功，如下图示：

![](https://github.com/alanzhang211/blog-image/raw/master/2014/12/31/14.JPG)

### 导入成功
![](https://github.com/alanzhang211/blog-image/raw/master/2014/12/31/15.JPG)

至此，完成秘钥的管理工作。
