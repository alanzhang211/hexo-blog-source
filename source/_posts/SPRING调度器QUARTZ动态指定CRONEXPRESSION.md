---
title: SPRING调度器QUARTZ动态指定CRONEXPRESSION
date: 2016-08-02 21:33:52
tags: [2016,spring,quartz]
category: spring
---
            雨昏连夜催炎暑，暑炎催夜连昏雨。

            长簟水波凉，凉波水簟长。

            翠鬟双倚醉，醉倚双鬟翠。

            香枕印红妆，妆红印枕香。

                        –《菩萨蛮·雨昏连夜催炎暑》

![夜雨](http://of7369y0i.bkt.clouddn.com/2016/yeyu.jpg)
*丝丝凉意沁人心脾，梳理万千思绪。*

<!--more-->
# 背景
项目为实现定时向数据仓库中抽取元数据。
技术：定时Spring调度quartz框架+ThreadPoolTaskExecutor连接池+元数据抽取db-meta（在github的基础上改造，支持pg、hive、impala）。
# 实现方式
每天定时去数据仓库取数据，由于数据仓库中的数据库比较多，引入线程池，以数据库为粒度，每个数据库起一个线程进行抽取。
本文主要介绍自己在使用quartz上遇到的一些“坑”。
quartz的调度时间使用cron表达式。简单的做法是直接在xml配置文件中配置。

本文主要介绍自己在使用quartz上遇到的一些“坑”。
quartz的调度时间使用cron表达式。简单的做法是直接在xml配置文件中配置。
```
<!-- 配置Cron触发器(CronTriggerFactoryBean) -->
<bean id="myJobTrigger"
class="org.springframework.scheduling.quartz.CronTriggerFactoryBean">
<property name="jobDetail">
<ref bean="metaSchedulejobDetail" />
</property>
<property name="cronExpression">
<!-- 每天0点执行 -->
<value>0 0 0 * * ?</value>
</property>
</bean>
```
项目需求是需要动态指定，从数据库中读取cronExpression表达式。

# 实现
编写InitializingCronTrigger继承于Spring的CronTriggerFactoryBean。然后对其中的属性cronExpression进行动态指定（本项目从数据库中读取）。

## 配置文件
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

<!-- 配置自定义的时间任务(Job) -->
<bean id="metaSchedulejob" class="com.greenline.wemeta.schedule.MetaScheduleJob" />
<bean id="scheduleService" class="com.greenline.wemeta.biz.manager.impl.ScheduleServiceImpl" />

<!-- 配置方法调用任务工厂(XXXJobDetailFactoryBean) -->
<bean id="metaSchedulejobDetail" class="org.springframework.scheduling.quartz.MethodInvokingJobDetailFactoryBean">
<property name="targetObject">
<ref bean="metaSchedulejob" />
</property>
<property name="targetMethod">
<value>scheduleExecute</value>
</property>
<property name="concurrent">
<value>false</value>
</property>
</bean>

<!-- 配置Cron触发器(CronTriggerFactoryBean) -->
<bean id="myJobTrigger"
class="org.springframework.scheduling.quartz.CronTriggerFactoryBean">
<property name="jobDetail">
<ref bean="metaSchedulejobDetail" />
</property>
<property name="cronExpression">
<!-- 每天0点执行 -->
<value>0 0 0 * * ?</value>
</property>
</bean>

<!--自定义cron-->
<bean id="myCronTrigger" class="com.greenline.wemeta.schedule.InitializingCronTrigger">
<property name="jobDetail" ref="metaSchedulejobDetail"/>
<property name="scheduleService" ref="scheduleService"/>
</bean>

<!-- 配置调度器工厂(SchedulerFactoryBean) -->
<bean name="startQuertz" lazy-init="true" autowire="no"
class="org.springframework.scheduling.quartz.SchedulerFactoryBean">
<property name="triggers">
<list>
<ref bean="myCronTrigger" />
</list>
</property>
</bean>
</beans>
```
## InitializingCronTrigger类实现
```
public class InitializingCronTrigger extends CronTriggerFactoryBean {

private ScheduleService scheduleService;

private Logger logger = LoggerFactory.getLogger(InitializingCronTrigger.class);
public InitializingCronTrigger (){
super();
//this.getCronExpressionFromDB();
}

/**
    * 从数据库中获取cron表达式
    */
private void getCronExpressionFromDB() {
       MtScheduleVo scheduleVo = new MtScheduleVo();
       scheduleVo.setIsDeleted(WeMetaConstant.DELETED_NO);
       List<MtScheduleDo> scheduleDoList = scheduleService.getSchedules(scheduleVo);
       String cronExpression = WeMetaConstant.DEFAULT_CRON_EXPRESSION;
for (MtScheduleDo scheduleDo : scheduleDoList) {
if (WeMetaConstant.SCHEDULE_TYPE == scheduleDo.getScheduleType()) {
               cronExpression = scheduleDo.getCron();
           }
       }
logger.debug("cronExpression={}", JSON.toJSON(cronExpression));
//调用父类方法设置cron表达式
super.setCronExpression(cronExpression);
   }

public ScheduleService getScheduleService() {
return scheduleService;
   }

public void setScheduleService(ScheduleService scheduleService) {
this.scheduleService = scheduleService;
this.getCronExpressionFromDB();
}
}
```
其他基础Bean及相关基础配置说明省略（不是本文重点）。
需要注意的地方：
1. 使用Spring的property方式注入，Spring实现原理是set注入的实现。所以InitializingCronTrigger必须要有对应的set方法。
2. 关于this.getCronExpressionFromDB();的调用时机？
起初，我是在构造函数中调用类的私有方法this.getCronExpressionFromDB();结果junit跑用例发现
```
List<MtScheduleDo> scheduleDoList = scheduleService.getSchedules(scheduleVo);
```
scheduleService一直为null，没有注入成功。

后续将this.getCronExpressionFromDB();放在set方法内部，结果成功。

# 反思
类构造函数和类成员变量
jvm加载类的顺序：执行构造函数，初始化实例变量，执行实例方法。
通过spring的set注入的方式进行属性设值。
以下是debug控制台打印。
![](http://of7369y0i.bkt.clouddn.com/2016/121.JPG)
