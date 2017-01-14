---
title: redis服务端连接断开问题诊断
date: 2014-06-01 21:29:28
tags:
- redis
- 连接断开
categories: 
- Redis
---

问题现象
--

前段时间，由于线上redis服务器的内存使用率达到了机器总内存的50%以上，导致内存数据的dump持久化一直失败。扩展到多台redis后，应用系统访问redis时，在业务量较少时，时不时会出现以下异常，当业务量较大，redis访问频率很高时，却不会发生这个异常，一时觉得很诡异。

> redis.clients.jedis.exceptions.JedisConnectionException: It seems like server has closed the connection.
> at redis.clients.util.RedisInputStream.readLine(RedisInputStream.java:90) ~[jedis-2.1.0.jar:na]
> at redis.clients.jedis.Protocol.processInteger(Protocol.java:110) ~[jedis-2.1.0.jar:na]
> at redis.clients.jedis.Protocol.process(Protocol.java:70) ~[jedis-2.1.0.jar:na]
> at redis.clients.jedis.Protocol.read(Protocol.java:131) ~[jedis-2.1.0.jar:na]
> at redis.clients.jedis.Connection.getIntegerReply(Connection.java:188) ~[jedis-2.1.0.jar:na]
> at redis.clients.jedis.Jedis.sismember(Jedis.java:1266) ~[jedis-2.1.0.jar:na]

看提示，应该是服务端主动关闭了连接。查看了新上线的redis服务器的配置，有这么一项：

> \# Close the connection after a client is idle for N seconds (0 to disable)
> timeout 120

这项配置指的是客户端连接空闲超过多少秒后，服务端主动关闭连接，默认值0表示服务端永远不主动关闭。而op人员把服务器端的超时时间设置为了120秒。

这就解释了发生这个异常的原因。客户端使用了一个连接池管理访问redis的所有连接，这些连接是长连接，当业务量较小时，客户端部分连接使用率较低，当两次使用之间的间隔超过120秒时，redis服务端就主动关闭了这个连接，而等客户端下次再使用这个连接对象时，发现服务端已经关闭了连接，进而报错。

于是，再查看访问redis的系统（客户端）的配置：

![](https://raw.githubusercontent.com/maohong/picture/master/20140601-redis%E8%BF%9E%E6%8E%A5%E9%97%AE%E9%A2%98%E6%8E%92%E6%9F%A5/1.png)

客户端使用的是jedis内置的连接池，看其源码本质上是基于apache commons-pool实现的，其中有一个eviction线程，用于回收idle对象，对于redis连接池来说，也就是回收空闲连接。

JedisPoolConfig类继承自GenericObjectPoolConfig并覆盖了几项关于eviction线程的配置，具体如下：

![](https://raw.githubusercontent.com/maohong/picture/master/20140601-redis%E8%BF%9E%E6%8E%A5%E9%97%AE%E9%A2%98%E6%8E%92%E6%9F%A5/2.png)

*<font color=red>_timeBetweenEvictionRunsMillis</font>*：eviction线程的运行周期。默认是-1，表示不启动eviction线程。这里设置为30秒。

*<font color=red>_minEvictableIdleTimeMillis</font>*：对象处于idle状态的最长时间，默认是30分钟，这里设置为60秒。

通过客户端的默认配置看，对象的最大空闲时长是小于服务端的配置的，应该不是配置上的问题了。

于是，继续看是不是客户端代码使用上的问题。追踪到客户端代码如下：

![](https://raw.githubusercontent.com/maohong/picture/master/20140601-redis%E8%BF%9E%E6%8E%A5%E9%97%AE%E9%A2%98%E6%8E%92%E6%9F%A5/3.png)

可见，客户端首先尝试从本线程的ThreadLocal对象中获取jedis对象，若获取不到，再从masterJedisPool中取得jedis对象并放入ThreadLocal对象以便下次使用，并且jedis对象使用完毕后，没有从ThreadLocal中清除，也没有returnResource给masterJedisPool。

因此，问题产生的原因就在于此。ThreadLocal中的这个jedis对象被取出后没有return，对于对象池来说是处于非idle状态，因此不会被对象池evict。<font color=red>当业务量大时，这个jedis会被频繁使用，服务端认为这个jedis对应的连接是非空闲的，或者空闲时间达不到120秒，不会主动关闭，所以没什么问题。然而当业务量小时，这个jedis使用频率很低，当两次之间的使用间隔超出120秒时，服务端会主动把这个jedis的连接关闭，第二次调用时，就会出现上面的报错。</font>

从代码开发者的角度来说，这么做的目的是避免频繁从pool中获取jedis对象和return jedis对象以提高性能。

解决方案有两个：

1. 在redis-cli下在线修改redis 的配置，把timeout改回为0，无需重启redis即可直接生效，但redis若重启，配置会恢复。

2. 修改客户端代码，使用完jedis对象后，从ThreadLocal中清除，再返回给连接池。

出于改动成本考虑，先采用了第一种方案，在线修改redis配置后，报错不再出现。