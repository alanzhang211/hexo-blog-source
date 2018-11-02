---
title: 《Java多线程编程核心技术》读书笔记
date: 2016-11-10 22:06:23
tags: [2016,java,线程]
category: 读书笔记
---
![Java多线程编程核心技术](https://github.com/alanzhang211/blog-image/raw/master/2016/book/1110197774.jpg)

+ [百度网盘资源下载](http://pan.baidu.com/s/1slKqcYL)

<!--more-->

# 停止线程
![](https://github.com/alanzhang211/blog-image/raw/master//book/2016/Java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%BC%96%E7%A8%8B%E6%A0%B8%E5%BF%83%E6%8A%80%E6%9C%AF/clipboard.png)

如果在for循环中interrupted，for循环后续的语句还会执行，线程并非停止，可以抛出异常。

![](https://github.com/alanzhang211/blog-image/raw/master/book/2016/10clipboard.png)

如果在for循环中interrupted，for循环后续的语句还会执行，线程并非停止，可以抛出异常。
![](https://github.com/alanzhang211/blog-image/raw/master/book/2016/10clipboard1.png)


![](https://github.com/alanzhang211/blog-image/raw/master/book/2016/10clipboard2.png)


### 在沉睡中停止
#### 如果线程在sleep下停止会怎样？
sleep下停止线程，会清楚停止状态值。

### stop暴力停止
stop会抛出ThreadDeath异常；次异常不需要显示的补获。
此方法已被作废；如果使用会导致一些清理工作没能完成，导致数据不一致问题。

### 使用return停止线程
[图片]

**建议使用抛出异常的方式实现线程停止，这样在catch块中可以向上抛**

### 暂停线程
java使用suspend暂停线程，使用resume恢复

#### suspend与resume的缺陷
+ 独占
+ 不同步


### yield

#### 作用
放弃当前cpu资源，给其他线程。但放弃的时间不确定。

### 线程的优先级
设置线程优先级有助于“线程规划器”决定下一个执行线程。

使用setPriority

jdk中优先级分为1-10个等级。

#### 优先级的继承特性
如：A线程启动B线程，则B的优先级和A的相同。

#### 优先级具有规则性
+ 高优先级的总是**大部分**先执行。
+ 线程谁先执行完和代码调用无关。



#### 优先级具有随机性
+ 优先级高的不一定每次都先执行完。

#### 守护线程
+ 当进程中不存在非守护线程了，守护线程自动销毁。**典型的如垃圾回收线程**
+ 作用：为其他线程的运行提供便利服务

---
# 对象及变量的并发访问
## synchronized
### 方法内的变量是线程安全的
+ 方法内部变量时私有的

### 实例变量是非线程安全的
+ 两个线程访问**同一个对象**的同步方法，一定是线程安全的。

### synchronized方法和锁对象
+ 访问锁对象的被synchronized修改的的实例方法；是同步的，需要排队访问。

### 脏读


### synchronized锁重入

+ 在synchronized的方法或块内部调用synchronized方法或块，jing将可以得到锁。
+ 可重入锁：自己可以再次获得自己的内部锁


### 出现异常，锁自动释放

### 同步不具有继承性
---

## synchronized同步语句块
### synchronized代码块间的同步性
+ 当一个线程访问object的一个synchronized(this)同步快时，其他线程对同一个object的其他synchronized(this)同步块的访问是阻塞的，说明，synchronized使用的“对象解释器”只有一个。
 ![image](https://github.com/alanzhang211/blog-image/raw/master/2016/book/Java%E5%A4%9A%E7%BA%BF%E7%A8%8B%E7%BC%96%E7%A8%8B%E6%A0%B8%E5%BF%83%E6%8A%80%E6%9C%AFsychronized.JPG)

### 任意对象作为对象监视器
+ java 支持对“任意对象”作为监视器，任意对象大多数是实例变量及方法参数，使用格式synchronized(非this对象)。
+ 多个线程持有同一个对象的“对象监视器”，只能有一个线程可以访问。
+ 捕鱼其他锁this争抢this锁，可提高效率。
+ 非this监视器必须是同一个对象，如：实例变量。（局部变量不满足）

### 静态synchronized方法与synchronized(class)代码块。

+ 同步静态方法，是对Class对象持有锁。
+ 同步class对象和同步静态方法作用一样。

### 数据类型String的常量池特性
+ jvm具有String常量池的缓存功能。
+ 不使用String作为锁对象。

### 同步synchronized方法无限等待与解决
+ 使用同步快处理（synchronized(Object)）

### 多线程的死锁

### 内置类和静态内置类

### 锁对象的改变
+ 持有相同锁对象，线程同步。
+ 只要对象不变，对象属性改变，依旧是同步的。

---
## volatile
+ 作用：使变量在多个线程间可见。

### 关键字volatile与死循环
+ 从线程公共堆栈中获取变量值，而不是线程的私有栈。
+ volatile不支持原子操作。

### synchronized与volatile对比
+ volatile更加轻量级，性能比synchronized好。
+ volatile只能修饰变量，synchronized可以修饰方法，以及代码块。
+ 多线程访问volatile不会阻塞。
+ volatile保证数据的可见性，不能保证原子性。
+ synchronized可以保证可见性和原子性。
+ volatile解决的是多线程变量的可见性；synchronized解决的是多线程资源的可见性。
+ 线程安全包括：可见性和原子性。

### volatile非原子特性
+ 使用synchronized进行同步
+ 使用AtomicInteger可进行原子操作。

### synchronized代码块有volatile同步的功能
+ 互斥性和可见性

---
# 线程间通信
## 等待/通知机制（wait/notify）
### wait
+ wait是object对象的方法
+ 在wait之前必须获取对象级别的锁，即只能在同步方法或同步块中调用wait方法
+ 调用wait时没有持有适当的锁，则抛出运行时异常illegalMonitorStateException，无需使用try...catch...捕获。

### notify
+ notify是Object对象的方法。
+ 必须获取对象级别的锁，即只能在同步方法或同步块中调用
+ 在执行notify后，当前线程不会立即释放该对象的锁，wait的也不会立刻得到该对象锁。
+ 执行notify的线程执行完，即退出synchronized代码块，才会释放锁，呈wait的线程才能得到对象锁。
+ 第一个获得notify的线程执行完毕，释放对象锁，如果没有发出notify或notifyAll,其他线程也不能获取到对象锁。


+ **方法wait被执行后，所被释放；执行完notify后，锁却不被释放；必须执行完notify所在的synchronized代码块才能释放。**
+ **执行完同步代码块就会释放锁对象。**
+ **在执行同步代码块的过程中，遇到异常而导致线程终止，会释放锁。**
+ **在执行同步代码块中执行wait，会释放锁。**

### notifyAll
+ 唤醒所有等待线程

## 生产者&消费者
==使用notifyAll可以避免程序假死==

### 通过管道进行线程间通信
#### pipeStream用于不同线程之间传递数据

## join
等待线程对象销毁

+ join具有使线程运行的作用，有些类似同步的效果。
+ join在内部使用的是wait方法进行等待；synchronized使用“对象监视器”原理做同步。

### join与异常
+ join与interrupt方法如果彼此相遇，会出现异常。

### join(long)与sleep(long)区别
+ join释放锁

---
## ThreadLocal
*变量共享可以使用public static变量的形式，所有线程都使用同一个public static变量。如果实现每个线程都有自己的共享变量，可以使用ThreadLocal。*

*ThreadLocal可以比喻全局存放数据的盒子；盒子可以存储每个线程的私有数据。*

### 方法get与null
+ ThreadLocal解决的是不同线程之间的隔离性。

### InheritableThreadLocal
+ 可以在子线程中获取主线程中取值。
+ **如果主线程将值改变，则子线程依旧拿到的是旧值。**

---
# Lock
## ReentrantLock
+ lock\unclock

### Condition实现等待/通知
+ 一个lock可以创建多个Condition对象。
+ notify/notifyAll 选择线程是随机的；Condition可以选择性的通知。
+ 必须在condition.await()之前调用lock.lock()
+ signal/signalAll

### 公平锁/非公平锁
+ 公平锁：按照线程加锁的顺序来分配
+ 非公平锁：抢占机制

## ReentrantReadWriteLock
+ 读锁共享
+ 写锁互斥
+ 读写互斥

# 定时器Timer
+ 设置计划任务
+ 执行任务的类TimerTask

## Timer的使用
### schedule
+ Timer.schedule 启动一个新线程。如果没有设置Timer为守护线程，则则一直运行。
+ 执行时间早于当前时间，则任务立刻执行。
+ timer中允许多个task。以队列的方式执行，可能和预期的不一致。

### TimerTask
+ cancel将自身从队列中移除，其他任务不受影响。
+ timer的cancel是将队列全部清除。
+ Timer类中的cancel类有可能没有抢占到queue的锁，TimerTask类中的任务继续执行。

# 单例模式与多线程
## 延迟加载
多线程下，延迟加载时错误的单例模式

**解决方法**
+ synchronized关键字，对getInstance方法加锁。
+ 使用DCL双检查锁机制。

## 使用静态内之类实现单例模式


## 序列化与反序列化

## 使用static静态代码块实现单例
+ static中的代码在实用类时已经执行。

## 使用enum枚举类只想单例模式
+ 在使用枚举类时，构造方法自动被调用。
