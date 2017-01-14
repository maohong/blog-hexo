---
title: Spark on yarn的内存分配问题
date: 2015-08-11 13:23:13
tags:
- Spark
- Yarn
- 内存分配
categories:
- Spark
---

问题描述
--

在测试spark on yarn时，发现一些内存分配上的问题，具体如下。

在$SPARK_HOME/conf/spark-env.sh中配置如下参数：

> SPARK_EXECUTOR_INSTANCES=4            *在yarn集群中启动的executor进程数*
> 
> SPARK_EXECUTOR_MEMORY=2G              *为每个executor进程分配的内存大小*
> 
> SPARK_DRIVER_MEMORY=1G                *为spark-driver进程分配的内存大小*

执行$SPARK_HOME/bin/spark-sql –master yarn，按yarn-client模式启动spark-sql交互命令行（即driver程序运行在本地，而非yarn的container中），日志显示的关于AppMaster和Executor的内存信息如下：

![](https://raw.githubusercontent.com/maohong/picture/master/20150813/sparkonyarn-1.png)

![](https://raw.githubusercontent.com/maohong/picture/master/20150813/sparkonyarn-2.png)

日志显示，AppMaster的内存是896MB，其中包含了384MB的memoryOverhead；启动了5个executor，第一个的可用内存是530.3MB，其余每个Executor的可用内存是1060.3MB。

到yarnUI看下资源使用情况，共启动了5个container，占用内存13G，其中一台NodeManager启动了2个container，占用内存4G（1个AppMaster占1G、另一个占3G），另外3台各启了1个container，每个占用3G内存。

![](https://raw.githubusercontent.com/maohong/picture/master/20150813/sparkonyarn-3.png)

再到sparkUI看下executors的情况，这里有5个executor，其中driver是运行在执行spark-sql命令的本地服务器上，另外4个是运行在yarn集群中。Driver的可用storage memory为530.3MB，另外4个都是1060.3MB（与日志信息一致）。

![](https://raw.githubusercontent.com/maohong/picture/master/20150813/sparkonyarn-4.png)

那么问题来了：

1. Yarn为container分配的最小内存由yarn.scheduler.minimum-allocation-mb参数决定，默认是1G，从yarnUI中看确实如此，可为何spark的日志里显示AppMaster的实际内存是896-384=512MB呢？384MB是怎么算出来的？

2. spark配置文件里指定了每个executor的内存为2G，为何日志和sparkUI上显示的是1060.3MB？

3. driver的内存配置为1G，为何sparkUI里显示的是530.3MB呢？

4. 为何yarn中每个container分配的内存是3G，而不是executor需要的2G呢？

问题解析
--
进过一番调研，发现这里有些概念容易混淆，整理如下，序号对应上面的问题：
<!--more-->
(1) spark的yarn-client向ResourceManager申请提交作业/启动AppMaster时，会判断是否是集群模式，如果是集群模式，则AppMaster的内存大小与driver内存大小一致，否则由spark.yarn.am.memory决定，这个参数的默认值是512MB。我们使用的是yarn-client模式，所以实际内存是512MB。

<font color='red'>384MB是spark-client为appMaster额外申请的内存</font>，计算方法如下：

![](https://raw.githubusercontent.com/maohong/picture/master/20150813/sparkonyarn-5.png)

即，默认从参数读取（集群模式从spark.yarn.driver.memoryOverhead参数读，否则从spark.yarn.am.memoryOverhead参数读），若没配此参数，则从AppMaster的内存*一定系数和默认最小overhead中取较大值。

在spark-1.4.1版本中，MEMORY_OVERHEAD_FACTOR的默认值为0.10（之前是0.07），MEMORY_OVERHEAD_MIN默认为384，我们没有指定spark.yarn.driver.memoryOverhead和spark.yarn.am.memoryOverhead，而amMemory=512M（由spark.yarn.am.memory决定），因此memoryOverhead为max(512*0.10, 384)=384MB。

Executor的memoryOverhead计算方法与此一样，只是不区分是否集群模式，都默认由spark.yarn.executor.memoryOverhead配置。

(2) <font color='red'>日志和sparkUI上显示的是executor内部用于缓存计算结果的内存空间，并不是executor所拥有的全部内存</font>。这部分内存是由以下公式计算：

![](https://raw.githubusercontent.com/maohong/picture/master/20150813/sparkonyarn-6.png)

Runtime.getRuntime.maxMemory按2048MB算，storage memory大小为1105.92MB，sparkUI显示的略小于此值，是正常的。

(3) 与上述第2点一样，storage memory的大小略小于1024*0.9*0.6=552.96MB

(4) 前面提到spark会为container额外申请一部分内存（memoryOverhead），因此，实际为container提交申请的内存大小是2048 + max(2048*0.10, 384) = 2432MB，而<font color='red'>yarn在做资源分配时会做资源规整化，即应用程序申请的资源量一定是最小可申请资源量的整数倍（向上取整）</font>，最小可申请内存量由yarn.scheduler.minimum-allocation-mb指定，因此，会为container分配3G内存。

验证
--

为了验证上述规则，继续修改配置参数：

> SPARK_EXECUTOR_INSTANCES=4          *在yarn集群中启动的executor进程数*
> 
> SPARK_EXECUTOR_MEMORY=4G            *为每个executor进程分配的内存大小*
> 
> SPARK_DRIVER_MEMORY=3G              *为spark-driver进程分配的内存大小*

并在启动spark-sql时指定spark.yarn.am.memory参数：

**bin/spark-sql –master yarn –conf spark.yarn.am.memory=1024m**

再看日志信息：

![](https://raw.githubusercontent.com/maohong/picture/master/20150813/sparkonyarn-7.png)

![](https://raw.githubusercontent.com/maohong/picture/master/20150813/sparkonyarn-8.png)

yarnUI状态：

![](https://raw.githubusercontent.com/maohong/picture/master/20150813/sparkonyarn-9.png)

sparkUI的executors信息：

![](https://raw.githubusercontent.com/maohong/picture/master/20150813/sparkonyarn-10.png)

可见，AppMaster的实际内存为1024M（1408-384），而其在yarn中的container内存大小为2G（1408大于1G，yarn按资源规整化原则为其分配2G）。

同理，driver的storage memory空间为3G\*0.9\*0.6=1.62G，executor的storage memory空间为4G\*0.9\*0.6=2.16G，executor所在container占用5G内存（4096+max(4096*0.10,384)= 4505.6，大于4G， yarn按资源规整化原则为其分配5G）。

Yarn集群的内存总占用空间为2+5*4=22G。