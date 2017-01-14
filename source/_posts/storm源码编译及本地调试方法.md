---
title: storm源码编译及本地调试方法
date: 2016-07-13 23:53:12
tags: 
- storm
- 源码编译
- 本地调试
categories:
- Storm
---

基础环境
--

* IDE开发环境：intelliJIdea
* JDK1.7  64bit
* intelliJIdea安装maven插件，配置好仓库源
* intelliJIdea安装clojure插件Cursive（需要注册并获取一个license，否则只能使用30天）
* 如果需要自己创建clojure项目进行开发，需要安装leiningen，[下载地址](http://leiningen.org/)

源码获取
--

从github checkout代码到本地即可，https://github.com/apache/storm.git

我这里编译的是我们目前正在用的0.10.0版本的代码。


导入idea及编译
--

打开idea，新建project，从源码导入，如下：

![](https://raw.githubusercontent.com/maohong/picture/master/20160713-storm%E6%BA%90%E7%A0%81%E7%BC%96%E8%AF%91%E5%8F%8A%E8%B0%83%E8%AF%95/1.png)

导入后，idea会自动根据pom.xml下载相关依赖包，部分依赖包如果下载不到，需要手动添加。完成后，可以看到project的module如下图所示：

![](https://raw.githubusercontent.com/maohong/picture/master/20160713-storm%E6%BA%90%E7%A0%81%E7%BC%96%E8%AF%91%E5%8F%8A%E8%B0%83%E8%AF%95/2.png)

<!--more-->

这时候，通过idea就可以直接跟踪看源码了，但直接运行storm-starter中的例子还是会报错并提示有些类找不到，经查看是clojure的代码还未编译出class文件。可以在源码目录下执行mvn compile进行编译。

使用idea调试源码
--

编译完成后，可以直接启动storm-starter中的例子运行。期间可能出现找不到类，检查classpath，依赖包的scope由provided改为compile。

在源代码中加断点，run或者debug即可。

> 2739 [main] INFO  b.s.u.Utils - Using defaults.yaml from resources
> 4546 [main] INFO  b.s.u.Utils - Using defaults.yaml from resources
> 5218 [main] INFO  b.s.zookeeper - Starting inprocess zookeeper at port 2000 and dir /var/folders/c0/0bgvmbb10jz1609_1xjqdsj00000gn/T//eeb57be9-5478-4fa9-ab31-6dfce38e7695
> 5243 [main] INFO  b.s.u.Utils - Using defaults.yaml from resources
> 5340 [main] INFO  b.s.d.nimbus - Starting Nimbus with conf {"topology.builtin.metrics.bucket.size.secs" 60, ......
> 5342 [main] INFO  b.s.d.nimbus - Using default scheduler
> 5360 [main] INFO  b.s.u.StormBoundedExponentialBackoffRetry - The baseSleepTimeMs [1000] the maxSleepTimeMs [30000] the maxRetries [5]
> 5457 [main] INFO  o.a.c.f.i.CuratorFrameworkImpl - Starting
> 5529 [main-EventThread] INFO  o.a.c.f.s.ConnectionStateManager - State change: CONNECTED
> 5531 [main-EventThread] INFO  b.s.zookeeper - Zookeeper state update: :connected:none
> 6569 [main] INFO  b.s.u.StormBoundedExponentialBackoffRetry - The baseSleepTimeMs [1000] the maxSleepTimeMs [30000] the maxRetries [5]
> 6569 [main] INFO  o.a.c.f.i.CuratorFrameworkImpl - Starting
> 6574 [main-EventThread] INFO  o.a.c.f.s.ConnectionStateManager - State change: CONNECTED
> 6605 [main] INFO  b.s.u.StormBoundedExponentialBackoffRetry - The baseSleepTimeMs [1000] the maxSleepTimeMs [30000] the maxRetries [5]
> 6605 [main] INFO  o.a.c.f.i.CuratorFrameworkImpl - Starting
> 6609 [main-EventThread] INFO  o.a.c.f.s.ConnectionStateManager - State change: CONNECTED
> 6609 [main-EventThread] INFO  b.s.zookeeper - Zookeeper state update: :connected:none
> 6617 [main] INFO  b.s.u.StormBoundedExponentialBackoffRetry - The baseSleepTimeMs [1000] the maxSleepTimeMs [30000] the maxRetries [5]
> 6618 [main] INFO  o.a.c.f.i.CuratorFrameworkImpl - Starting
> 6620 [main] INFO  b.s.u.StormBoundedExponentialBackoffRetry - The baseSleepTimeMs [1000] the maxSleepTimeMs [30000] the maxRetries [5]
> 6621 [main] INFO  o.a.c.f.i.CuratorFrameworkImpl - Starting
> 6623 [main-EventThread] INFO  o.a.c.f.s.ConnectionStateManager - State change: CONNECTED
> 6625 [main-EventThread] INFO  o.a.c.f.s.ConnectionStateManager - State change: CONNECTED
> 6649 [main-EventThread] INFO  b.s.zookeeper - Zookeeper state update: :connected:none
> 6652 [main] INFO  b.s.u.StormBoundedExponentialBackoffRetry - The baseSleepTimeMs [1000] the maxSleepTimeMs [30000] the maxRetries [5]
> 6653 [main] INFO  o.a.c.f.i.CuratorFrameworkImpl - Starting
> 6657 [main-EventThread] INFO  o.a.c.f.s.ConnectionStateManager - State change: CONNECTED
> 6671 [main] INFO  b.s.d.supervisor - Starting Supervisor with conf {"topology.builtin.metrics.bucket.size.secs" 60, ......
> 6693 [main] INFO  b.s.u.StormBoundedExponentialBackoffRetry - The baseSleepTimeMs [1000] the maxSleepTimeMs [30000] the maxRetries [5]
> 6694 [main] INFO  o.a.c.f.i.CuratorFrameworkImpl - Starting
> 6697 [main-EventThread] INFO  o.a.c.f.s.ConnectionStateManager - State change: CONNECTED
> 6697 [main-EventThread] INFO  b.s.zookeeper - Zookeeper state update: :connected:none
> 6700 [main] INFO  b.s.u.StormBoundedExponentialBackoffRetry - The baseSleepTimeMs [1000] the maxSleepTimeMs [30000] the maxRetries [5]
> 6701 [main] INFO  o.a.c.f.i.CuratorFrameworkImpl - Starting
> 6704 [main-EventThread] INFO  o.a.c.f.s.ConnectionStateManager - State change: CONNECTED
> 6722 [main] INFO  b.s.d.supervisor - Starting supervisor with id 913c90f6-3f78-4646-8998-aa901ae3c360 at host localhost
> 6725 [main] INFO  b.s.d.supervisor - Starting Supervisor with conf {"topology.builtin.metrics.bucket.size.secs" 60, .....
> 6732 [main] INFO  b.s.u.StormBoundedExponentialBackoffRetry - The baseSleepTimeMs [1000] the maxSleepTimeMs [30000] the maxRetries [5]
> 6732 [main] INFO  o.a.c.f.i.CuratorFrameworkImpl - Starting
> 6736 [main-EventThread] INFO  o.a.c.f.s.ConnectionStateManager - State change: CONNECTED
> 6736 [main-EventThread] INFO  b.s.zookeeper - Zookeeper state update: :connected:none
> 6740 [main] INFO  b.s.u.StormBoundedExponentialBackoffRetry - The baseSleepTimeMs [1000] the maxSleepTimeMs [30000] the maxRetries [5]
> 6741 [main] INFO  o.a.c.f.i.CuratorFrameworkImpl - Starting
> 6744 [main-EventThread] INFO  o.a.c.f.s.ConnectionStateManager - State change: CONNECTED
> 6753 [main] INFO  b.s.d.supervisor - Starting supervisor with id 49c35a73-7500-4ea4-aaa2-4b1c1f231fd4 at host localhost
> 7035 [main] INFO  b.s.d.nimbus - [req 1] Access from:  principal: op:submitTopology
> 7113 [main] INFO  b.s.d.nimbus - Received topology submission for wordCounter with conf {"topology.max.task.parallelism" nil, "topology.submitter.principal" "", "topology.acker.executors" nil, "topology.max.spout.pending" 20, "storm.zookeeper.superACL" nil, "topology.users" (), "topology.submitter.user" "", "topology.kryo.register" {"storm.trident.topology.TransactionAttempt" nil, "storm.trident.spout.RichSpoutBatchId" "storm.trident.spout.RichSpoutBatchIdSerializer"}, "topology.kryo.decorators" (), "storm.id" "wordCounter-1-1468420782", "topology.name" "wordCounter"}
> 7123 [main] INFO  b.s.d.nimbus - nimbus file location:/var/folders/c0/0bgvmbb10jz1609_1xjqdsj00000gn/T//333ed6da-9ef5-4781-bd82-4f315facd4a8/nimbus/stormdist/wordCounter-1-1468420782
> 7152 [main] INFO  b.s.d.nimbus - Activating wordCounter: wordCounter-1-1468420782
> 7346 [main] INFO  b.s.s.EvenScheduler - Available slots: (["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1028] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1029] ["913c90f6-3f78-4646-8998-aa901ae3c360" 1024] ["913c90f6-3f78-4646-8998-aa901ae3c360" 1025] ["913c90f6-3f78-4646-8998-aa901ae3c360" 1026])
> 7398 [main] INFO  b.s.d.nimbus - Setting new assignment for topology id wordCounter-1-1468420782: #backtype.storm.daemon.common.Assignment{:master-code-dir "/var/folders/c0/0bgvmbb10jz1609_1xjqdsj00000gn/T//333ed6da-9ef5-4781-bd82-4f315facd4a8/nimbus/stormdist/wordCounter-1-1468420782", :node->host {"49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" "localhost"}, :executor->node+port {[8 8] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [12 12] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [2 2] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [7 7] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [22 22] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [3 3] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [24 24] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [1 1] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [18 18] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [6 6] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [20 20] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [9 9] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [23 23] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [11 11] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [16 16] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [13 13] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [19 19] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [21 21] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [5 5] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [26 26] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [10 10] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [14 14] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [4 4] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [15 15] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [25 25] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027], [17 17] ["49c35a73-7500-4ea4-aaa2-4b1c1f231fd4" 1027]}, :executor->start-time-secs {[8 8] 1468420782, [12 12] 1468420782, [2 2] 1468420782, [7 7] 1468420782, [22 22] 1468420782, [3 3] 1468420782, [24 24] 1468420782, [1 1] 1468420782, [18 18] 1468420782, [6 6] 1468420782, [20 20] 1468420782, [9 9] 1468420782, [23 23] 1468420782, [11 11] 1468420782, [16 16] 1468420782, [13 13] 1468420782, [19 19] 1468420782, [21 21] 1468420782, [5 5] 1468420782, [26 26] 1468420782, [10 10] 1468420782, [14 14] 1468420782, [4 4] 1468420782, [15 15] 1468420782, [25 25] 1468420782, [17 17] 1468420782}}
> 7751 [Thread-7] INFO  b.s.d.supervisor - Extracting resources from jar at /Library/Java/JavaVirtualMachines/jdk1.7.0_79.jdk/Contents/Home/lib/ant-javafx.jar to /var/folders/c0/0bgvmbb10jz1609_1xjqdsj00000gn/T//29645b09-90e9-4b9a-a657-60c418f92841/supervisor/stormdist/wordCounter-1-1468420782/resources
> 7788 [Thread-8] INFO  b.s.d.supervisor - Launching worker with assignment {:storm-id "wordCounter-1-1468420782", :executors [[8 8] [12 12] [2 2] [7 7] [22 22] [3 3] [24 24] [1 1] [18 18] [6 6] [20 20] [9 9] [23 23] [11 11] [16 16] [13 13] [19 19] [21 21] [5 5] [26 26] [10 10] [14 14] [4 4] [15 15] [25 25] [17 17]]} for this supervisor 49c35a73-7500-4ea4-aaa2-4b1c1f231fd4 on port 1027 with id 9dd8aeac-1cd6-467a-a84c-2637d0825d99
> 7791 [Thread-8] INFO  b.s.d.worker - Launching worker for wordCounter-1-1468420782 on 49c35a73-7500-4ea4-aaa2-4b1c1f231fd4:1027 with id 9dd8aeac-1cd6-467a-a84c-2637d0825d99 and conf {"topology.builtin.metrics.bucket.size.secs" 60, ......
> 7793 [Thread-8] INFO  b.s.u.StormBoundedExponentialBackoffRetry - The baseSleepTimeMs [1000] the maxSleepTimeMs [30000] the maxRetries [5]
> 7794 [Thread-8] INFO  o.a.c.f.i.CuratorFrameworkImpl - Starting
> 7798 [Thread-8-EventThread] INFO  o.a.c.f.s.ConnectionStateManager - State change: CONNECTED
> 7798 [Thread-8-EventThread] INFO  b.s.zookeeper - Zookeeper state update: :connected:none
> 7801 [Thread-8] INFO  b.s.u.StormBoundedExponentialBackoffRetry - The baseSleepTimeMs [1000] the maxSleepTimeMs [30000] the maxRetries [5]
> 7802 [Thread-8] INFO  o.a.c.f.i.CuratorFrameworkImpl - Starting
> 7805 [Thread-8-EventThread] INFO  o.a.c.f.s.ConnectionStateManager - State change: CONNECTED
> 7809 [Thread-8] INFO  b.s.s.a.AuthUtils - Got AutoCreds []
> 7811 [Thread-8] INFO  b.s.d.worker - Reading Assignments.
> 7881 [Thread-8] INFO  b.s.d.worker - Launching receive-thread for 49c35a73-7500-4ea4-aaa2-4b1c1f231fd4:1027
> 7884 [Thread-9-worker-receiver-thread-0] INFO  b.s.m.loader - Starting receive-thread: [stormId: wordCounter-1-1468420782, port: 1027, thread-id: 0 ]
> 8261 [Thread-8] INFO  b.s.d.executor - Loading executor b-2:[8 8]
> 8285 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks b-2:[8 8]
> 8300 [Thread-8] INFO  b.s.d.executor - Finished loading executor b-2:[8 8]
> 8311 [Thread-8] INFO  b.s.d.executor - Loading executor b-2:[12 12]
> 8329 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks b-2:[12 12]
> 8331 [Thread-8] INFO  b.s.d.executor - Finished loading executor b-2:[12 12]
> 8340 [Thread-8] INFO  b.s.d.executor - Loading executor $spoutcoord-spout0:[2 2]
> 8343 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks $spoutcoord-spout0:[2 2]
> 8346 [Thread-8] INFO  b.s.d.executor - Finished loading executor $spoutcoord-spout0:[2 2]
> 8355 [Thread-8] INFO  b.s.d.executor - Loading executor b-2:[7 7]
> 8372 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks b-2:[7 7]
> 8375 [Thread-8] INFO  b.s.d.executor - Finished loading executor b-2:[7 7]
> 8381 [Thread-8] INFO  b.s.d.executor - Loading executor b-3:[22 22]
> 8401 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks b-3:[22 22]
> 8404 [Thread-8] INFO  b.s.d.executor - Finished loading executor b-3:[22 22]
> 8412 [Thread-8] INFO  b.s.d.executor - Loading executor __acker:[3 3]
> 8414 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks __acker:[3 3]
> 8424 [Thread-8] INFO  b.s.d.executor - Timeouts disabled for executor __acker:[3 3]
> 8425 [Thread-8] INFO  b.s.d.executor - Finished loading executor __acker:[3 3]
> 8443 [Thread-8] INFO  b.s.d.executor - Loading executor b-5:[24 24]
> 8465 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks b-5:[24 24]
> 8467 [Thread-8] INFO  b.s.d.executor - Finished loading executor b-5:[24 24]
> 8530 [Thread-8] INFO  b.s.d.executor - Loading executor $mastercoord-bg0:[1 1]
> 8539 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks $mastercoord-bg0:[1 1]
> 8576 [Thread-8] INFO  b.s.d.executor - Finished loading executor $mastercoord-bg0:[1 1]
> 8603 [Thread-8] INFO  b.s.d.executor - Loading executor b-2:[18 18]
> 8633 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks b-2:[18 18]
> 8635 [Thread-8] INFO  b.s.d.executor - Finished loading executor b-2:[18 18]
> 8646 [Thread-8] INFO  b.s.d.executor - Loading executor b-2:[6 6]
> 8681 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks b-2:[6 6]
> 8683 [Thread-8] INFO  b.s.d.executor - Finished loading executor b-2:[6 6]
> 8719 [Thread-8] INFO  b.s.d.executor - Loading executor b-2:[20 20]
> 8757 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks b-2:[20 20]
> 8763 [Thread-8] INFO  b.s.d.executor - Finished loading executor b-2:[20 20]
> 8782 [Thread-8] INFO  b.s.d.executor - Loading executor b-2:[9 9]
> 8808 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks b-2:[9 9]
> 8818 [Thread-8] INFO  b.s.d.executor - Finished loading executor b-2:[9 9]
> 8828 [Thread-8] INFO  b.s.d.executor - Loading executor b-4:[23 23]
> 8847 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks b-4:[23 23]
> 8851 [Thread-8] INFO  b.s.d.executor - Finished loading executor b-4:[23 23]
> 8858 [refresh-active-timer] INFO  b.s.d.worker - All connections are ready for worker 49c35a73-7500-4ea4-aaa2-4b1c1f231fd4:1027 with id 9dd8aeac-1cd6-467a-a84c-2637d0825d99
> 8864 [Thread-8] INFO  b.s.d.executor - Loading executor b-2:[11 11]
> 8877 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks b-2:[11 11]
> 8879 [Thread-8] INFO  b.s.d.executor - Finished loading executor b-2:[11 11]
> 8886 [Thread-8] INFO  b.s.d.executor - Loading executor __system:[-1 -1]
> 8887 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks __system:[-1 -1]
> 8890 [Thread-8] INFO  b.s.d.executor - Finished loading executor __system:[-1 -1]
> 8914 [Thread-8] INFO  b.s.d.executor - Loading executor b-2:[16 16]
> 9052 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks b-2:[16 16]
> 9055 [Thread-8] INFO  b.s.d.executor - Finished loading executor b-2:[16 16]
> 9070 [Thread-8] INFO  b.s.d.executor - Loading executor b-2:[13 13]
> 9081 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks b-2:[13 13]
> 9089 [Thread-8] INFO  b.s.d.executor - Finished loading executor b-2:[13 13]
> 9116 [Thread-8] INFO  b.s.d.executor - Loading executor b-2:[19 19]
> 9129 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks b-2:[19 19]
> 9132 [Thread-8] INFO  b.s.d.executor - Finished loading executor b-2:[19 19]
> 9148 [Thread-8] INFO  b.s.d.executor - Loading executor b-2:[21 21]
> 9160 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks b-2:[21 21]
> 9163 [Thread-8] INFO  b.s.d.executor - Finished loading executor b-2:[21 21]
> 9178 [Thread-8] INFO  b.s.d.executor - Loading executor b-1:[5 5]
> 9192 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks b-1:[5 5]
> 9194 [Thread-8] INFO  b.s.d.executor - Finished loading executor b-1:[5 5]
> 9204 [Thread-8] INFO  b.s.d.executor - Loading executor spout1:[26 26]
> 9205 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks spout1:[26 26]
> 9208 [Thread-8] INFO  b.s.d.executor - Finished loading executor spout1:[26 26]
> 9220 [Thread-8] INFO  b.s.d.executor - Loading executor b-2:[10 10]
> 9226 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks b-2:[10 10]
> 9228 [Thread-8] INFO  b.s.d.executor - Finished loading executor b-2:[10 10]
> 9234 [Thread-8] INFO  b.s.d.executor - Loading executor b-2:[14 14]
> 9237 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks b-2:[14 14]
> 9239 [Thread-8] INFO  b.s.d.executor - Finished loading executor b-2:[14 14]
> 9244 [Thread-8] INFO  b.s.d.executor - Loading executor b-0:[4 4]
> 9248 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks b-0:[4 4]
> 9249 [Thread-8] INFO  b.s.d.executor - Finished loading executor b-0:[4 4]
> 9255 [Thread-8] INFO  b.s.d.executor - Loading executor b-2:[15 15]
> 9260 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks b-2:[15 15]
> 9261 [Thread-8] INFO  b.s.d.executor - Finished loading executor b-2:[15 15]
> 9273 [Thread-8] INFO  b.s.d.executor - Loading executor spout0:[25 25]
> 9275 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks spout0:[25 25]
> 9277 [Thread-8] INFO  b.s.d.executor - Finished loading executor spout0:[25 25]
> 9284 [Thread-8] INFO  b.s.d.executor - Loading executor b-2:[17 17]
> 9289 [Thread-8] INFO  b.s.d.executor - Loaded executor tasks b-2:[17 17]
> 9291 [Thread-8] INFO  b.s.d.executor - Finished loading executor b-2:[17 17]
> 9298 [Thread-8] INFO  b.s.d.worker - Worker has topology config {"topology.builtin.metrics.bucket.size.secs" 60, ......
> 9298 [Thread-8] INFO  b.s.d.worker - Worker 9dd8aeac-1cd6-467a-a84c-2637d0825d99 for storm wordCounter-1-1468420782 on 49c35a73-7500-4ea4-aaa2-4b1c1f231fd4:1027 has finished loading
> 9298 [Thread-8] INFO  b.s.config - SET worker-user 9dd8aeac-1cd6-467a-a84c-2637d0825d99 
> 9875 [Thread-27-b-2] INFO  b.s.d.executor - Preparing bolt b-2:(18)
> 9882 [Thread-35-b-4] INFO  b.s.d.executor - Preparing bolt b-4:(23)
> 9882 [Thread-41-b-2] INFO  b.s.d.executor - Preparing bolt b-2:(16)
> 9883 [Thread-13-b-2] INFO  b.s.d.executor - Preparing bolt b-2:(12)
> 9883 [Thread-59-b-2] INFO  b.s.d.executor - Preparing bolt b-2:(15)
> 9883 [Thread-47-b-2] INFO  b.s.d.executor - Preparing bolt b-2:(21)
> 9893 [Thread-35-b-4] INFO  b.s.d.executor - Prepared bolt b-4:(23)
> 9896 [Thread-47-b-2] INFO  b.s.d.executor - Prepared bolt b-2:(21)
> 9896 [Thread-59-b-2] INFO  b.s.d.executor - Prepared bolt b-2:(15)
> 9896 [Thread-27-b-2] INFO  b.s.d.executor - Prepared bolt b-2:(18)
> 9896 [Thread-13-b-2] INFO  b.s.d.executor - Prepared bolt b-2:(12)
> 9896 [Thread-41-b-2] INFO  b.s.d.executor - Prepared bolt b-2:(16)
> 9898 [Thread-31-b-2] INFO  b.s.d.executor - Preparing bolt b-2:(20)
> 9898 [Thread-15-$spoutcoord-spout0] INFO  b.s.d.executor - Preparing bolt $spoutcoord-spout0:(2)
> 9899 [Thread-61-spout0] INFO  b.s.d.executor - Preparing bolt spout0:(25)
> 9900 [Thread-15-$spoutcoord-spout0] INFO  b.s.u.StormBoundedExponentialBackoffRetry - The baseSleepTimeMs [1000] the maxSleepTimeMs [30000] the maxRetries [5]
> 9900 [Thread-61-spout0] INFO  b.s.d.executor - Prepared bolt spout0:(25)
> 9901 [Thread-15-$spoutcoord-spout0] INFO  o.a.c.f.i.CuratorFrameworkImpl - Starting
> 9901 [Thread-31-b-2] INFO  b.s.d.executor - Prepared bolt b-2:(20)
> 9907 [Thread-15-$spoutcoord-spout0-EventThread] INFO  o.a.c.f.s.ConnectionStateManager - State change: CONNECTED
> 9908 [Thread-43-b-2] INFO  b.s.d.executor - Preparing bolt b-2:(13)
> 9908 [Thread-37-b-2] INFO  b.s.d.executor - Preparing bolt b-2:(11)
> 9908 [Thread-63-b-2] INFO  b.s.d.executor - Preparing bolt b-2:(17)
> 9910 [Thread-43-b-2] INFO  b.s.d.executor - Prepared bolt b-2:(13)
> 9910 [Thread-37-b-2] INFO  b.s.d.executor - Prepared bolt b-2:(11)
> 9911 [Thread-63-b-2] INFO  b.s.d.executor - Prepared bolt b-2:(17)
> 9918 [Thread-49-b-1] INFO  b.s.d.executor - Preparing bolt b-1:(5)
> 9918 [Thread-39-__system] INFO  b.s.d.executor - Preparing bolt __system:(-1)
> 9918 [Thread-29-b-2] INFO  b.s.d.executor - Preparing bolt b-2:(6)
> 9920 [Thread-49-b-1] INFO  b.s.d.executor - Prepared bolt b-1:(5)
> 9920 [Thread-29-b-2] INFO  b.s.d.executor - Prepared bolt b-2:(6)
> 9921 [Thread-15-$spoutcoord-spout0] INFO  b.s.u.StormBoundedExponentialBackoffRetry - The baseSleepTimeMs [1000] the maxSleepTimeMs [30000] the maxRetries [5]
> 9922 [Thread-15-$spoutcoord-spout0] INFO  o.a.c.f.i.CuratorFrameworkImpl - Starting
> 9924 [Thread-39-__system] INFO  b.s.d.executor - Prepared bolt __system:(-1)
> 9929 [Thread-51-spout1] INFO  b.s.d.executor - Opening spout spout1:(26)
> 9929 [Thread-25-$mastercoord-bg0] INFO  b.s.d.executor - Opening spout $mastercoord-bg0:(1)
> 9929 [Thread-15-$spoutcoord-spout0-EventThread] INFO  o.a.c.f.s.ConnectionStateManager - State change: CONNECTED
> 9938 [Thread-51-spout1] INFO  b.s.d.executor - Opened spout spout1:(26)
> 9937 [Thread-25-$mastercoord-bg0] INFO  b.s.u.StormBoundedExponentialBackoffRetry - The baseSleepTimeMs [1000] the maxSleepTimeMs [30000] the maxRetries [5]
> 9940 [Thread-25-$mastercoord-bg0] INFO  o.a.c.f.i.CuratorFrameworkImpl - Starting
> 9940 [Thread-33-b-2] INFO  b.s.d.executor - Preparing bolt b-2:(9)
> 9942 [Thread-51-spout1] INFO  b.s.d.executor - Activating spout spout1:(26)
> 9942 [Thread-33-b-2] INFO  b.s.d.executor - Prepared bolt b-2:(9)
> 9947 [Thread-53-b-2] INFO  b.s.d.executor - Preparing bolt b-2:(10)
> 9950 [Thread-53-b-2] INFO  b.s.d.executor - Prepared bolt b-2:(10)
> 9956 [Thread-11-b-2] INFO  b.s.d.executor - Preparing bolt b-2:(8)
> 9956 [Thread-45-b-2] INFO  b.s.d.executor - Preparing bolt b-2:(19)
> 9957 [Thread-23-b-5] INFO  b.s.d.executor - Preparing bolt b-5:(24)
> 9958 [Thread-23-b-5] INFO  b.s.d.executor - Prepared bolt b-5:(24)
> 9958 [Thread-11-b-2] INFO  b.s.d.executor - Prepared bolt b-2:(8)
> 9958 [Thread-17-b-2] INFO  b.s.d.executor - Preparing bolt b-2:(7)
> 9959 [Thread-55-b-2] INFO  b.s.d.executor - Preparing bolt b-2:(14)
> 9959 [Thread-19-b-3] INFO  b.s.d.executor - Preparing bolt b-3:(22)
> 9960 [Thread-19-b-3] INFO  b.s.d.executor - Prepared bolt b-3:(22)
> 9960 [Thread-25-$mastercoord-bg0-EventThread] INFO  o.a.c.f.s.ConnectionStateManager - State change: CONNECTED
> 9960 [Thread-17-b-2] INFO  b.s.d.executor - Prepared bolt b-2:(7)
> 9962 [Thread-45-b-2] INFO  b.s.d.executor - Prepared bolt b-2:(19)
> 9963 [Thread-55-b-2] INFO  b.s.d.executor - Prepared bolt b-2:(14)
> 9964 [Thread-57-b-0] INFO  b.s.d.executor - Preparing bolt b-0:(4)
> 9964 [Thread-21-__acker] INFO  b.s.d.executor - Preparing bolt __acker:(3)
> 9965 [Thread-57-b-0] INFO  b.s.d.executor - Prepared bolt b-0:(4)
> 9966 [Thread-21-__acker] INFO  b.s.d.executor - Prepared bolt __acker:(3)
> 9969 [Thread-15-$spoutcoord-spout0] INFO  b.s.d.executor - Prepared bolt $spoutcoord-spout0:(2)
> 9971 [Thread-25-$mastercoord-bg0] INFO  b.s.u.StormBoundedExponentialBackoffRetry - The baseSleepTimeMs [1000] the maxSleepTimeMs [30000] the maxRetries [5]
> 9972 [Thread-25-$mastercoord-bg0] INFO  o.a.c.f.i.CuratorFrameworkImpl - Starting
> 9984 [Thread-25-$mastercoord-bg0-EventThread] INFO  o.a.c.f.s.ConnectionStateManager - State change: CONNECTED
> DRPC RESULT: [[0]]
> 9988 [Thread-25-$mastercoord-bg0] INFO  b.s.d.executor - Opened spout $mastercoord-bg0:(1)
> 9988 [Thread-25-$mastercoord-bg0] INFO  b.s.d.executor - Activating spout $mastercoord-bg0:(1)
> DRPC RESULT: [[60]]
> DRPC RESULT: [[120]]
> DRPC RESULT: [[179]]
> DRPC RESULT: [[239]]
> DRPC RESULT: [[299]]
> DRPC RESULT: [[359]]
> DRPC RESULT: [[414]]
> DRPC RESULT: [[474]]
> DRPC RESULT: [[534]]
> DRPC RESULT: [[593]]
> DRPC RESULT: [[653]]
> DRPC RESULT: [[713]]
> DRPC RESULT: [[768]]
> 
> Process finished with exit code 130



