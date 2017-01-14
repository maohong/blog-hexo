---
title: 使用zookeeper协调多服务器的任务处理
date: 2012-11-13 16:16:43
tags: 
- zookeeper
- 横向扩展
- 分布式协调
categories: 
- 分布式应用
---

背景
--
**Zookeeper**是hadoop的子项目，是google的chubby的开源实现，是一个针对大规模分布式系统的可靠的分布式协调系统。Zookeeper一般部署在一个集群上，通过在集群间维护一个数据树，使得连接到集群的client能够获得统一的数据信息，比如系统公共配置信息、节点存活状态等等。因此，在互联网公司中，zookeeper被广泛运用于统一配置管理、名字服务、分布式同步等。
问题
--
我们看下这样一种场景：  
前台系统每时每刻都生成大量数据，这些原生数据由后台系统处理完毕后再作他用，我们暂且不谈这些数据的存储形式，只关注如何能够尽可能高效的处理。举个例子，前台系统可能是微博的前端发布系统、搜索引擎上的广告投放系统，或者是任务发布系统，后台系统则可能是对微博和广告信息的审查系统，比如用户发的微博如果包含近期敏感信息则不予显示，若是任务，后台系统则负责处理任务具体的执行。  
若数据量和任务量较小，单节点的后台系统或许可以处理得过来，但是如果数据量和任务量很大（比如新浪微博，龙年正月初一0点0分0秒，共有32312条微博同时发布），单节点的后台系统肯定吃不消，这时候，可想而知的是多节点同时处理前台过来的数据。  
最简单的方法是，按消息id对后台节点数取模（msgid%server_num=mod），每个后台节点取自己那份数据进行处理，这就需要每个节点都知晓当前有多少个后台节点以及本节点所应取的mod数。但是，当某个节点宕机时，这个节点所应处理的数据无法被继续处理了，势必会造成阻塞，除非重新配置各节点上的参数，将节点数server_num减1，并修改各节点取数据的mod数。  
毋庸置疑，这样非常麻烦！如果能够将这种配置信息（实际上是数据在节点间分配的控制信息）统一管理起来，在配置信息发生变化时，各个后台节点能够及时知晓其变化，就可以避免上述情况的发生。  
因此，采用多节点处理数据时，有两个问题：  
1.避免多个节点重复处理同一条数据，否则造成资源浪费。  
2.不能有数据被遗漏处理，尤其是在有后台节点down掉的时候。  
也就是说，采用多节点同时处理数据时，需要将数据隔离开，分别给不同的节点处理，而且在有节点宕机的情况下，所有数据也必须可以无误的被其他可用节点处理。如何做到这一点呢，使用zookeeper吧！  
<!--more-->
解决方案
--
我们通过zookeeper维护一个目录（比如/app/config），服务器启动时连接zookeeper集群并在该目录下创建表示自己的临时节点（CreateMode.EPHEMERAL），相当于注册一个节点，节点名可以是本服务器的ip，节点的值为该服务器的mod值，按注册顺序从0递增，即第一个注册的节点值为0，第二个为1，依次下去，因此/app/config的子节点数就是注册到zookeeper的服务器数。同时，各服务器监听/app/config目录，当其发生变化（新加入子节点、子节点失效等）时，每个服务器都将获取到这个事件并进行相应的处理。  
demo
--
下面针对以上场景给出一个示例demo。  
**Server类**：服务器  
**ClientThread类**：服务器上的单个线程  
**NodeStateWatcher类**：服务器监听zookeeper集群的监听器  
**ZkOperationImpl类**：zookeeper的操作封装（实现ZkOperation接口）  
Server.java
```java
public class Server extends Thread
{
 private ClientThread[] clients = new ClientThread[Constant.THREAD_COUNT]; // 数据处理线程
 private ZkOperation operationCient = null; // 与zookeeper的连接
 private Watcher nodeWatcher = null;  // 向zookeeper注册的监听器
 private String name; // 服务器名
 private String ip; // 服务器ip
 
 public Server(String name, String ip) throws IOException, KeeperException, InterruptedException
 {
  this.name = name;
  this.ip = ip;
  this.operationCient = new ZkOperationImpl();
  this.nodeWatcher = new NodeStateWatcher(this);
  this.operationCient.init(Constant.ZK_ADDRESS, nodeWatcher);
 
  for (int i=0; i<Constant.THREAD_COUNT; ++i)
  {
   ClientThread c = new ClientThread(i, ip, name);
   this.clients[i]= c;
  }
 
  initialize();
 }
 
 /**
  * 向zookeeper集群注册
  * @throws InterruptedException
  * @throws KeeperException
  */
 private void registerServer() throws KeeperException, InterruptedException
 {
  List<String> children = operationCient.getChilds(Constant.ROOT_PATH);
  int max = -1;
  for (String childName : children)
  {
   String childPath = Constant.ROOT_PATH + "/" + childName;
   int mod = Integer.parseInt(operationCient.getData(childPath));
   if (mod > max)
    max = mod;
  }
  String path = Constant.ROOT_PATH + "/" + ip;
  operationCient.apendTempNode(path, String.valueOf(max<0 ? 0 : ++max));
 }
 
 /**
  * 启动数据处理线程
  * @throws Exception
  */
 public void run()
 {
  for (ClientThread c : clients)
  {
   CommonUtil.log("Start thread-" + c);
   c.start();
  }
 }
 
 /**
  * 服务器初始化
  * @throws InterruptedException
  * @throws KeeperException
  */
 private void initialize() throws KeeperException, InterruptedException
 {
  CommonUtil.log("================");
  CommonUtil.log(this + " initializing...");
 
  // 配置信息的上级目录不存在
  if (!operationCient.exist(Constant.ROOT_PATH))
  {
   System.err.println("Root path " + Constant.ROOT_PATH + "does not exist!!! Create root path...");
   operationCient.apendPresistentNode(Constant.ROOT_PATH, "1");
   CommonUtil.log("Create root path " + Constant.ROOT_PATH + " successfully!");
  }
 
  registerServer();
 
  refreshConfig();
 
  CommonUtil.log(this + " finish initializing...");
  CommonUtil.log("================");
 }
 
 /**
  * watch到节点变化后，刷新节点数和模数
  * @throws InterruptedException
  * @throws KeeperException
  */
 public void refresh() throws KeeperException, InterruptedException
 {
  CommonUtil.log("================");
  CommonUtil.log(this + ":freshing...");
 
  refreshConfig();
 
  CommonUtil.log(this + ":end freshing...");
  CommonUtil.log("================");
 }
 
 private void refreshConfig() throws KeeperException, InterruptedException
 {
  String version = operationCient.getData(Constant.ROOT_PATH);
  CommonUtil.log("SYSTEM VERSION: " + version);
  List<String> children = operationCient.getChilds(Constant.ROOT_PATH);
 
  // 1. 服务器数量为子节点的个数
  int nodeCount = children.size();
  CommonUtil.log("Server count:" + nodeCount);
  synchronized (CommonUtil.BASE)
  {
   CommonUtil.BASE = nodeCount * Constant.THREAD_COUNT;
  }
 
  if (CommonUtil.BASE.intValue() == 0)
   return;
 
  Integer mod = null;
 
  for (String childName : children)
  {
   // 2. 获取本服务器的模数
   if (childName.equals(ip))
   {
    String childPath = Constant.ROOT_PATH + "/" + childName;
    mod = Integer.parseInt(operationCient.getData(childPath));
    break;
   }
  }
  // 3. 刷新数据处理线程的取模数
  if (mod == null)
  {
   System.err.println("Did not get the mod number for " + this);
  }
  else
  {
   CommonUtil.log(this + ", mod=" + mod + ",base=" + CommonUtil.BASE);
   for (ClientThread c : clients)
   {
    c.refresh(mod);
   }
  }
 }
 
 public String toString()
 {
  return this.name + "@" + this.ip + "";
 }
 
 public ClientThread[] getClients()
 {
  return clients;
 }
 
 public ZkOperation getOperationCient()
 {
  return operationCient;
 }
 
 public Watcher getNodeWatcher()
 {
  return nodeWatcher;
 }
 
 public String getIp()
 {
  return ip;
 }
}
```
ClientThread.java
```java
public class ClientThread extends Thread
{
 
 private Integer modNum = -1;
 private Integer threadId;
 private String ip;
 private String clientName;
 
 public ClientThread(Integer threadId, String ip, String clientName) throws IOException, KeeperException, InterruptedException
 {
  this.threadId = threadId;
  this.ip = ip;
  this.clientName = clientName;
 }
 
 /**
  * watch到节点变化后，调用刷新节点数和模数
  * @throws InterruptedException
  * @throws KeeperException
  */
 public void refresh(int mod) throws KeeperException, InterruptedException
 {
//  CommonUtil.log("================");
//  CommonUtil.log(this + ":freshing...");
 
  synchronized (this.modNum)
  {
   this.modNum = threadId + mod * Constant.THREAD_COUNT;
  }
 
  CommonUtil.log(this + ":" + modNum + "/" + CommonUtil.BASE);
 
//  CommonUtil.log(this + ":end freshing...");
//  CommonUtil.log("================");
 }
 
 @Override
 public void run()
 {
  long start = System.currentTimeMillis();
  while (System.currentTimeMillis() - start < Constant.DURATION)
  {
   // 处理数据
   processData();
   try
   {
    Thread.sleep(5000); //等待2秒
   }
   catch (InterruptedException e)
   {
    e.printStackTrace();
   }
  }
 
 }
 
 /**
  * 模拟处理数据逻辑：打印属于本线程的数据
  */
 private void processData()
 {
  if (CommonUtil.BASE.equals(0) || modNum.equals(-1))
  {
   CommonUtil.err(this + ": did not get server_count and modNum!!!");
   return;
  }
 
  StringBuilder sb = new StringBuilder(this + "-" + modNum + "/" + CommonUtil.BASE + ":");
  for (int i=0; i<Constant.NUMBERS.length; ++i)
  {
   int n = Constant.NUMBERS[i];
   if (n % CommonUtil.BASE == modNum)
   {
    sb.append(n).append(" ");
   }
  }
  CommonUtil.log(sb.toString());
 }
 
 @Override
 public String toString()
 {
  return "ClientThread_" + this.clientName + "@" + this.ip + "-thread_" + this.threadId;
 }
 
 public Integer getModNum()
 {
  return modNum;
 }
 
 public synchronized void setModNum(Integer modNum)
 {
  this.modNum = modNum;
 }
 
 public String getClientName()
 {
  return clientName;
 }
}
```
NodeStateWatcher.java
```java
public class NodeStateWatcher implements Watcher
{
 private Server server;
 
 public NodeStateWatcher(Server server)
 {
  this.server = server;
 }
 
 @Override
 public void process(WatchedEvent event)
 {
  StringBuilder outputStr = new StringBuilder();
  if (server.getName() != null)
  {
   outputStr.append(server.getName() + " get an event.");
  }
  outputStr.append("Path:" + event.getPath());
  outputStr.append(",state:" + event.getState());
  outputStr.append(",type:" + event.getType());
  CommonUtil.log(outputStr.toString());
 
  // 发现子节点有变化
  if (event.getType() == EventType.NodeChildrenChanged
    || event.getType() == EventType.NodeDataChanged
    || event.getType() == EventType.NodeDeleted)
  {
   CommonUtil.log("In event: " + event.getType());
   try
   {
    server.refresh();
   }
   catch (KeeperException e)
   {
    e.printStackTrace();
   }
   catch (InterruptedException e)
   {
    e.printStackTrace();
   }
   CommonUtil.log("End event: " + event.getType());
  }
 }
}
```
ZkOperationImpl.java 部分zk操作代码
```java
@Override
 public void apendPresistentNode(String path, String data)
   throws KeeperException, InterruptedException
 {
  if (zk != null)
  {
   zk.create(path, data.getBytes(), Ids.OPEN_ACL_UNSAFE,
     CreateMode.PERSISTENT);
  }
 }
 
 @Override
 public void delNode(String path) throws KeeperException,
   InterruptedException
 {
  if (zk != null)
  {
   zk.delete(path, -1);
  }
 }
 
 @Override
 public boolean exist(String path) throws KeeperException,
   InterruptedException
 {
  if (zk != null)
  {
   return zk.exists(path, true) != null;
  }
  return false;
 }
}
```

Main.java：主类，启动demo

```java
public class Main
{
 public static void main(String[] args) throws Exception
 {
  Server c1 = new Server("ServerA", "1.1.1.1");
  Server c2 = new Server("ServerB", "1.1.1.2");
  Server c3 = new Server("ServerC", "1.1.1.3");
 
  c1.start();
  c2.start();
  c3.start();
 }
}
```


验证
--
由于Server的3个实例在同一台机器上运行，连接到zookeeper时，用的是一个session，所以demo中没有通过程序断开server与zookeeper的连接，如果serverA断开，那么serverB和serverC与zookeeper的session连接也会失效，达不到演示效果，所以我们只能暂时在zookeeper客户端手工更改zookeeper上的配置信息，用于模拟server与zookeeper集群断开连接和增加server的情形。server启动后，会先向zookeeper注册节点，因此我们先手工删除节点，再手工添加节点。  
手工执行的命令如下：  
> [zk: localhost:2181(CONNECTED) 141] delete /demo/1.1.1.3
> [zk: localhost:2181(CONNECTED) 142] delete /demo/1.1.1.2
> [zk: localhost:2181(CONNECTED) 143] delete /demo/1.1.1.1
> [zk: localhost:2181(CONNECTED) 144] create -e /demo/1.1.1.1 0
> [zk: localhost:2181(CONNECTED) 145] create -e /demo/1.1.1.2 1
> [zk: localhost:2181(CONNECTED) 146] create -e /demo/1.1.1.3 2  

可以通过程序打印信息发现，在节点配置信息每个服务器(Server)上的线程会动态的获取属于自己的数据并打印。当然，这里对数据的处理逻辑很简单，仅仅是打印出来，处理的数据也只是内存中的一个数组，对于类似这样的但是更复杂的应用场景，zookeeper同样适用，但是需要更多的考虑服务器与zookeeper集群连接的可靠性（比如session超时重连）、权限机制等等。  
上面的demo程序打印信息如下：  

> [2012-11-14 15:18:42] New zk connection session: 0
> [2012-11-14 15:18:42] ================
> [2012-11-14 15:18:42] ServerA@1.1.1.1 initializing…
> [2012-11-14 15:18:47] Thread-0 get an event.Path:null,state:SyncConnected,type:None
> [2012-11-14 15:18:47] Thread-0 get an event.Path:/demo,state:SyncConnected,type:NodeChildrenChanged
> [2012-11-14 15:18:47] In event: NodeChildrenChanged
> [2012-11-14 15:18:47] ================
> [2012-11-14 15:18:47] ServerA@1.1.1.1:freshing…
> [2012-11-14 15:18:47] SYSTEM VERSION: 1
> [2012-11-14 15:18:47] SYSTEM VERSION: 1
> [2012-11-14 15:18:47] Server count:1
> [2012-11-14 15:18:47] Server count:1
> [2012-11-14 15:18:47] ServerA@1.1.1.1, mod=0,base=5
> [2012-11-14 15:18:47] ClientThread_ServerA@1.1.1.1-thread_0:0/5
> [2012-11-14 15:18:47] ClientThread_ServerA@1.1.1.1-thread_1:1/5
> [2012-11-14 15:18:47] ClientThread_ServerA@1.1.1.1-thread_2:2/5
> [2012-11-14 15:18:47] ClientThread_ServerA@1.1.1.1-thread_3:3/5
> [2012-11-14 15:18:47] ClientThread_ServerA@1.1.1.1-thread_4:4/5
> [2012-11-14 15:18:47] ServerA@1.1.1.1 finish initializing…
> [2012-11-14 15:18:47] ================
> [2012-11-14 15:18:47] ServerA@1.1.1.1, mod=0,base=5
> [2012-11-14 15:18:47] ClientThread_ServerA@1.1.1.1-thread_0:0/5
> [2012-11-14 15:18:47] ClientThread_ServerA@1.1.1.1-thread_1:1/5
> [2012-11-14 15:18:47] ClientThread_ServerA@1.1.1.1-thread_2:2/5
> [2012-11-14 15:18:47] ClientThread_ServerA@1.1.1.1-thread_3:3/5
> [2012-11-14 15:18:47] ClientThread_ServerA@1.1.1.1-thread_4:4/5
> [2012-11-14 15:18:47] ServerA@1.1.1.1:end freshing…
> [2012-11-14 15:18:47] ================
> [2012-11-14 15:18:47] End event: NodeChildrenChanged
> [2012-11-14 15:18:47] New zk connection session: 0
> [2012-11-14 15:18:47] ================
> [2012-11-14 15:18:47] ServerB@1.1.1.2 initializing…
> [2012-11-14 15:18:51] Thread-6 get an event.Path:null,state:SyncConnected,type:None
> [2012-11-14 15:18:51] Thread-0 get an event.Path:/demo,state:SyncConnected,type:NodeChildrenChanged
> [2012-11-14 15:18:51] In event: NodeChildrenChanged
> [2012-11-14 15:18:51] ================
> [2012-11-14 15:18:51] ServerA@1.1.1.1:freshing…
> [2012-11-14 15:18:51] Thread-6 get an event.Path:/demo,state:SyncConnected,type:NodeChildrenChanged
> [2012-11-14 15:18:51] In event: NodeChildrenChanged
> [2012-11-14 15:18:51] ================
> [2012-11-14 15:18:51] ServerB@1.1.1.2:freshing…
> [2012-11-14 15:18:51] SYSTEM VERSION: 1
> [2012-11-14 15:18:51] SYSTEM VERSION: 1
> [2012-11-14 15:18:51] Server count:2
> [2012-11-14 15:18:51] SYSTEM VERSION: 1
> [2012-11-14 15:18:51] ServerA@1.1.1.1, mod=0,base=10
> [2012-11-14 15:18:51] ClientThread_ServerA@1.1.1.1-thread_0:0/10
> [2012-11-14 15:18:51] ClientThread_ServerA@1.1.1.1-thread_1:1/10
> [2012-11-14 15:18:51] ClientThread_ServerA@1.1.1.1-thread_2:2/10
> [2012-11-14 15:18:51] ClientThread_ServerA@1.1.1.1-thread_3:3/10
> [2012-11-14 15:18:51] ClientThread_ServerA@1.1.1.1-thread_4:4/10
> [2012-11-14 15:18:51] ServerA@1.1.1.1:end freshing…
> [2012-11-14 15:18:51] Server count:2
> [2012-11-14 15:18:51] ================
> [2012-11-14 15:18:51] End event: NodeChildrenChanged
> [2012-11-14 15:18:51] Server count:2
> [2012-11-14 15:18:51] ServerB@1.1.1.2, mod=1,base=10
> [2012-11-14 15:18:51] ClientThread_ServerB@1.1.1.2-thread_0:5/10
> [2012-11-14 15:18:51] ClientThread_ServerB@1.1.1.2-thread_1:6/10
> [2012-11-14 15:18:51] ClientThread_ServerB@1.1.1.2-thread_2:7/10
> [2012-11-14 15:18:51] ClientThread_ServerB@1.1.1.2-thread_3:8/10
> [2012-11-14 15:18:51] ClientThread_ServerB@1.1.1.2-thread_4:9/10
> [2012-11-14 15:18:51] ServerB@1.1.1.2:end freshing…
> [2012-11-14 15:18:51] ================
> [2012-11-14 15:18:51] End event: NodeChildrenChanged
> [2012-11-14 15:18:51] ServerB@1.1.1.2, mod=1,base=10
> [2012-11-14 15:18:51] ClientThread_ServerB@1.1.1.2-thread_0:5/10
> [2012-11-14 15:18:51] ClientThread_ServerB@1.1.1.2-thread_1:6/10
> [2012-11-14 15:18:51] ClientThread_ServerB@1.1.1.2-thread_2:7/10
> [2012-11-14 15:18:51] ClientThread_ServerB@1.1.1.2-thread_3:8/10
> [2012-11-14 15:18:51] ClientThread_ServerB@1.1.1.2-thread_4:9/10
> [2012-11-14 15:18:51] ServerB@1.1.1.2 finish initializing…
> [2012-11-14 15:18:51] ================
> [2012-11-14 15:18:51] New zk connection session: 0
> [2012-11-14 15:18:51] ================
> [2012-11-14 15:18:51] ServerC@1.1.1.3 initializing…
> [2012-11-14 15:18:56] Thread-12 get an event.Path:null,state:SyncConnected,type:None
> [2012-11-14 15:18:56] Thread-0 get an event.Path:/demo,state:SyncConnected,type:NodeChildrenChanged
> [2012-11-14 15:18:56] In event: NodeChildrenChanged
> [2012-11-14 15:18:56] ================
> [2012-11-14 15:18:56] ServerA@1.1.1.1:freshing…
> [2012-11-14 15:18:56] Thread-6 get an event.Path:/demo,state:SyncConnected,type:NodeChildrenChanged
> [2012-11-14 15:18:56] In event: NodeChildrenChanged
> [2012-11-14 15:18:56] ================
> [2012-11-14 15:18:56] ServerB@1.1.1.2:freshing…
> [2012-11-14 15:18:56] Thread-12 get an event.Path:/demo,state:SyncConnected,type:NodeChildrenChanged
> [2012-11-14 15:18:56] In event: NodeChildrenChanged
> [2012-11-14 15:18:56] ================
> [2012-11-14 15:18:56] ServerC@1.1.1.3:freshing…
> [2012-11-14 15:18:56] SYSTEM VERSION: 1
> [2012-11-14 15:18:56] SYSTEM VERSION: 1
> [2012-11-14 15:18:56] SYSTEM VERSION: 1
> [2012-11-14 15:18:56] Server count:3
> [2012-11-14 15:18:56] ServerA@1.1.1.1, mod=0,base=15
> [2012-11-14 15:18:56] ClientThread_ServerA@1.1.1.1-thread_0:0/15
> [2012-11-14 15:18:56] Server count:3
> [2012-11-14 15:18:56] ClientThread_ServerA@1.1.1.1-thread_1:1/15
> [2012-11-14 15:18:56] ClientThread_ServerA@1.1.1.1-thread_2:2/15
> [2012-11-14 15:18:56] ClientThread_ServerA@1.1.1.1-thread_3:3/15
> [2012-11-14 15:18:56] ClientThread_ServerA@1.1.1.1-thread_4:4/15
> [2012-11-14 15:18:56] ServerA@1.1.1.1:end freshing…
> [2012-11-14 15:18:56] ================
> [2012-11-14 15:18:56] End event: NodeChildrenChanged
> [2012-11-14 15:18:56] SYSTEM VERSION: 1
> [2012-11-14 15:18:56] Server count:3
> [2012-11-14 15:18:56] ServerB@1.1.1.2, mod=1,base=15
> [2012-11-14 15:18:56] ClientThread_ServerB@1.1.1.2-thread_0:5/15
> [2012-11-14 15:18:56] ClientThread_ServerB@1.1.1.2-thread_1:6/15
> [2012-11-14 15:18:56] ClientThread_ServerB@1.1.1.2-thread_2:7/15
> [2012-11-14 15:18:56] ClientThread_ServerB@1.1.1.2-thread_3:8/15
> [2012-11-14 15:18:56] ClientThread_ServerB@1.1.1.2-thread_4:9/15
> [2012-11-14 15:18:56] ServerB@1.1.1.2:end freshing…
> [2012-11-14 15:18:56] ================
> [2012-11-14 15:18:56] End event: NodeChildrenChanged
> [2012-11-14 15:18:56] Server count:3
> [2012-11-14 15:18:56] ServerC@1.1.1.3, mod=2,base=15
> [2012-11-14 15:18:56] ClientThread_ServerC@1.1.1.3-thread_0:10/15
> [2012-11-14 15:18:56] ClientThread_ServerC@1.1.1.3-thread_1:11/15
> [2012-11-14 15:18:56] ClientThread_ServerC@1.1.1.3-thread_2:12/15
> [2012-11-14 15:18:56] ClientThread_ServerC@1.1.1.3-thread_3:13/15
> [2012-11-14 15:18:56] ClientThread_ServerC@1.1.1.3-thread_4:14/15
> [2012-11-14 15:18:56] ServerC@1.1.1.3:end freshing…
> [2012-11-14 15:18:56] ================
> [2012-11-14 15:18:56] End event: NodeChildrenChanged
> [2012-11-14 15:18:56] ServerC@1.1.1.3, mod=2,base=15
> [2012-11-14 15:18:56] ClientThread_ServerC@1.1.1.3-thread_0:10/15
> [2012-11-14 15:18:56] ClientThread_ServerC@1.1.1.3-thread_1:11/15
> [2012-11-14 15:18:56] ClientThread_ServerC@1.1.1.3-thread_2:12/15
> [2012-11-14 15:18:56] ClientThread_ServerC@1.1.1.3-thread_3:13/15
> [2012-11-14 15:18:56] ClientThread_ServerC@1.1.1.3-thread_4:14/15
> [2012-11-14 15:18:56] ServerC@1.1.1.3 finish initializing…
> [2012-11-14 15:18:56] ================
> [2012-11-14 15:18:56] Start thread-ClientThread_ServerA@1.1.1.1-thread_0
> [2012-11-14 15:18:56] Start thread-ClientThread_ServerB@1.1.1.2-thread_0
> [2012-11-14 15:18:56] Start thread-ClientThread_ServerC@1.1.1.3-thread_0
> [2012-11-14 15:18:56] Start thread-ClientThread_ServerB@1.1.1.2-thread_1
> [2012-11-14 15:18:56] Start thread-ClientThread_ServerA@1.1.1.1-thread_1
> [2012-11-14 15:18:56] ClientThread_ServerA@1.1.1.1-thread_0-0/15:15 30
> [2012-11-14 15:18:56] Start thread-ClientThread_ServerA@1.1.1.1-thread_2
> [2012-11-14 15:18:56] ClientThread_ServerC@1.1.1.3-thread_0-10/15:10 25
> [2012-11-14 15:18:56] Start thread-ClientThread_ServerA@1.1.1.1-thread_3
> [2012-11-14 15:18:56] ClientThread_ServerB@1.1.1.2-thread_0-5/15:5 20
> [2012-11-14 15:18:56] Start thread-ClientThread_ServerA@1.1.1.1-thread_4
> [2012-11-14 15:18:56] ClientThread_ServerA@1.1.1.1-thread_1-1/15:1 16
> [2012-11-14 15:18:56] ClientThread_ServerA@1.1.1.1-thread_3-3/15:3 18
> [2012-11-14 15:18:56] ClientThread_ServerA@1.1.1.1-thread_2-2/15:2 17
> [2012-11-14 15:18:56] Start thread-ClientThread_ServerB@1.1.1.2-thread_2
> [2012-11-14 15:18:56] ClientThread_ServerB@1.1.1.2-thread_1-6/15:6 21
> [2012-11-14 15:18:56] Start thread-ClientThread_ServerB@1.1.1.2-thread_3
> [2012-11-14 15:18:56] ClientThread_ServerB@1.1.1.2-thread_2-7/15:7 22
> [2012-11-14 15:18:56] ClientThread_ServerB@1.1.1.2-thread_3-8/15:8 23
> [2012-11-14 15:18:56] Start thread-ClientThread_ServerC@1.1.1.3-thread_1
> [2012-11-14 15:18:56] Start thread-ClientThread_ServerB@1.1.1.2-thread_4
> [2012-11-14 15:18:56] ClientThread_ServerC@1.1.1.3-thread_1-11/15:11 26
> [2012-11-14 15:18:56] ClientThread_ServerA@1.1.1.1-thread_4-4/15:4 19
> [2012-11-14 15:18:56] Start thread-ClientThread_ServerC@1.1.1.3-thread_2
> [2012-11-14 15:18:56] ClientThread_ServerB@1.1.1.2-thread_4-9/15:9 24
> [2012-11-14 15:18:56] Start thread-ClientThread_ServerC@1.1.1.3-thread_3
> [2012-11-14 15:18:56] ClientThread_ServerC@1.1.1.3-thread_2-12/15:12 27
> [2012-11-14 15:18:56] Start thread-ClientThread_ServerC@1.1.1.3-thread_4
> [2012-11-14 15:18:56] ClientThread_ServerC@1.1.1.3-thread_3-13/15:13 28
> [2012-11-14 15:18:56] ClientThread_ServerC@1.1.1.3-thread_4-14/15:14 29
> [2012-11-14 15:19:01] ClientThread_ServerB@1.1.1.2-thread_0-5/15:5 20
> [2012-11-14 15:19:01] ClientThread_ServerA@1.1.1.1-thread_3-3/15:3 18
> [2012-11-14 15:19:01] ClientThread_ServerA@1.1.1.1-thread_0-0/15:15 30
> [2012-11-14 15:19:01] ClientThread_ServerB@1.1.1.2-thread_1-6/15:6 21
> [2012-11-14 15:19:01] ClientThread_ServerA@1.1.1.1-thread_1-1/15:1 16
> [2012-11-14 15:19:01] ClientThread_ServerC@1.1.1.3-thread_0-10/15:10 25
> [2012-11-14 15:19:01] ClientThread_ServerA@1.1.1.1-thread_2-2/15:2 17
> [2012-11-14 15:19:01] ClientThread_ServerB@1.1.1.2-thread_2-7/15:7 22
> [2012-11-14 15:19:01] ClientThread_ServerC@1.1.1.3-thread_1-11/15:11 26
> [2012-11-14 15:19:01] ClientThread_ServerB@1.1.1.2-thread_3-8/15:8 23
> [2012-11-14 15:19:01] ClientThread_ServerA@1.1.1.1-thread_4-4/15:4 19
> [2012-11-14 15:19:01] ClientThread_ServerC@1.1.1.3-thread_2-12/15:12 27
> [2012-11-14 15:19:01] ClientThread_ServerB@1.1.1.2-thread_4-9/15:9 24
> [2012-11-14 15:19:01] ClientThread_ServerC@1.1.1.3-thread_3-13/15:13 28
> [2012-11-14 15:19:01] ClientThread_ServerC@1.1.1.3-thread_4-14/15:14 29
> [2012-11-14 15:19:02] Thread-0 get an event.Path:/demo/1.1.1.1,state:SyncConnected,type:NodeDeleted
> [2012-11-14 15:19:02] In event: NodeDeleted
> [2012-11-14 15:19:02] ================
> [2012-11-14 15:19:02] ServerA@1.1.1.1:freshing…
> [2012-11-14 15:19:02] Thread-12 get an event.Path:/demo/1.1.1.1,state:SyncConnected,type:NodeDeleted
> [2012-11-14 15:19:02] In event: NodeDeleted
> [2012-11-14 15:19:02] ================
> [2012-11-14 15:19:02] ServerC@1.1.1.3:freshing…
> [2012-11-14 15:19:02] Thread-6 get an event.Path:/demo/1.1.1.1,state:SyncConnected,type:NodeDeleted
> [2012-11-14 15:19:02] In event: NodeDeleted
> [2012-11-14 15:19:02] ================
> [2012-11-14 15:19:02] ServerB@1.1.1.2:freshing…
> [2012-11-14 15:19:02] SYSTEM VERSION: 1
> [2012-11-14 15:19:02] SYSTEM VERSION: 1
> [2012-11-14 15:19:02] SYSTEM VERSION: 1
> [2012-11-14 15:19:02] Server count:2
> [2012-11-14 15:19:02] Server count:2
> Did not get the mod number for ServerA@1.1.1.1
> [2012-11-14 15:19:02] ServerA@1.1.1.1:end freshing…
> [2012-11-14 15:19:02] ================
> [2012-11-14 15:19:02] End event: NodeDeleted
> [2012-11-14 15:19:02] Thread-0 get an event.Path:/demo,state:SyncConnected,type:NodeChildrenChanged
> [2012-11-14 15:19:02] In event: NodeChildrenChanged
> [2012-11-14 15:19:02] ================
> [2012-11-14 15:19:02] ServerA@1.1.1.1:freshing…
> [2012-11-14 15:19:02] Server count:2
> [2012-11-14 15:19:02] ServerC@1.1.1.3, mod=2,base=10
> [2012-11-14 15:19:02] ClientThread_ServerC@1.1.1.3-thread_0:10/10
> [2012-11-14 15:19:02] ClientThread_ServerC@1.1.1.3-thread_1:11/10
> [2012-11-14 15:19:02] ClientThread_ServerC@1.1.1.3-thread_2:12/10
> [2012-11-14 15:19:02] ClientThread_ServerC@1.1.1.3-thread_3:13/10
> [2012-11-14 15:19:02] ClientThread_ServerC@1.1.1.3-thread_4:14/10
> [2012-11-14 15:19:02] ServerC@1.1.1.3:end freshing…
> [2012-11-14 15:19:02] ================
> [2012-11-14 15:19:02] End event: NodeDeleted
> [2012-11-14 15:19:02] Thread-12 get an event.Path:/demo,state:SyncConnected,type:NodeChildrenChanged
> [2012-11-14 15:19:02] In event: NodeChildrenChanged
> [2012-11-14 15:19:02] ================
> [2012-11-14 15:19:02] ServerC@1.1.1.3:freshing…
> [2012-11-14 15:19:02] ServerB@1.1.1.2, mod=1,base=10
> [2012-11-14 15:19:02] SYSTEM VERSION: 1
> [2012-11-14 15:19:02] ClientThread_ServerB@1.1.1.2-thread_0:5/10
> [2012-11-14 15:19:02] ClientThread_ServerB@1.1.1.2-thread_1:6/10
> [2012-11-14 15:19:02] ClientThread_ServerB@1.1.1.2-thread_2:7/10
> [2012-11-14 15:19:02] ClientThread_ServerB@1.1.1.2-thread_3:8/10
> [2012-11-14 15:19:02] ClientThread_ServerB@1.1.1.2-thread_4:9/10
> [2012-11-14 15:19:02] ServerB@1.1.1.2:end freshing…
> [2012-11-14 15:19:02] ================
> [2012-11-14 15:19:02] End event: NodeDeleted
> [2012-11-14 15:19:02] Thread-6 get an event.Path:/demo,state:SyncConnected,type:NodeChildrenChanged
> [2012-11-14 15:19:02] In event: NodeChildrenChanged
> [2012-11-14 15:19:02] ================
> [2012-11-14 15:19:02] ServerB@1.1.1.2:freshing…
> [2012-11-14 15:19:02] SYSTEM VERSION: 1
> [2012-11-14 15:19:02] Server count:2
> [2012-11-14 15:19:02] ServerA@1.1.1.1:end freshing…
> [2012-11-14 15:19:02] ================
> [2012-11-14 15:19:02] End event: NodeChildrenChanged
> Did not get the mod number for ServerA@1.1.1.1
> [2012-11-14 15:19:02] SYSTEM VERSION: 1
> [2012-11-14 15:19:02] Server count:2
> [2012-11-14 15:19:02] Server count:2
> [2012-11-14 15:19:02] ServerC@1.1.1.3, mod=2,base=10
> [2012-11-14 15:19:02] ClientThread_ServerC@1.1.1.3-thread_0:10/10
> [2012-11-14 15:19:02] ClientThread_ServerC@1.1.1.3-thread_1:11/10
> [2012-11-14 15:19:02] ClientThread_ServerC@1.1.1.3-thread_2:12/10
> [2012-11-14 15:19:02] ClientThread_ServerC@1.1.1.3-thread_3:13/10
> [2012-11-14 15:19:02] ServerB@1.1.1.2, mod=1,base=10
> [2012-11-14 15:19:02] ClientThread_ServerC@1.1.1.3-thread_4:14/10
> [2012-11-14 15:19:02] ClientThread_ServerB@1.1.1.2-thread_0:5/10
> [2012-11-14 15:19:02] ServerC@1.1.1.3:end freshing…
> [2012-11-14 15:19:02] ClientThread_ServerB@1.1.1.2-thread_1:6/10
> [2012-11-14 15:19:02] ================
> [2012-11-14 15:19:02] End event: NodeChildrenChanged
> [2012-11-14 15:19:02] ClientThread_ServerB@1.1.1.2-thread_2:7/10
> [2012-11-14 15:19:02] ClientThread_ServerB@1.1.1.2-thread_3:8/10
> [2012-11-14 15:19:02] ClientThread_ServerB@1.1.1.2-thread_4:9/10
> [2012-11-14 15:19:02] ServerB@1.1.1.2:end freshing…
> [2012-11-14 15:19:02] ================
> [2012-11-14 15:19:02] End event: NodeChildrenChanged
> [2012-11-14 15:19:06] ClientThread_ServerA@1.1.1.1-thread_1-1/10:1 11 21
> [2012-11-14 15:19:06] ClientThread_ServerB@1.1.1.2-thread_1-6/10:6 16 26
> [2012-11-14 15:19:06] ClientThread_ServerB@1.1.1.2-thread_0-5/10:5 15 25
> [2012-11-14 15:19:06] ClientThread_ServerA@1.1.1.1-thread_2-2/10:2 12 22
> [2012-11-14 15:19:06] ClientThread_ServerC@1.1.1.3-thread_0-10/10:
> [2012-11-14 15:19:06] ClientThread_ServerA@1.1.1.1-thread_3-3/10:3 13 23
> [2012-11-14 15:19:06] ClientThread_ServerA@1.1.1.1-thread_0-0/10:10 20 30
> [2012-11-14 15:19:06] ClientThread_ServerA@1.1.1.1-thread_4-4/10:4 14 24
> [2012-11-14 15:19:06] ClientThread_ServerB@1.1.1.2-thread_2-7/10:7 17 27
> [2012-11-14 15:19:06] ClientThread_ServerB@1.1.1.2-thread_3-8/10:8 18 28
> [2012-11-14 15:19:06] ClientThread_ServerC@1.1.1.3-thread_1-11/10:
> [2012-11-14 15:19:06] ClientThread_ServerC@1.1.1.3-thread_2-12/10:
> [2012-11-14 15:19:06] ClientThread_ServerB@1.1.1.2-thread_4-9/10:9 19 29
> [2012-11-14 15:19:06] ClientThread_ServerC@1.1.1.3-thread_3-13/10:
> [2012-11-14 15:19:06] ClientThread_ServerC@1.1.1.3-thread_4-14/10:
> [2012-11-14 15:19:07] Thread-0 get an event.Path:/demo,state:SyncConnected,type:NodeChildrenChanged
> [2012-11-14 15:19:07] In event: NodeChildrenChanged
> [2012-11-14 15:19:07] ================
> [2012-11-14 15:19:07] ServerA@1.1.1.1:freshing…
> [2012-11-14 15:19:07] Thread-12 get an event.Path:/demo/1.1.1.2,state:SyncConnected,type:NodeDeleted
> [2012-11-14 15:19:07] In event: NodeDeleted
> [2012-11-14 15:19:07] ================
> [2012-11-14 15:19:07] ServerC@1.1.1.3:freshing…
> [2012-11-14 15:19:07] Thread-6 get an event.Path:/demo/1.1.1.2,state:SyncConnected,type:NodeDeleted
> [2012-11-14 15:19:07] In event: NodeDeleted
> [2012-11-14 15:19:07] ================
> [2012-11-14 15:19:07] ServerB@1.1.1.2:freshing…
> [2012-11-14 15:19:07] SYSTEM VERSION: 1
> Did not get the mod number for ServerA@1.1.1.1
> [2012-11-14 15:19:07] Server count:1
> [2012-11-14 15:19:07] ServerA@1.1.1.1:end freshing…
> [2012-11-14 15:19:07] ================
> [2012-11-14 15:19:07] End event: NodeChildrenChanged
> [2012-11-14 15:19:07] SYSTEM VERSION: 1
> [2012-11-14 15:19:07] SYSTEM VERSION: 1
> [2012-11-14 15:19:07] Server count:1
> [2012-11-14 15:19:07] Server count:1
> [2012-11-14 15:19:07] ServerB@1.1.1.2:end freshing…
> [2012-11-14 15:19:07] ================
> [2012-11-14 15:19:07] End event: NodeDeleted
> [2012-11-14 15:19:07] Thread-6 get an event.Path:/demo,state:SyncConnected,type:NodeChildrenChanged
> [2012-11-14 15:19:07] In event: NodeChildrenChanged
> [2012-11-14 15:19:07] ================
> [2012-11-14 15:19:07] ServerB@1.1.1.2:freshing…
> Did not get the mod number for ServerB@1.1.1.2
> [2012-11-14 15:19:07] ServerC@1.1.1.3, mod=2,base=5
> [2012-11-14 15:19:07] ClientThread_ServerC@1.1.1.3-thread_0:10/5
> [2012-11-14 15:19:07] ClientThread_ServerC@1.1.1.3-thread_1:11/5
> [2012-11-14 15:19:07] ClientThread_ServerC@1.1.1.3-thread_2:12/5
> [2012-11-14 15:19:07] ClientThread_ServerC@1.1.1.3-thread_3:13/5
> [2012-11-14 15:19:07] ClientThread_ServerC@1.1.1.3-thread_4:14/5
> [2012-11-14 15:19:07] ServerC@1.1.1.3:end freshing…
> [2012-11-14 15:19:07] ================
> [2012-11-14 15:19:07] End event: NodeDeleted
> [2012-11-14 15:19:07] Thread-12 get an event.Path:/demo,state:SyncConnected,type:NodeChildrenChanged
> [2012-11-14 15:19:07] In event: NodeChildrenChanged
> [2012-11-14 15:19:07] ================
> [2012-11-14 15:19:07] ServerC@1.1.1.3:freshing…
> [2012-11-14 15:19:07] SYSTEM VERSION: 1
> Did not get the mod number for ServerB@1.1.1.2
> [2012-11-14 15:19:07] Server count:1
> [2012-11-14 15:19:07] ServerB@1.1.1.2:end freshing…
> [2012-11-14 15:19:07] ================
> [2012-11-14 15:19:07] End event: NodeChildrenChanged
> [2012-11-14 15:19:07] SYSTEM VERSION: 1
> [2012-11-14 15:19:07] Server count:1
> [2012-11-14 15:19:07] ServerC@1.1.1.3, mod=2,base=5
> [2012-11-14 15:19:07] ClientThread_ServerC@1.1.1.3-thread_0:10/5
> [2012-11-14 15:19:07] ClientThread_ServerC@1.1.1.3-thread_1:11/5
> [2012-11-14 15:19:07] ClientThread_ServerC@1.1.1.3-thread_2:12/5
> [2012-11-14 15:19:07] ClientThread_ServerC@1.1.1.3-thread_3:13/5
> [2012-11-14 15:19:07] ClientThread_ServerC@1.1.1.3-thread_4:14/5
> [2012-11-14 15:19:07] ServerC@1.1.1.3:end freshing…
> [2012-11-14 15:19:07] ================
> [2012-11-14 15:19:07] End event: NodeChildrenChanged
> [2012-11-14 15:19:11] ClientThread_ServerB@1.1.1.2-thread_1-6/5:
> [2012-11-14 15:19:11] ClientThread_ServerA@1.1.1.1-thread_0-0/5:5 10 15 20 25 30
> [2012-11-14 15:19:11] ClientThread_ServerC@1.1.1.3-thread_0-10/5:
> [2012-11-14 15:19:11] ClientThread_ServerA@1.1.1.1-thread_1-1/5:1 6 11 16 21 26
> [2012-11-14 15:19:11] ClientThread_ServerA@1.1.1.1-thread_3-3/5:3 8 13 18 23 28
> [2012-11-14 15:19:11] ClientThread_ServerB@1.1.1.2-thread_0-5/5:
> [2012-11-14 15:19:11] ClientThread_ServerA@1.1.1.1-thread_2-2/5:2 7 12 17 22 27
> [2012-11-14 15:19:11] ClientThread_ServerC@1.1.1.3-thread_1-11/5:
> [2012-11-14 15:19:11] ClientThread_ServerB@1.1.1.2-thread_3-8/5:
> [2012-11-14 15:19:11] ClientThread_ServerB@1.1.1.2-thread_2-7/5:
> [2012-11-14 15:19:11] ClientThread_ServerA@1.1.1.1-thread_4-4/5:4 9 14 19 24 29
> [2012-11-14 15:19:11] ClientThread_ServerB@1.1.1.2-thread_4-9/5:
> [2012-11-14 15:19:11] ClientThread_ServerC@1.1.1.3-thread_2-12/5:
> [2012-11-14 15:19:11] ClientThread_ServerC@1.1.1.3-thread_4-14/5:
> [2012-11-14 15:19:11] ClientThread_ServerC@1.1.1.3-thread_3-13/5:
> [2012-11-14 15:19:12] Thread-0 get an event.Path:/demo,state:SyncConnected,type:NodeChildrenChanged
> [2012-11-14 15:19:12] In event: NodeChildrenChanged
> [2012-11-14 15:19:12] ================
> [2012-11-14 15:19:12] ServerA@1.1.1.1:freshing…
> [2012-11-14 15:19:12] Thread-12 get an event.Path:/demo/1.1.1.3,state:SyncConnected,type:NodeDeleted
> [2012-11-14 15:19:12] In event: NodeDeleted
> [2012-11-14 15:19:12] ================
> [2012-11-14 15:19:12] ServerC@1.1.1.3:freshing…
> [2012-11-14 15:19:12] Thread-6 get an event.Path:/demo,state:SyncConnected,type:NodeChildrenChanged
> [2012-11-14 15:19:12] In event: NodeChildrenChanged
> [2012-11-14 15:19:12] ================
> [2012-11-14 15:19:12] ServerB@1.1.1.2:freshing…
> [2012-11-14 15:19:12] SYSTEM VERSION: 1
> [2012-11-14 15:19:12] SYSTEM VERSION: 1
> [2012-11-14 15:19:12] Server count:0
> [2012-11-14 15:19:12] ServerB@1.1.1.2:end freshing…
> [2012-11-14 15:19:12] ================
> [2012-11-14 15:19:12] End event: NodeChildrenChanged
> [2012-11-14 15:19:12] SYSTEM VERSION: 1
> [2012-11-14 15:19:12] Server count:0
> [2012-11-14 15:19:12] ServerA@1.1.1.1:end freshing…
> [2012-11-14 15:19:12] ================
> [2012-11-14 15:19:12] End event: NodeChildrenChanged
> [2012-11-14 15:19:12] Server count:0
> [2012-11-14 15:19:12] ServerC@1.1.1.3:end freshing…
> [2012-11-14 15:19:12] ================
> [2012-11-14 15:19:12] End event: NodeDeleted
> [2012-11-14 15:19:12] Thread-12 get an event.Path:/demo,state:SyncConnected,type:NodeChildrenChanged
> [2012-11-14 15:19:12] In event: NodeChildrenChanged
> [2012-11-14 15:19:12] ================
> [2012-11-14 15:19:12] ServerC@1.1.1.3:freshing…
> [2012-11-14 15:19:12] SYSTEM VERSION: 1
> [2012-11-14 15:19:12] Server count:0
> [2012-11-14 15:19:12] ServerC@1.1.1.3:end freshing…
> [2012-11-14 15:19:12] ================
> [2012-11-14 15:19:12] End event: NodeChildrenChanged
> [2012-11-14 15:19:16] ClientThread_ServerB@1.1.1.2-thread_1: did not get server_count and modNum!!!
> [2012-11-14 15:19:16] ClientThread_ServerA@1.1.1.1-thread_2: did not get server_count and modNum!!!
> [2012-11-14 15:19:16] ClientThread_ServerB@1.1.1.2-thread_0: did not get server_count and modNum!!!
> [2012-11-14 15:19:16] ClientThread_ServerA@1.1.1.1-thread_1: did not get server_count and modNum!!!
> [2012-11-14 15:19:16] ClientThread_ServerC@1.1.1.3-thread_0: did not get server_count and modNum!!!
> [2012-11-14 15:19:16] ClientThread_ServerA@1.1.1.1-thread_3: did not get server_count and modNum!!!
> [2012-11-14 15:19:16] ClientThread_ServerA@1.1.1.1-thread_0: did not get server_count and modNum!!!
> [2012-11-14 15:19:16] ClientThread_ServerA@1.1.1.1-thread_4: did not get server_count and modNum!!!
> [2012-11-14 15:19:16] ClientThread_ServerB@1.1.1.2-thread_3: did not get server_count and modNum!!!
> [2012-11-14 15:19:16] ClientThread_ServerC@1.1.1.3-thread_1: did not get server_count and modNum!!!
> [2012-11-14 15:19:16] ClientThread_ServerB@1.1.1.2-thread_2: did not get server_count and modNum!!!
> [2012-11-14 15:19:16] ClientThread_ServerB@1.1.1.2-thread_4: did not get server_count and modNum!!!
> [2012-11-14 15:19:16] ClientThread_ServerC@1.1.1.3-thread_2: did not get server_count and modNum!!!
> [2012-11-14 15:19:16] ClientThread_ServerC@1.1.1.3-thread_3: did not get server_count and modNum!!!
> [2012-11-14 15:19:16] ClientThread_ServerC@1.1.1.3-thread_4: did not get server_count and modNum!!!
> [2012-11-14 15:19:20] Thread-0 get an event.Path:/demo,state:SyncConnected,type:NodeChildrenChanged
> [2012-11-14 15:19:20] In event: NodeChildrenChanged
> [2012-11-14 15:19:20] ================
> [2012-11-14 15:19:20] ServerA@1.1.1.1:freshing…
> [2012-11-14 15:19:20] Thread-6 get an event.Path:/demo,state:SyncConnected,type:NodeChildrenChanged
> [2012-11-14 15:19:20] In event: NodeChildrenChanged
> [2012-11-14 15:19:20] ================
> [2012-11-14 15:19:20] ServerB@1.1.1.2:freshing…
> [2012-11-14 15:19:20] Thread-12 get an event.Path:/demo,state:SyncConnected,type:NodeChildrenChanged
> [2012-11-14 15:19:20] In event: NodeChildrenChanged
> [2012-11-14 15:19:20] ================
> [2012-11-14 15:19:20] ServerC@1.1.1.3:freshing…
> [2012-11-14 15:19:20] SYSTEM VERSION: 1
> [2012-11-14 15:19:20] SYSTEM VERSION: 1
> [2012-11-14 15:19:20] SYSTEM VERSION: 1
> [2012-11-14 15:19:20] Server count:1
> Did not get the mod number for ServerC@1.1.1.3
> [2012-11-14 15:19:20] Server count:1
> [2012-11-14 15:19:20] ServerC@1.1.1.3:end freshing…
> [2012-11-14 15:19:20] ================
> [2012-11-14 15:19:20] End event: NodeChildrenChanged
> [2012-11-14 15:19:20] ServerA@1.1.1.1, mod=0,base=5
> [2012-11-14 15:19:20] ClientThread_ServerA@1.1.1.1-thread_0:0/5
> [2012-11-14 15:19:20] ClientThread_ServerA@1.1.1.1-thread_1:1/5
> [2012-11-14 15:19:20] ClientThread_ServerA@1.1.1.1-thread_2:2/5
> [2012-11-14 15:19:20] ClientThread_ServerA@1.1.1.1-thread_3:3/5
> [2012-11-14 15:19:20] ClientThread_ServerA@1.1.1.1-thread_4:4/5
> [2012-11-14 15:19:20] ServerA@1.1.1.1:end freshing…
> [2012-11-14 15:19:20] ================
> [2012-11-14 15:19:20] End event: NodeChildrenChanged
> Did not get the mod number for ServerB@1.1.1.2
> [2012-11-14 15:19:20] Server count:1
> [2012-11-14 15:19:20] ServerB@1.1.1.2:end freshing…
> [2012-11-14 15:19:20] ================
> [2012-11-14 15:19:20] End event: NodeChildrenChanged
> [2012-11-14 15:19:21] ClientThread_ServerB@1.1.1.2-thread_1-6/5:
> [2012-11-14 15:19:21] ClientThread_ServerA@1.1.1.1-thread_0-0/5:5 10 15 20 25 30
> [2012-11-14 15:19:21] ClientThread_ServerA@1.1.1.1-thread_2-2/5:2 7 12 17 22 27
> [2012-11-14 15:19:21] ClientThread_ServerA@1.1.1.1-thread_1-1/5:1 6 11 16 21 26
> [2012-11-14 15:19:21] ClientThread_ServerA@1.1.1.1-thread_3-3/5:3 8 13 18 23 28
> [2012-11-14 15:19:21] ClientThread_ServerC@1.1.1.3-thread_0-10/5:
> [2012-11-14 15:19:21] ClientThread_ServerB@1.1.1.2-thread_0-5/5:
> [2012-11-14 15:19:21] ClientThread_ServerC@1.1.1.3-thread_1-11/5:
> [2012-11-14 15:19:21] ClientThread_ServerB@1.1.1.2-thread_3-8/5:
> [2012-11-14 15:19:21] ClientThread_ServerA@1.1.1.1-thread_4-4/5:4 9 14 19 24 29
> [2012-11-14 15:19:21] ClientThread_ServerB@1.1.1.2-thread_2-7/5:
> [2012-11-14 15:19:21] ClientThread_ServerC@1.1.1.3-thread_2-12/5:
> [2012-11-14 15:19:21] ClientThread_ServerB@1.1.1.2-thread_4-9/5:
> [2012-11-14 15:19:21] ClientThread_ServerC@1.1.1.3-thread_4-14/5:
> [2012-11-14 15:19:21] ClientThread_ServerC@1.1.1.3-thread_3-13/5:
> [2012-11-14 15:19:25] Thread-0 get an event.Path:/demo,state:SyncConnected,type:NodeChildrenChanged
> [2012-11-14 15:19:25] In event: NodeChildrenChanged
> [2012-11-14 15:19:25] ================
> [2012-11-14 15:19:25] ServerA@1.1.1.1:freshing…
> [2012-11-14 15:19:25] Thread-6 get an event.Path:/demo,state:SyncConnected,type:NodeChildrenChanged
> [2012-11-14 15:19:25] In event: NodeChildrenChanged
> [2012-11-14 15:19:25] ================
> [2012-11-14 15:19:25] ServerB@1.1.1.2:freshing…
> [2012-11-14 15:19:25] Thread-12 get an event.Path:/demo,state:SyncConnected,type:NodeChildrenChanged
> [2012-11-14 15:19:25] In event: NodeChildrenChanged
> [2012-11-14 15:19:25] ================
> [2012-11-14 15:19:25] ServerC@1.1.1.3:freshing…
> [2012-11-14 15:19:25] SYSTEM VERSION: 1
> [2012-11-14 15:19:25] SYSTEM VERSION: 1
> [2012-11-14 15:19:25] SYSTEM VERSION: 1
> [2012-11-14 15:19:25] Server count:2
> [2012-11-14 15:19:25] ServerC@1.1.1.3:end freshing…
> [2012-11-14 15:19:25] ================
> Did not get the mod number for ServerC@1.1.1.3
> [2012-11-14 15:19:25] End event: NodeChildrenChanged
> [2012-11-14 15:19:25] Server count:2
> [2012-11-14 15:19:25] Server count:2
> [2012-11-14 15:19:25] ServerB@1.1.1.2, mod=1,base=10
> [2012-11-14 15:19:25] ClientThread_ServerB@1.1.1.2-thread_0:5/10
> [2012-11-14 15:19:25] ClientThread_ServerB@1.1.1.2-thread_1:6/10
> [2012-11-14 15:19:25] ClientThread_ServerB@1.1.1.2-thread_2:7/10
> [2012-11-14 15:19:25] ClientThread_ServerB@1.1.1.2-thread_3:8/10
> [2012-11-14 15:19:25] ClientThread_ServerB@1.1.1.2-thread_4:9/10
> [2012-11-14 15:19:25] ServerB@1.1.1.2:end freshing…
> [2012-11-14 15:19:25] ================
> [2012-11-14 15:19:25] End event: NodeChildrenChanged
> [2012-11-14 15:19:25] ServerA@1.1.1.1, mod=0,base=10
> [2012-11-14 15:19:25] ClientThread_ServerA@1.1.1.1-thread_0:0/10
> [2012-11-14 15:19:25] ClientThread_ServerA@1.1.1.1-thread_1:1/10
> [2012-11-14 15:19:25] ClientThread_ServerA@1.1.1.1-thread_2:2/10
> [2012-11-14 15:19:25] ClientThread_ServerA@1.1.1.1-thread_3:3/10
> [2012-11-14 15:19:25] ClientThread_ServerA@1.1.1.1-thread_4:4/10
> [2012-11-14 15:19:25] ServerA@1.1.1.1:end freshing…
> [2012-11-14 15:19:25] ================
> [2012-11-14 15:19:25] End event: NodeChildrenChanged
> [2012-11-14 15:19:26] ClientThread_ServerA@1.1.1.1-thread_2-2/10:2 12 22
> [2012-11-14 15:19:26] ClientThread_ServerA@1.1.1.1-thread_3-3/10:3 13 23
> [2012-11-14 15:19:26] ClientThread_ServerA@1.1.1.1-thread_0-0/10:10 20 30
> [2012-11-14 15:19:26] ClientThread_ServerA@1.1.1.1-thread_1-1/10:1 11 21
> [2012-11-14 15:19:26] ClientThread_ServerC@1.1.1.3-thread_0-10/10:
> [2012-11-14 15:19:26] ClientThread_ServerB@1.1.1.2-thread_0-5/10:5 15 25
> [2012-11-14 15:19:26] ClientThread_ServerB@1.1.1.2-thread_1-6/10:6 16 26
> [2012-11-14 15:19:26] ClientThread_ServerA@1.1.1.1-thread_4-4/10:4 14 24
> [2012-11-14 15:19:26] ClientThread_ServerC@1.1.1.3-thread_1-11/10:
> [2012-11-14 15:19:26] ClientThread_ServerB@1.1.1.2-thread_3-8/10:8 18 28
> [2012-11-14 15:19:26] ClientThread_ServerB@1.1.1.2-thread_2-7/10:7 17 27
> [2012-11-14 15:19:26] ClientThread_ServerB@1.1.1.2-thread_4-9/10:9 19 29
> [2012-11-14 15:19:26] ClientThread_ServerC@1.1.1.3-thread_2-12/10:
> [2012-11-14 15:19:26] ClientThread_ServerC@1.1.1.3-thread_4-14/10:
> [2012-11-14 15:19:26] ClientThread_ServerC@1.1.1.3-thread_3-13/10:
> [2012-11-14 15:19:31] ClientThread_ServerA@1.1.1.1-thread_1-1/10:1 11 21
> [2012-11-14 15:19:31] ClientThread_ServerA@1.1.1.1-thread_0-0/10:10 20 30
> [2012-11-14 15:19:31] ClientThread_ServerB@1.1.1.2-thread_0-5/10:5 15 25
> [2012-11-14 15:19:31] ClientThread_ServerC@1.1.1.3-thread_0-10/10:
> [2012-11-14 15:19:31] ClientThread_ServerA@1.1.1.1-thread_2-2/10:2 12 22
> [2012-11-14 15:19:31] ClientThread_ServerA@1.1.1.1-thread_3-3/10:3 13 23
> [2012-11-14 15:19:31] ClientThread_ServerB@1.1.1.2-thread_1-6/10:6 16 26
> [2012-11-14 15:19:31] ClientThread_ServerC@1.1.1.3-thread_1-11/10:
> [2012-11-14 15:19:31] ClientThread_ServerA@1.1.1.1-thread_4-4/10:4 14 24
> [2012-11-14 15:19:31] ClientThread_ServerB@1.1.1.2-thread_3-8/10:8 18 28
> [2012-11-14 15:19:31] ClientThread_ServerB@1.1.1.2-thread_2-7/10:7 17 27
> [2012-11-14 15:19:31] ClientThread_ServerB@1.1.1.2-thread_4-9/10:9 19 29
> [2012-11-14 15:19:31] ClientThread_ServerC@1.1.1.3-thread_2-12/10:
> [2012-11-14 15:19:31] ClientThread_ServerC@1.1.1.3-thread_3-13/10:
> [2012-11-14 15:19:31] ClientThread_ServerC@1.1.1.3-thread_4-14/10:
> [2012-11-14 15:19:31] Thread-0 get an event.Path:/demo,state:SyncConnected,type:NodeChildrenChanged
> [2012-11-14 15:19:31] In event: NodeChildrenChanged
> [2012-11-14 15:19:31] ================
> [2012-11-14 15:19:31] ServerA@1.1.1.1:freshing…
> [2012-11-14 15:19:31] Thread-12 get an event.Path:/demo,state:SyncConnected,type:NodeChildrenChanged
> [2012-11-14 15:19:31] In event: NodeChildrenChanged
> [2012-11-14 15:19:31] ================
> [2012-11-14 15:19:31] ServerC@1.1.1.3:freshing…
> [2012-11-14 15:19:31] Thread-6 get an event.Path:/demo,state:SyncConnected,type:NodeChildrenChanged
> [2012-11-14 15:19:31] In event: NodeChildrenChanged
> [2012-11-14 15:19:31] ================
> [2012-11-14 15:19:31] ServerB@1.1.1.2:freshing…
> [2012-11-14 15:19:31] SYSTEM VERSION: 1
> [2012-11-14 15:19:31] SYSTEM VERSION: 1
> [2012-11-14 15:19:31] SYSTEM VERSION: 1
> [2012-11-14 15:19:31] Server count:3
> [2012-11-14 15:19:31] Server count:3
> [2012-11-14 15:19:31] ServerA@1.1.1.1, mod=0,base=15
> [2012-11-14 15:19:31] ClientThread_ServerA@1.1.1.1-thread_0:0/15
> [2012-11-14 15:19:31] ClientThread_ServerA@1.1.1.1-thread_1:1/15
> [2012-11-14 15:19:31] ClientThread_ServerA@1.1.1.1-thread_2:2/15
> [2012-11-14 15:19:31] ClientThread_ServerA@1.1.1.1-thread_3:3/15
> [2012-11-14 15:19:31] ClientThread_ServerA@1.1.1.1-thread_4:4/15
> [2012-11-14 15:19:31] ServerA@1.1.1.1:end freshing…
> [2012-11-14 15:19:31] ================
> [2012-11-14 15:19:31] End event: NodeChildrenChanged
> [2012-11-14 15:19:31] Server count:3
> [2012-11-14 15:19:31] ServerC@1.1.1.3, mod=2,base=15
> [2012-11-14 15:19:31] ClientThread_ServerC@1.1.1.3-thread_0:10/15
> [2012-11-14 15:19:31] ClientThread_ServerC@1.1.1.3-thread_1:11/15
> [2012-11-14 15:19:31] ClientThread_ServerC@1.1.1.3-thread_2:12/15
> [2012-11-14 15:19:31] ClientThread_ServerC@1.1.1.3-thread_3:13/15
> [2012-11-14 15:19:31] ClientThread_ServerC@1.1.1.3-thread_4:14/15
> [2012-11-14 15:19:31] ServerC@1.1.1.3:end freshing…
> [2012-11-14 15:19:31] ================
> [2012-11-14 15:19:31] End event: NodeChildrenChanged
> [2012-11-14 15:19:31] ServerB@1.1.1.2, mod=1,base=15
> [2012-11-14 15:19:31] ClientThread_ServerB@1.1.1.2-thread_0:5/15
> [2012-11-14 15:19:31] ClientThread_ServerB@1.1.1.2-thread_1:6/15
> [2012-11-14 15:19:31] ClientThread_ServerB@1.1.1.2-thread_2:7/15
> [2012-11-14 15:19:31] ClientThread_ServerB@1.1.1.2-thread_3:8/15
> [2012-11-14 15:19:31] ClientThread_ServerB@1.1.1.2-thread_4:9/15
> [2012-11-14 15:19:31] ServerB@1.1.1.2:end freshing…
> [2012-11-14 15:19:31] ================
> [2012-11-14 15:19:31] End event: NodeChildrenChanged
> [2012-11-14 15:19:36] ClientThread_ServerB@1.1.1.2-thread_0-5/15:5 20
> [2012-11-14 15:19:36] ClientThread_ServerA@1.1.1.1-thread_2-2/15:2 17
> [2012-11-14 15:19:36] ClientThread_ServerA@1.1.1.1-thread_3-3/15:3 18
> [2012-11-14 15:19:36] ClientThread_ServerC@1.1.1.3-thread_0-10/15:10 25
> [2012-11-14 15:19:36] ClientThread_ServerA@1.1.1.1-thread_0-0/15:15 30
> [2012-11-14 15:19:36] ClientThread_ServerA@1.1.1.1-thread_1-1/15:1 16
> [2012-11-14 15:19:36] ClientThread_ServerB@1.1.1.2-thread_1-6/15:6 21
> [2012-11-14 15:19:36] ClientThread_ServerC@1.1.1.3-thread_1-11/15:11 26
> [2012-11-14 15:19:36] ClientThread_ServerB@1.1.1.2-thread_3-8/15:8 23
> [2012-11-14 15:19:36] ClientThread_ServerA@1.1.1.1-thread_4-4/15:4 19
> [2012-11-14 15:19:36] ClientThread_ServerB@1.1.1.2-thread_2-7/15:7 22
> [2012-11-14 15:19:36] ClientThread_ServerC@1.1.1.3-thread_2-12/15:12 27
> [2012-11-14 15:19:36] ClientThread_ServerB@1.1.1.2-thread_4-9/15:9 24
> [2012-11-14 15:19:36] ClientThread_ServerC@1.1.1.3-thread_4-14/15:14 29
> [2012-11-14 15:19:36] ClientThread_ServerC@1.1.1.3-thread_3-13/15:13 28
> [2012-11-14 15:19:41] ClientThread_ServerC@1.1.1.3-thread_0-10/15:10 25
> [2012-11-14 15:19:41] ClientThread_ServerA@1.1.1.1-thread_0-0/15:15 30
> [2012-11-14 15:19:41] ClientThread_ServerB@1.1.1.2-thread_1-6/15:6 21
> [2012-11-14 15:19:41] ClientThread_ServerA@1.1.1.1-thread_3-3/15:3 18
> [2012-11-14 15:19:41] ClientThread_ServerB@1.1.1.2-thread_0-5/15:5 20
> [2012-11-14 15:19:41] ClientThread_ServerA@1.1.1.1-thread_1-1/15:1 16
> [2012-11-14 15:19:41] ClientThread_ServerA@1.1.1.1-thread_2-2/15:2 17
> [2012-11-14 15:19:41] ClientThread_ServerB@1.1.1.2-thread_3-8/15:8 23
> [2012-11-14 15:19:41] ClientThread_ServerB@1.1.1.2-thread_2-7/15:7 22
> [2012-11-14 15:19:41] ClientThread_ServerA@1.1.1.1-thread_4-4/15:4 19
> [2012-11-14 15:19:41] ClientThread_ServerC@1.1.1.3-thread_1-11/15:11 26
> [2012-11-14 15:19:41] ClientThread_ServerC@1.1.1.3-thread_2-12/15:12 27
> [2012-11-14 15:19:41] ClientThread_ServerB@1.1.1.2-thread_4-9/15:9 24
> [2012-11-14 15:19:41] ClientThread_ServerC@1.1.1.3-thread_4-14/15:14 29
> [2012-11-14 15:19:41] ClientThread_ServerC@1.1.1.3-thread_3-13/15:13 28
