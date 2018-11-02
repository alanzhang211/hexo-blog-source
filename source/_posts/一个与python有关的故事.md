---
title: 一个与python有关的故事
date: 2017-11-22 08:44:44
tags: [2017,软件设计]
category: 软件设计
---
# 背景
近期系统需求上来迟缓，也腾出了一些时间来梳理沉淀。对系统的代码进行审视，sonar静态检测，消除检测问题。也是还债的过程。回顾过往的坑，有些东西还是要沉淀下来。接下来节听听故事的来龙去脉。

# 一个故事
先说说之前印象比较深刻都一个需求，一个踩了很多坑的故事。

<!--more-->

## 需求
> 实现一个web端的python代码执行器，能够完成简单的数据分析功能。

需求很简单；编程语言使用java，实现一个python的执行器功能。先从网上搜一把有没有现成的轮子。

## 技术选型

1. 方案1：使用Jython包实现。

优点：封装良好

缺点：对python的第三方包的支持不好。

2. 方案2：使用Runtime.getRuntime()执行脚本文件。

优点：调用简单（同在命令行中执行python）

缺点：无封装调用。第三方包必须安装在运行环境中。

> 综合：采用方案2实现python执行器。

## 设计
### 组件图

![组件图](https://github.com/alanzhang211/blog-image/raw/master//2017/11/sheji/%E7%BB%84%E4%BB%B6%E5%9B%BE.png)
> 说明：
> 1、通信方式采用http和websocket进行。其中：websocke负责python的执行和停止事件处理（需要持续推送执行状态）。其他操作统一采用http方式。
> 2、manager模块负责核心业务分发。
> 3、执行器抽离，面向接口编程，方便后续扩展其他执行器类型。

### 类图

![类图](https://github.com/alanzhang211/blog-image/raw/master//2017/11/sheji/%E7%B1%BB%E5%9B%BE.png)

> 说明
> 1、PythonController： http接口层，提供给前端调用，返回json格式数据。
> 2、PythonSocket：  WebSocket服务，和客户端建立socket通信。
> 3、IExector： 执行器接口，提供（execute和stop接口）；PythonExector Python执行器实现。
> 4、PythonThread ：线程，采用线程池进行异步处理。
> 5、PythonManager ：python业务实现类，提供核心业务处理。

## 代码实现

java 中执行python代码片段：

```
Process process = null;
logger.info("python version={}",pythonVersion);
String cmd = String.format("python"+pythonVersion+" %s", pythonFile);
logger.info("执行python, 命令:{}", cmd);
// 执行命令
process = Runtime.getRuntime().exec(cmd);
```

## 遇到的问题
### 如何实现执行超时处理
```
process.waitFor();
```
是一个阻塞调用，一直等待，直到有响应为止。所以，为了避免系统资源占用。需要设置一个超时，超过指定时间，线程终止。

这里增加一个超时线程进行处理。
```
    /**
     * 超时线程
     */
    private static class TimeoutWorker extends Thread {
        private final Process process;
        private Integer exit;

        private TimeoutWorker(Process process) {
            this.process = process;
        }

        public void run() {
            try {
                exit = process.waitFor();
            } catch (InterruptedException e) {
                return;
            }
        }
    }
```

```
......(略去上文)
if (process != null) {
    TimeoutWorker worker = new TimeoutWorker(process);//将执行进程放进超时线程中执行。
    worker.start();
    try {
        logger.info("timeout={}(ms)", timeout);
        worker.join(timeout);//加入当前线程，timeout后，退出。
        if (worker.exit != null){
            int insRet = worker.exit;
......(略去下文)
```

补充线程知识：
> thread.Join把指定的线程加入到当前线程，可以将两个交替执行的线程合并为顺序执行的线程。比如在线程B中调用了线程A的Join()方法，直到线程A执行完毕后，才会继续执行线程B。

### 如何实现安装包安装
python的安装使用

```
python setup.py install
```

进行安装第三方包。在linux环境下，需要切到安装包的解压根目录，然后执行以上指令。于是，简单编写shell脚本python_install.sh。同时，实现动态注射shell指令参数。

```
#!/bin/bash
filepath=$1
pythonVersion=$2
echo "the file path is : ${filepath}"
echo "the python version is : ${pythonVersion}"
cd ${filepath} && python${pythonVersion} setup.py install
```

> 遇到个坑：.sh脚本在windows系统下编辑后，放在Linux环境，发现.sh脚本文件运行失败。单独执行语句，没问题。后来发现原来文件编码导致linux不识别。参见：[ 执行shell脚本时提示bad interpreter:No such file or directory的解决办法](http://blog.csdn.net/russ44/article/details/51694047)

### 如何获取process的错误信息
程序最初通过标准输出流获取process执行输出，本以为可以回去到所有的结果输出（正确输出和错误信息）。发现，并没有。而且，有时候程序一只卡死，waitFor()方法阻塞无法返回，直到超时为止。
```
......(略去上文)
InputStreamReader ir  = new InputStreamReader(process.getInputStream());
BufferedReader bufferedReader = new BufferedReader(ir);
String data = null;
while ((data = bufferedReader.readLine()) != null) {
......(略去下文)

```
后来，网上搜了搜，发现：
> Runtime对象调用exec(cmd)后，JVM会启动一个子进程，该进程会与JVM进程建立三个管道连接：标准输入，标准输出和标准错误流。
```
process.getErrorStream();
process.getInputStream();
process.getOutputStream();
```
waitFor()方法阻塞无法返回的问题。原因是getErrorStream()所对应的那个缓冲区没有被清空。所以，程序中增加对getErrorStream()错误输出流处理。同时，也解决了无法获取到process错误信息的问题。

所以，类似像上面那样。读取标准错误流，就可以接收到process的错误信息。
```
......(略去上文)
InputStreamReader ir  = new InputStreamReader(process.getErrorStream());
BufferedReader bufferedReader = new BufferedReader(ir);
String data = null;
while ((data = bufferedReader.readLine()) != null) {
......(略去下文)

```

# 故事结尾
这个故事让我学习了python，成了py新手。踩了不少的坑，就要挤出时间去填上。
