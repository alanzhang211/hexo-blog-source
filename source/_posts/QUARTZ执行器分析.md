---
title: QUARTZ执行器分析
date: 2016-08-19 20:18:20
tags: [2016,quartz,spring]
category: spring
description: 项目近几天停滞，原因：前端资源缺少。自己也开始看了不少前端技术，如vue等。当然，此篇不介绍前端技术。还是对以前自己的疑惑点进行梳理。本文主要解读Spring+Quartz进行任务调度遇到的问题。
---
![click](https://github.com/alanzhang211/blog-image/raw/master/2016/quartz/click.jpg)
## 背景
在测试是Quartz调度任务时，有时会遇到修改服务器系统时间来实现任务重跑的效果。然而，这样并没什么卵用。现象是：怎么任务还没有触发，没有触发。。。然后，只能宠重启web服务器，然后就奇迹般的依据预定的时间触发了。
刨根问底：意识到Quartz将任务job信息进行缓存。导致，无论你改任务参数，打偶无法达到动态修改job的目的。
这几天有时间发刊Quartz源码并结合晚上已有的资料进行了梳理。
参见资料：http://blog.itpub.net/11627468/viewspace-1763498/
版本信息：org.quartz-scheduler-2.2.3
调度触发由trigger开始。如下图：

![时序图](https://github.com/alanzhang211/blog-image/raw/master/2016/quartz/shixutu.jpg)

## 原理分析
然后，依据轮询线程QuartzSchedulerThread进行轮询。从RAMJobStore中获取下一个可执行job。这里的RAMJobStore即使job的内存缓存容器。是在QuartzScheduler构造函数中初始化的。

![](https://github.com/alanzhang211/blog-image/raw/master/2016/quartz/p1.png)
QuartzSchedulerThread的run方法就是关键所在，去获得下一个可执行job。
跟进去看到，RAMJobStore会与job集合中的getNextFireTime判断，如下：

![](https://github.com/alanzhang211/blog-image/raw/master/2016/quartz/clipboard12.png)

### 问题：
**job执行后，如何去更新下次点火时间的？**

代码回到QuartzSchedulerThread的run方法中，看到如下一段代码：

![](https://github.com/alanzhang211/blog-image/raw/master/2016/quartz/clipboard21.png)

![](https://github.com/alanzhang211/blog-image/raw/master/2016/quartz/clipboard31.png)

进入CronTriggerImpl看到真相啦。

![](https://github.com/alanzhang211/blog-image/raw/master/2016/quartz/clipboard4.png)

至此，了解了job每次“点火”成功后都会更新下次点火时间。这就是为什么，将服务时间修改未过去时间，操了cron表达式指定的时间，无法出发job的原
### 问题：如何实现动态的修改cron表达式？

#### 简单思路
从内存jobStore中取出，trigger信息。然后更新相关出发参数。

#### 实现代码：
 ```
 @Service
public class SchedulerService {
@Resource
private SchedulerFactoryBean schedulerFactoryBean;/**
* 更新制定触发器cron表达式，并更新jobStore中的trigger
* @param newCron 新表达式
* @param triggerKey 触发器的key
*/
public void updateCron(String newCron,TriggerKey triggerKey) {
try {
Scheduler scheduler = schedulerFactoryBean.getScheduler();
//获取trigger
CronTrigger trigger = this.getTrigger(triggerKey);
//表达式调度构建器
CronScheduleBuilder scheduleBuilder = CronScheduleBuilder.cronSchedule(newCron);

//按新的cronExpression表达式重新构建trigger
trigger = trigger.getTriggerBuilder().withIdentity(triggerKey)
.withSchedule(scheduleBuilder).build();

//按新的trigger重新设置job执行
scheduler.rescheduleJob(triggerKey, trigger);
} catch (SchedulerException e) {
e.printStackTrace();
}
}
 ```

 #### 测试代码
 ```
 @RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = “classpath:spring/manager.xml”)
public class SchedulerServiceTest {
@Resource
private SchedulerService schedulerService;
@Test
public void testUpdateCron() throws Exception {
TriggerKey triggerKey = TriggerKey.triggerKey(“myCronTrigger”,”DEFAULT”);
String newCron = “0 45 20 * * ?”;
schedulerService.updateCron(newCron,triggerKey);
Trigger trigger = schedulerService.getTrigger(triggerKey);
System.out.println(JSON.toJSON(trigger));
}@Test
public void testGetTrigger() throws Exception {
TriggerKey triggerKey = TriggerKey.triggerKey(“myCronTrigger”,”DEFAULT”);
Trigger trigger = schedulerService.getTrigger(triggerKey);
System.out.println(JSON.toJSON(trigger));
}
}
```

*参见：http://my.oschina.net/u/1177710/blog/284608*
