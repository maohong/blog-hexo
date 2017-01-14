---
title: 将Hadoop RPC框架应用于多节点任务调度
date: 2013-01-21 20:53:38
tags: 
- hadoop
- RPC
- 任务调度
- 分布式应用
categories: 
- Hadoop
---
背景
--
在hadoop中，主从节点之间保持着心跳通信，用于传输节点状态信息、任务调度信息以及节点动作信息等等。 hdfs的namenode与datanode，mapreduce的jobtracker与tasktracker，hbase的hmaster与 regionserver之间的通信，都是基于hadoop RPC。Hadoop RPC是hadoop里非常基础的通信框架。hadoop 2.0以前hadoop RPC的数据序列化是通过实现自己定义的Writable接口实现，而从hadoop 2.0开始，数据的序列化工作交给了ProtocolBuffer去做。关于Hadoop RPC的实现原理已经有很多文章进行了详细的介绍（[源码级强力分析hadoop的RPC机制](http://weixiaolu.iteye.com/blog/1504898)，[Hadoop基于Protocol Buffer的RPC实现代码分析-Server端](http://yanbohappy.sinaapp.com/?p=110)，[带有HA功能的Hadoop Client端RPC实现原理与代码分析](http://yanbohappy.sinaapp.com/?p=115)），这里就不在赘述了。下面就直接引入问题和方案吧。  

问题
--
工作中经常需要在定时任务系统上写一些定时任务，随着业务规模的增长和扩大，需要定时处理的任务越来越多，任务之间的执行间隔越来越小，某一时间段内（比如0点、整点或半点）执行的任务会越来越密集，只在一台机器上执行这些任务的话，会出现较大的风险：  
* 任务并发度较高时，单机的系统资源将成为瓶颈  
* 如果一个任务的运行占用了整个机器的大部分资源，比如sql查询耗费巨大内存和CPU资源，将直接影响其他任务的运行  
* 任务失败后，如果仍然在同一台节点自动重新执行，失败率较高  
* 机器宕机后，必须第一时间重启机器或重新部署定时任务系统，所有任务都不能按时执行  
* 等等  

方案
--
可想而知的是，可以通过将定时任务系统进行分布式改造，使用多个节点执行任务，将任务分发到不同节点上进行处理，并且完善失败重试机制，从而提高系统稳定性，实现任务系统的高可靠。  
既然是在多个节点之间分发任务，肯定得有个任务的管理者(主节点)，在我们现有的系统中，也就是一套可以部署定时任务的web系统，任务代码更新后，部署好这套web系统，即可通过web页面设置定时任务并且进行调度(在单个节点上执行)。执行任务的节点(子节点)有多个以后，如何分发任务到子节点呢，我们可以把任务的信息封装成一个bean，通过RPC发布给子节点，子节点通过这个任务bean获得任务信息，并在指定的时刻执行任务。同时，子节点可以通过与主节点的心跳通信将节点状态和执行任务的情况告诉主节点。  
这样其实就与hadoop mapreduce分发任务有点相似了，呵呵，这里主节点与子节点之间的通信，我们就可以通过Hadoop RPC框架来实现了，不同的是，我们分发的任务是定时任务，发布任务时需要将任务的定时信息一并发给子节点。  

实现
--
单点的定时任务系统是基于Quartz的，在分布式环境下，可以继续基于Quartz进行改造，任务的定时信息可以通过Quartz中的JobDetail和Trigger对象来描述并封装，加上任务执行的入口类信息，再通过RPC由主节点发给子节点。子节点收到封装好的任务信息对象后，再构造JobDetail和Trigger，设置好启动时间后，通过入口类启动任务。下面是一个简单的demo。  
<!--more-->
以下是一个简单的定时任务信息描述对象CronJobInfo，包括JobDetailInfo和TriggerInfo两个属性：  
```java
/**
* 定时任务信息，包括任务信息和触发器信息
*/
public class CronJobInfo implements Writable
{
    private JobDetailInfo jobDetailInfo = new JobDetailInfo();
    private TriggerInfo triggerInfo = new TriggerInfo();
 
    @Override
    public void readFields(DataInput in) throws IOException
    {
        jobDetailInfo.readFields(in);
        triggerInfo.readFields(in);
    }
 
    @Override
    public void write(DataOutput out) throws IOException
    {
        jobDetailInfo.write(out);
        triggerInfo.write(out);
    }
    // getters and setters...
}
```  
  
  
任务信息JobDetailInfo，由主节点构造，子节点解析构造JobDetail对象：  
```java
public class JobDetailInfo implements Writable
{
    private String name; // 任务名称
    private String group = Scheduler.DEFAULT_GROUP; // 任务组
    private String description; // 任务描述
    private Class jobClass; // 任务的启动类
    private JobDataMap jobDataMap; // 任务所需的参数，用来给作业提供数据支持的数据结构
    private boolean volatility = false; // <span>重启应用之后是否删除任务的相关信息,</span>
    private boolean durability = false; // 任务完成之后是否依然保留到数据库
    private boolean shouldRecover = false; // 应用重启之后时候忽略过期任务
 
    @Override
    public void readFields(DataInput in) throws IOException
    {
        name = WritableUtils.readString(in);
        group = WritableUtils.readString(in);
        description = WritableUtils.readString(in);
        String className = WritableUtils.readString(in);
        if (className != null)
        {
          try
          {
             jobClass = Class.forName(new String(className));
          }
          catch (ClassNotFoundException e)
          {
             e.printStackTrace();
          }
        }
        int dataMapSize = WritableUtils.readVInt(in);
        while (dataMapSize-- > 0)
        {
           String key = WritableUtils.readString(in);
           String value = WritableUtils.readString(in);
           jobDataMap.put(key, value);
        }
        volatility = in.readBoolean();
        durability = in.readBoolean();
        shouldRecover = in.readBoolean();
    }
 
    @Override
    public void write(DataOutput out) throws IOException
    {
        WritableUtils.writeString(out, name);
        WritableUtils.writeString(out, group);
        WritableUtils.writeString(out, description);
        WritableUtils.writeString(out, jobClass.getName());
        if (jobDataMap == null)
            WritableUtils.writeVInt(out, 0);
        else
        {
            WritableUtils.writeVInt(out, jobDataMap.size());
            for (Object k : jobDataMap.keySet())
            {
                WritableUtils.writeString(out, k.toString());
                WritableUtils.writeString(out, jobDataMap.get(k).toString());
            }
        }
        out.writeBoolean(volatility);
        out.writeBoolean(durability);
        out.writeBoolean(shouldRecover);
   }
   //getters and setters
   //.....
}
```  
  
  
任务触发器信息TriggerInfo ，由主节点构造，子节点解析构造Trigger对象：  
```java
public class TriggerInfo implements Writable
{
    private String name; // trigger名称
    private String group = Scheduler.DEFAULT_GROUP; // triger组名称
    private String description; // trigger描述
    private Date startTime; // 启动时间
    private Date endTime; // 结束时间
    private long repeatInterval; // 重试时间间隔
    private int repeatCount; //重试次数
 
    @Override
    public void readFields(DataInput in) throws IOException
    {
       name = WritableUtils.readString(in);
       group = WritableUtils.readString(in);
       description = WritableUtils.readString(in);
       long start = in.readLong();
       startTime = start==0 ? null : new Date(start);
       long end = in.readLong();
       endTime = end==0 ? null : new Date(end);
       repeatInterval = in.readLong();
       repeatCount = in.readInt();
    }
 
    @Override
    public void write(DataOutput out) throws IOException
    {
       WritableUtils.writeString(out, name);
       WritableUtils.writeString(out, group);
       WritableUtils.writeString(out, description);
       out.writeLong(startTime == null ? 0 : startTime.getTime());
       out.writeLong(endTime == null ? 0 : endTime.getTime());
       out.writeLong(repeatInterval);
       out.writeInt(repeatCount);
    }
    //getters and setters
    //.....
}
```  
  
  
主从节点通信的协议：  
```java
public interface TaskProtocol extends VersionedProtocol
{
    public CronJobInfo hearbeat();
}
```  
  
在这个demo中，主节点启动后，启动RPC server线程，等待客户端（子节点）的连接，当客户端调用heartbeat方法时，主节点将会生成一个任务信息返回给客户端：  
```java
public class TaskScheduler implements TaskProtocol
{
    private Logger logger = Logger.getLogger(getClass());
    private Server server;
 
    public TaskScheduler()
    {
        try
        {
            server = RPC.getServer(this, "192.168.1.101", 8888, new Configuration());
            server.start();
            server.join();
        }
        catch (UnknownHostException e)
        {
            e.printStackTrace();
        }
        catch (IOException e)
        {
            e.printStackTrace();
        }
        catch (InterruptedException e)
        {
            e.printStackTrace();
        }
    }
 
    @Override
    public long getProtocolVersion(String arg0, long arg1) throws IOException
    {
        return 1;
    }
 
    @Override
    public CronJobInfo generateCronJob()
    {
        // 1、创建JobDetial对象
        JobDetailInfo detail = new JobDetailInfo();
        // 设置工作项
        detail.setJobClass(DemoTask.class);
        detail.setName("MyJob_1");
        detail.setGroup("JobGroup_1");
 
        // 2、创建Trigger对象
        TriggerInfo trigger = new TriggerInfo();
        trigger.setName("Trigger_1");
        trigger.setGroup("Trigger_Group_1");
        trigger.setStartTime(new Date());
        // 设置重复停止时间，并销毁该Trigger对象
        Calendar c = Calendar.getInstance();
        c.setTimeInMillis(System.currentTimeMillis() + 1000 * 1L);
        trigger.setEndTime(c.getTime());
        // 设置重复间隔时间
        trigger.setRepeatInterval(1000 * 1L);
        // 设置重复执行次数
        trigger.setRepeatCount(3);
 
        CronJobInfo info = new CronJobInfo();
        info.setJobDetailInfo(detail);
        info.setTriggerInfo(trigger);
 
        return info;
    }
 
    public static void main(String[] args)
    {
        TaskScheduler ts = new TaskScheduler();
    }
 
}
```  
  
demo任务类，打印信息：  
```java
public class DemoTask implements Job
{
    public void execute(JobExecutionContext context)
            throws JobExecutionException
    {
        System.out.println(this + ": executing task @" + new Date());
    }
} 
```  
  
子节点demo，启动后连接主节点，远程调用generateCronJob方法，获得一个任务描述信息，并启动定时任务。  
```java
public class TaskRunner
{
    private Logger logger = Logger.getLogger(getClass());
    private TaskProtocol proxy;
 
    public TaskRunner()
    {
        InetSocketAddress addr = new InetSocketAddress("localhost", 8888);
        try
        {
            proxy = (TaskProtocol) RPC.waitForProxy(TaskProtocol.class, 1, addr,
                    new Configuration());
        }
        catch (IOException e)
        {
            e.printStackTrace();
        }
    }
 
    public void close()
    {
        RPC.stopProxy(proxy);
    }
 
    /**
     * 从server获取一个定时任务
     */
    public void getCronJob()
    {
        CronJobInfo info = proxy.generateCronJob();
        JobDetail jobDetail = getJobDetail(info.getJobDetailInfo());
        SimpleTrigger trigger = getTrigger(info.getTriggerInfo());
 
        // 创建Scheduler对象，并配置JobDetail和Trigger对象
        SchedulerFactory sf = new StdSchedulerFactory();
        Scheduler scheduler = null;
        try
        {
            scheduler = sf.getScheduler();
            scheduler.scheduleJob(jobDetail, trigger);
            // 执行启动操作
            scheduler.start();
 
        }
        catch (SchedulerException e)
        {
            e.printStackTrace();
        }
    }
 
    /**
     * @param jobDetailInfo
     * @return
     */
    private JobDetail getJobDetail(JobDetailInfo info)
    {
        JobDetail detail = new JobDetail();
        detail.setName(info.getName());
        detail.setGroup(info.getGroup());
        detail.setDescription(info.getDescription());
        detail.setJobClass(info.getJobClass());
        detail.setJobDataMap(info.getJobDataMap());
        detail.setRequestsRecovery(info.isShouldRecover());
        detail.setDurability(info.isDurability());
        detail.setVolatility(info.isVolatility());
        logger.info("client get jobdetail:" + detail);
        return detail;
    }
 
    /**
     * @param triggerInfo
     * @return
     */
    private SimpleTrigger getTrigger(TriggerInfo info)
    {
        SimpleTrigger trigger = new SimpleTrigger();
        trigger.setName(info.getName());
        trigger.setGroup(info.getGroup());
        trigger.setDescription(info.getDescription());
        trigger.setStartTime(info.getStartTime());
        trigger.setEndTime(info.getEndTime());
        trigger.setRepeatInterval(info.getRepeatInterval());
        trigger.setRepeatCount(info.getRepeatCount());
        logger.info("client get trigger:" + trigger);
        return trigger;
    }
 
    public static void main(String[] args)
    {
        TaskRunner t = new TaskRunner();
        t.getCronJob();
        t.close();
    }
}
```  
  
先启动TaskScheduler，再启动TaskRunner，结果如下：  

> TaskScheduler日志:
> 2013-01-20 15:42:21,661 [Socket Reader #1 for port 8888] INFO  [org.apache.hadoop.ipc.Server] – Starting Socket Reader #1 for port 8888
> 2013-01-20 15:42:21,662 [main] INFO  [org.apache.hadoop.ipc.metrics.RpcMetrics] – Initializing RPC Metrics with hostName=TaskScheduler, port=8888
> 2013-01-20 15:42:21,706 [main] INFO  [org.apache.hadoop.ipc.metrics.RpcDetailedMetrics] – Initializing RPC Metrics with hostName=TaskScheduler, port=8888
> 2013-01-20 15:42:21,710 [IPC Server listener on 8888] INFO  [org.apache.hadoop.ipc.Server] – IPC Server listener on 8888: starting
> 2013-01-20 15:42:21,711 [IPC Server Responder] INFO  [org.apache.hadoop.ipc.Server] – IPC Server Responder: starting
> 2013-01-20 15:42:21,711 [IPC Server handler 0 on 8888] INFO  [org.apache.hadoop.ipc.Server] – IPC Server handler 0 on 8888: starting
> 2013-01-20 15:42:24,084 [IPC Server handler 0 on 8888] INFO  [org.mh.rpc.task.TaskScheduler] – generate a task: org.mh.rpc.task.JobDetailInfo@1f26605
> 
> TaskRunner:
> 2013-01-20 15:42:26,323 [main] INFO  [org.mh.rpc.task.TaskRunner] – client get jobdetail:JobDetail ‘JobGroup_1.MyJob_1′:  jobClass: ‘org.mh.rpc.quartz.GetSumTask isStateful: false isVolatile: false isDurable: false requestsRecovers: false
> 2013-01-20 15:42:26,329 [main] INFO  [org.mh.rpc.task.TaskRunner] – client get trigger:Trigger ‘Trigger_Group_1.Trigger_1′:  triggerClass: ‘org.quartz.SimpleTrigger isVolatile: false calendar: ‘null’ misfireInstruction: 0 nextFireTime: null
> 2013-01-20 15:42:26,382 [main] INFO  [org.quartz.simpl.SimpleThreadPool] – Job execution threads will use class loader of thread: main
> 2013-01-20 15:42:26,411 [main] INFO  [org.quartz.core.SchedulerSignalerImpl] – Initialized Scheduler Signaller of type: class org.quartz.core.SchedulerSignalerImpl
> 2013-01-20 15:42:26,411 [main] INFO  [org.quartz.core.QuartzScheduler] – Quartz Scheduler v.1.6.5 created.
> 2013-01-20 15:42:26,413 [main] INFO  [org.quartz.simpl.RAMJobStore] – RAMJobStore initialized.
> 2013-01-20 15:42:26,413 [main] INFO  [org.quartz.impl.StdSchedulerFactory] – Quartz scheduler ‘DefaultQuartzScheduler’ initialized from default resource file in Quartz package: ‘quartz.properties’
> 2013-01-20 15:42:26,413 [main] INFO  [org.quartz.impl.StdSchedulerFactory] – Quartz scheduler version: 1.6.5
> 2013-01-20 15:42:26,415 [main] INFO  [org.quartz.core.QuartzScheduler] – Scheduler DefaultQuartzScheduler_$_NON_CLUSTERED started.
> org.mh.rpc.quartz.DemoTask@1b66b06: executing task @Sun Jan 20 15:42:26 CST 2013

上面是一个简单的demo，演示了如何通过RPC将任务调度给节点去执行，对于Quartz来说，任务的形式可以千变万化，关键就看怎么去使用了，分发到多个节点上执行的话，就还需要对任务的信息做更多的封装了。

