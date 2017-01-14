---
title: storm集群supervisor节点异常退出问题排查
date: 2015-07-03 20:48:50
tags:
- Storm
- Supervisor
- 异常排查
categories:
- Storm
---

问题出现
--

测试storm集群为0.9.4版本，前段时间出现supervisor进程挂掉，而其上work进程仍然运行的诡异情况，通过日志看到supervisor进程挂掉之前出现以下异常：

![](https://raw.githubusercontent.com/maohong/picture/master/20150701/1.png)

问题排查过程
--

很明显，是commons-io包的FileUtils工具类抛出的异常，原因是在调用commons-io包的FileUtils工具类做move directory操作时，目的文件夹已存在。

查看调用代码（supervisor.clj的第374行），是调用download-storm-code方法从nimbus下载topology的代码，并且download-storm-code方法中做代码下载前加了锁避免并发写文件。

![](https://raw.githubusercontent.com/maohong/picture/master/20150701/2.png)

![](https://raw.githubusercontent.com/maohong/picture/master/20150701/3.png)

果然，这里没有判断stormroot文件夹是否已存在，是个bug，具体可见这个issue：[https://issues.apache.org/jira/browse/STORM-805](https://issues.apache.org/jira/browse/STORM-805)。

这个问题在0.9.5版本中随着STORM-130一起修复了，代码如下：

![](https://raw.githubusercontent.com/maohong/picture/master/20150701/4.png)

但这里有三个问题：

1. 在调用download-storm-code方法前，代码中已做判断是否已下载topology代码，若已下载就不会调用download-storm-code方法了。为何进入这个方法后，做move directory操作时，代码却已经下载好了呢？

2. storm的历史发布版本有很多，为何0.9.4版本里会出现这个不该出现的问题，0.9.4相对老的版本是不是做了什么修改？

3. 为何抛出异常后，supervisor进程就这么直接退出了？太弱了吧。。
<!--more-->
继续看0.9.4的源码发现，**supervisor中有以下两个事件线程，都会调用download-storm-code方法**：

**一个是synchronize-supervisor，用于同步nimbus任务**，每隔10秒执行一次，会调用mk-synchronize-supervisor方法，以及时获取nimbus分配给该supervisor的新任务并移除已分配但不再需要执行的任务。

**另一个是sync-processes，用于根据任务变化同步管理worker进程**，执行周期由SUPERVISOR-MONITOR-FREQUENCY-SECS（默认3秒）指定，会调用sync-processes方法，以关闭当前不处于valid状态的worker和启动新分配给该supervisor的worker。

其中，mk-synchronize-supervisor方法和sync-processes方法都会调用download-storm-code方法。

两个事件线程的定义：

![](https://raw.githubusercontent.com/maohong/picture/master/20150701/5.png)

mk-synchronize-supervisor方法调用download-storm-code方法：

![](https://raw.githubusercontent.com/maohong/picture/master/20150701/6.png)

sync-processes调用download-storm-code方法：

![](https://raw.githubusercontent.com/maohong/picture/master/20150701/7.png)

mk-synchronize-supervisor方法和sync-processes方法在调用前都会判断topology代码是否已下载，所以，出现上述异常的原因很可能是两个线程再调用download-storm-code方法时不同步引起的，即同时判断到需要下载topology代码并进入了download-storm-code方法，从而产生两次move directory的操作引发异常。

虽然download-storm-code方法内部通过加锁控制了写文件时的并发，但对进入download-storm-code方法并没有做好同步。

再回过头看0.9.5版本的代码，虽然在move directory前判断了目的文件夹是否存在以避免问题，但实际上还是存在两个线程同时进入download-storm-code方法的问题。

最后再比较了下0.9.3和0.9.4的代码（supervisor.clj），发现0.9.4的sync-processes方法中调用download-storm-code的逻辑是新加进去的，也就是说这个bug是0.9.4新引入的，以前的版本不会存在这个问题。

左边为0.9.3，右边为0.9.4：

![](https://raw.githubusercontent.com/maohong/picture/master/20150701/8.png)

关于第3个问题，再回看定义synchronize-supervisor事件线程的代码，是通过事件管理器event-manager来实现的，查看event.clj中的实现，event-manager会从一个LinkedBlockingQueue取出新事件并启动线程处理，线程若抛出非Interrupted异常，则直接退出进程了。

![](https://raw.githubusercontent.com/maohong/picture/master/20150701/9.png)

至此，问题分析完毕。