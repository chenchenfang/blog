+++
title = '多线程初始化类死锁'
date = 2025-01-21T21:20:00+08:00
draft = false
+++
# 多线程初始化类死锁
## 前言
有一个MQ消费者程序,很久没有更新重启了,最近更新了代码,然后升级.
升级完之后看程序日志的打印,发现不往下走了,日志不动了,这个肯定不正常
## 解决思路
碰到这种问题,需要查看两个数据,一个是jvm内存数据也就是dump出来的数据,还有就是当前程序调用情况.
由于程序是刚重启,内存的问题可能性不大,所以我使用**jstack**把当前程序堆栈调用情况输出到文件中.
接下来就是检查这个文件中的内容,卡着不动,肯定是有代码了,然后发现很多都是`java.lang.Thread.State: RUNNABLE`,具体看如下

```java
"thread名称***" #32 [27804] prio=5 os_prio=0 cpu=1379.98ms elapsed=362.77s tid=0x00007fdd6c8ec670 nid=27804 waiting on condition  [0x00007fccf8d9b000]
   java.lang.Thread.State: RUNNABLE
	at staticdata.StaticDataLoad.loadCache(StaticDataLoad.java:129)
	- waiting on the Class initialization monitor for tools.Tool
	at staticdata.ExSystem.getCore_server_config(ExSystem.kt:1723)
	at staticdata.MediaDealIdInfo.init(MediaDealIdInfo.java:15)
	at staticdata.StaticDataLoad.<init>(StaticDataLoad.java:41)
	at staticdata.MediaDealIdInfo.<init>(MediaDealIdInfo.java:10)
	at staticdata.StaticData2.<clinit>(StaticData2.kt:452)
    ************************
	at java.lang.Thread.runWith(java.base@21.0.1/Thread.java:1596)
	at java.lang.Thread.run(java.base@21.0.1/Thread.java:1583)



"ConsumeMessageThread_******_2" #192 [27968] prio=5 os_prio=0 cpu=8.05ms elapsed=361.06s tid=0x00007fc9fc0025c0 nid=27968 waiting on condition  [0x00007fcad01ec000]
   java.lang.Thread.State: RUNNABLE
	at bean.config.OkHttpConfig.<init>(OkHttpConfig.java:33)
	- waiting on the Class initialization monitor for staticdata.StaticData2
	at tools.Tool.<clinit>(Tool.java:3779)
org.apache.rocketmq.client.impl.consumer.ConsumeMessageConcurrentlyService$ConsumeRequest.run(ConsumeMessageConcurrentlyService.java:402)
	at java.util.concurrent.Executors$RunnableAdapter.call(java.base@21.0.1/Executors.java:572)
	at java.util.concurrent.FutureTask.run(java.base@21.0.1/FutureTask.java:317)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(java.base@21.0.1/ThreadPoolExecutor.java:1144)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(java.base@21.0.1/ThreadPoolExecutor.java:642)
	at java.lang.Thread.runWith(java.base@21.0.1/Thread.java:1596)
	at java.lang.Thread.run(java.base@21.0.1/Thread.java:1583)
```

两个线程分别在等待对方类初始化完成,导致程序卡主不动了,由此就找到了原因

## 解决方案

减少类初始化时候的代码,改为方法调用,避免不必要的麻烦.
