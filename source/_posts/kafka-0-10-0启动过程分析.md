---
title: kafka-0.10.0启动过程分析
date: 2016-07-08 22:46:18
tags:
- kafka
- 源码分析
categories:
- Kafka
---

kafka-0.10.0是官方出的最新稳定版本，提供了大量新的feature，具体可见[这里](http://www.iteblog.com/archives/1677)，本文主要分析kafka-0.10-0的源码结构和启动过程。

源码结构
--
kafka-0.10.0的源码可以从github上fork一份，在源码目录下执行./gradlew idea生成idea项目，然后导入idea即可。这中间需要使用gradle进行依赖包的下载，导入后可以看到其源码结构如下图所示：

![](https://raw.githubusercontent.com/maohong/picture/master/20160708-kafka0.10.0%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%E5%88%86%E6%9E%90/1.png)

包括几大重要模块：

* clients主要是kafka-client相关的代码，包括consumer、producer，还包括一些公共逻辑，如授权认证、序列化等。
* connect主要是kafka-connect模块的代码逻辑，Kafka connect是0.9版本增加的特性,支持创建和管理数据流管道。通过它可以将大数据从其它系统导入到Kafka中，也可以从Kafka中导出到其它系统，比如数据库、elastic search等。
* core模块是kafka的核心部分，主要包括broker的实现逻辑、producer和consumer的javaapi等。
* streams模块主要是kafka-streaming的实现，提供了一整套描述常见流操作的高级语言API（比如 joining, filtering以及aggregation等），我们可以基于此开发流处理应用程序。

<!--more-->

启动入口
--
kafka的启动入口在core_main这个module下，入口函数如下：

```scala
def main(args: Array[String]): Unit = {
    try {
      val serverProps = getPropsFromArgs(args)
      val kafkaServerStartable = KafkaServerStartable.fromProps(serverProps)

      // attach shutdown handler to catch control-c
      Runtime.getRuntime().addShutdownHook(new Thread() {
        override def run() = {
          kafkaServerStartable.shutdown
        }
      })

      kafkaServerStartable.startup
      kafkaServerStartable.awaitShutdown
    }
    catch {
      case e: Throwable =>
        fatal(e)
        System.exit(1)
    }
    System.exit(0)
  }
```

先从命令行指定的配置文件加载配置，然后通过KafkaServerStartable类启动broker，实际上在KafkaServerStartable中维护了一个KafkaServer对象，它通过调用KafkaServer的startup方法启动broker。

broker启动过程
--

下面并启动过程代码按启动顺序分两部分做说明。

第一部分主要是核心模块的启动，代码如下：

```scala
metrics = new Metrics(metricConfig, reporters, kafkaMetricsTime, true)

        brokerState.newState(Starting)

        /* start scheduler */
        kafkaScheduler.startup()

        /* setup zookeeper */
        zkUtils = initZk()

        /* start log manager */
        logManager = createLogManager(zkUtils.zkClient, brokerState)
        logManager.startup()

        /* generate brokerId */
        config.brokerId =  getBrokerId
        this.logIdent = "[Kafka Server " + config.brokerId + "], "

        socketServer = new SocketServer(config, metrics, kafkaMetricsTime)
        socketServer.startup()

        /* start replica manager */
        replicaManager = new ReplicaManager(config, metrics, time, kafkaMetricsTime, zkUtils, kafkaScheduler, logManager,
          isShuttingDown)
        replicaManager.startup()

        /* start kafka controller */
        kafkaController = new KafkaController(config, zkUtils, brokerState, kafkaMetricsTime, metrics, threadNamePrefix)
        kafkaController.startup()

        /* start group coordinator */
        groupCoordinator = GroupCoordinator(config, zkUtils, replicaManager, kafkaMetricsTime)
        groupCoordinator.startup()

        /* Get the authorizer and initialize it if one is specified.*/
        authorizer = Option(config.authorizerClassName).filter(_.nonEmpty).map { authorizerClassName =>
          val authZ = CoreUtils.createObject[Authorizer](authorizerClassName)
          authZ.configure(config.originals())
          authZ
        }

        /* start processing requests */
        apis = new KafkaApis(socketServer.requestChannel, replicaManager, groupCoordinator,
          kafkaController, zkUtils, config.brokerId, config, metadataCache, metrics, authorizer)
        requestHandlerPool = new KafkaRequestHandlerPool(config.brokerId, socketServer.requestChannel, apis, config.numIoThreads)
        brokerState.newState(RunningAsBroker)
```

1. 首先是初始化Metrics注册信息。
2. 接着把当前broker的状态先置为Starting。
3. 启动kafkaScheduler，其内部维护了一个ScheduledThreadPoolExecutor，用于执行broker内置的一些周期性运行的job或定时job。比如，启动自动提交时，broker会定期维护客户端的消费topic-partition的offset信息。
4. 初始化zookeeper访问工具，建立必要的数据路径。
5. 启动LogManager，也就是日志数据管理子系统，负责日志数据的创建、截断、滚动、和清理等。
6. 启动SocketServer，一个基于NIO的socker服务端，其线程模型是有一个acceptor线程来接受客户端的连接，对应这个acceptor有N个processor线程，每个processor有自己的selector来从sockets读取收到的请求。另外，有M个handler线程专门处理请求并把处理结果返回给processor线程并通过socket写回给客户端。
7. 启动ReplicaManager，也即副本管理器，用于管理每个topic-partition的副本状态，包括主从、ISR列表等。
8. 启动KafkaController，可以理解为kafka集群的中央控制器，负责全局的协调，比如选取leader，reassignment等，其自身也支持动态选举高可用。
9. 启动GroupCoordinator，主要用于broker组管理和offset管理。
10. 初始化授权认证管理器，用户可以自己通过参数authorizer.class.name指定具体的Authorizer实现。kafka自带有SimpleAclAuthorizer的简单实现。
11. 初始化KafkaApis，用于统一接收外部请求。
12. 初始化KafkaRequestHandlerPool，内部是一个线程池，用于具体处理外部请求。
13. 将当前broker的状态置为RunningAsBroker，这时，broker已经可以对外提供服务了。

第二部分主要是辅助模块的启动，代码如下：

```scala
        Mx4jLoader.maybeLoad()

        /* start dynamic config manager */
        dynamicConfigHandlers = Map[String, ConfigHandler](ConfigType.Topic -> new TopicConfigHandler(logManager, config),
                                                           ConfigType.Client -> new ClientIdConfigHandler(apis.quotaManagers))

        // Apply all existing client configs to the ClientIdConfigHandler to bootstrap the overrides
        // TODO: Move this logic to DynamicConfigManager
        AdminUtils.fetchAllEntityConfigs(zkUtils, ConfigType.Client).foreach {
          case (clientId, properties) => dynamicConfigHandlers(ConfigType.Client).processConfigChanges(clientId, properties)
        }

        // Create the config manager. start listening to notifications
        dynamicConfigManager = new DynamicConfigManager(zkUtils, dynamicConfigHandlers)
        dynamicConfigManager.startup()

        /* tell everyone we are alive */
        val listeners = config.advertisedListeners.map {case(protocol, endpoint) =>
          if (endpoint.port == 0)
            (protocol, EndPoint(endpoint.host, socketServer.boundPort(protocol), endpoint.protocolType))
          else
            (protocol, endpoint)
        }
        kafkaHealthcheck = new KafkaHealthcheck(config.brokerId, listeners, zkUtils, config.rack,
          config.interBrokerProtocolVersion)
        kafkaHealthcheck.startup()

        // Now that the broker id is successfully registered via KafkaHealthcheck, checkpoint it
        checkpointBrokerId(config.brokerId)

        /* register broker metrics */
        registerStats()

        shutdownLatch = new CountDownLatch(1)
        startupComplete.set(true)
        isStartingUp.set(false)
        AppInfoParser.registerAppInfo(jmxPrefix, config.brokerId.toString)
        info("started")
```

1. 启动jmx，通过参数kafka_mx4jenable控制是否启用jmx，默认为false。
2. 初始化TopicConfigHandler和ClientIdConfigHandler，前者用于处理zk上的topic配置变更信息，后者用于zk上的clientId配置变更信息。
3. 启动DynamicConfigManager，通过动态配置管理器，监听zk上的配置节点变化，并根据具体变化的配置信息调用TopicConfigHandler或ClientIdConfigHandler更新配置。
4. 启动KafkaHealthcheck，用于在zk上注册当前broker节点信息，以便节点退出时其他broker和consumer能监听到，目前的节点健康度判断比较简单，只是单纯的看zk上的节点是否存在。
5. 最后，在本地对当前broker做个checkpoint，并注册jmx bean信息






