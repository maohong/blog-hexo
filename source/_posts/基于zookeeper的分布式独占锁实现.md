---
title: 基于zookeeper的分布式独占锁实现
date: 2014-05-13 21:05:01
tags:
- zookeeper
- 分布式应用
- 分布式协调
categories:
- 分布式应用
---

背景
--

在分布式系统中，经常遇到这样一种场景：选举一个节点执行某一个任务，当此节点宕机后，其他节点可以接管并继续执行这个任务。由于各个节点运行的代码是一样的，彼此之间也是平等的，各个节点如何可以知道自己是否可以执行这个任务呢？当有节点宕机时，又如何判断自己是否可以接管任务呢？在我们的分布式任务调度系统中，需要选取调度器集群中的一个节点进行轮询任务状态，这里使用了zookeeper来实现一个统一的分布式锁，从而选出轮询节点。

原理
--

如图所示，每台服务器启动后，都在同一目录下建一个临时顺序节点（EPHEMERAL_SEQUENTIAL），并获取此目录下的所有节点信息，如果自己的序号是最小的，就认为获取到了锁，可以执行任务。若自己的节点不是最小的，就认为自己没有获取到锁，不执行任务，同时，在比自己小1个序号的节点上增加监听。当比自己小1个序号的节点发生变化的时候，再次检查自己是否是最小序号的节点，如果是则获取锁，否则继续监听比自己小1个序号的节点。

![](https://raw.githubusercontent.com/maohong/picture/master/20140502-%E5%9F%BA%E4%BA%8Ezookeeper%E7%9A%84%E5%88%86%E5%B8%83%E5%BC%8F%E7%8B%AC%E5%8D%A0%E9%94%81%E5%AE%9E%E7%8E%B0/1.jpg)

实现
--

以下是一个demo实现程序：

```java
public class DistributedExclusiveLock implements Watcher
{
	private ZooKeeper zk;
	private String lockDir = "/testlock";//锁节点所在zk的目录
	private String lockSymbol = "_lock_";//锁节点标志
	private String lockName;//锁节点前缀，构造锁时由外部传入
	private String waitNodePath;//等待的前一个锁的节点名称
	private String myNodePath;//当前锁
	private CountDownLatch latch;//计数器
	private String threadId;

	/**
	 * 创建分布式锁
	 * @param lockName 竞争资源标志,lockName中不能包含单词lock
	 * @throws Exception
	 */
	public DistributedExclusiveLock(String zkServers, String lockName) throws Exception
	{
		//简单校验lockDir路径
		if (!lockDir.startsWith("/"))
			throw new Exception("LockDir Path must start with / character! lockDir=" + lockDir);
		if (lockDir.endsWith("/"))
			throw new Exception("LockDir Path must not end with / character! lockDir=" + lockDir);

		this.lockName = lockName;
		this.threadId = getThreadId();
		// 创建一个与服务器的连接
		try
		{
			zk = new ZooKeeper(zkServers, 3000, this);
			createLockDirIfNecessary(lockDir);
		} catch (Exception e) {
			throw new Exception("Error while initializing DistributedExclusiveLock!" + e.getMessage(), e);
		}
	}

	private String getThreadId()
	{
		return "Thread-" + Thread.currentThread().getId();
	}

	/**
	 * 在zk上建立lock目录，如果目录不存在，逐级创建节点
	 */
	private synchronized void createLockDirIfNecessary(String zkDir) throws KeeperException, InterruptedException
	{
		//zkDir是一级节点，如/cloudscheduler
		if (zkDir.indexOf("/") == zkDir.lastIndexOf("/"))
		{
			Stat stat = zk.exists(zkDir, false);
			if(stat == null){
				// 创建一级节点
				zk.create(zkDir, new byte[0], Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
			}
		}
		else	//zkDir非一级节点
		{
			String parentDir = zkDir.substring(0, zkDir.lastIndexOf("/"));
			if (zk.exists(parentDir, false) != null)
			{	//如果父节点存在，建当前节点
				Stat stat = zk.exists(zkDir, false);
				if(stat == null){
					// 创建非一级节点
					zk.create(zkDir, new byte[0], Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
				}
			}
			else
			{	//否则，先建父节点，再建当前节点
				createLockDirIfNecessary(parentDir);
				createLockDirIfNecessary(zkDir);
			}
		}
	}

	/**
	 * zookeeper节点的监视器
	 */
	@Override
	public void process(WatchedEvent event)
	{
		if (event.getType() == EventType.NodeDeleted)
		{
			if (this.latch!=null)
				this.latch.countDown();
			try
			{
				List<String> childrenNodes = zk.getChildren(lockDir, false);
				// 排序
				Collections.sort(childrenNodes);
				System.out.println("Node: " + event.getPath()
						+ " change event is deleted! Current locked nodes:\n\t"
						+ StringUtils.join(childrenNodes,"\n\t"));
			}
			catch (KeeperException e)
			{
				e.printStackTrace();
			}
			catch (InterruptedException e)
			{
				e.printStackTrace();
			}
		}
    }

	public boolean tryLock()
	{
		try
		{
			if(tryLockInner())
				return true;
			else
				return waitForLockInner(waitNodePath);
		}
		catch (Exception e)
		{
			e.printStackTrace();
			return false;
		}
	}

	private boolean tryLockInner() throws Exception
	{
		try
		{
			if(lockName.contains(lockSymbol))
				throw new Exception("lockName can not contains " + lockSymbol);
			//创建临时子节点
			myNodePath = zk.create(lockDir + "/" + lockName + lockSymbol, new byte[0], Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
			System.out.println(threadId + " created " + myNodePath);
			//取出所有子节点
			List<String> subNodes = zk.getChildren(lockDir, false);
			//取出所有lockName的锁
			List<String> lockedNodes = new ArrayList<String>();
			for (String node : subNodes) {
				String nodePrefix = node.split(lockSymbol)[0];
				if(nodePrefix.equals(lockName)){//对锁名做个判断，前缀相同即为同一组锁
					lockedNodes.add(node);
				}
			}
			Collections.sort(lockedNodes);
			System.out.println("Current locked nodes: \n\t" + StringUtils.join(lockedNodes, "\n\t"));
			if(myNodePath.equals(lockDir + "/" + lockedNodes.get(0))){
				//如果是最小的节点,则表示取得锁
	            return true;
	        }
			//如果不是最小的节点，找到比自己小1的节点，在List中的位置是自己的前一位
			String myZnodeName = myNodePath.substring(myNodePath.lastIndexOf("/") + 1);
			waitNodePath = lockDir + "/" + lockedNodes.get(lockedNodes.indexOf(myZnodeName)-1);
		}
		catch (KeeperException e)
		{
			e.printStackTrace();
		}
		catch (InterruptedException e)
		{
			e.printStackTrace();
		}
		return false;
	}

	private boolean waitForLockInner(String waitPath) throws InterruptedException, KeeperException {
        Stat stat = zk.exists(waitPath, true);
        //判断比自己小一个数的节点是否存在,如果存在则需等待锁,同时注册监听
        if (stat != null)
        {
        	System.out.println(threadId + " waiting for " + waitPath);
        	this.latch = new CountDownLatch(1);
        	this.latch.await(); //不加超时时间，无限等待
        	//
        	//waiting
        	//Zzzzz...
        	//still waiting
        	//
        	// 探测到节点变化，刷新节点信息
        	this.latch = null;
        	try
			{
				// 确认myNodePath是否真的是列表中的最小节点
				List<String> childrenNodes = zk.getChildren(lockDir, false);
				// 排序
				Collections.sort(childrenNodes);
				if(myNodePath.equals(lockDir + "/" + childrenNodes.get(0)))
					return true;
				else
				{
				    // 说明waitNodePath是由于出现异常而挂掉的 , 更新waitNodePath
					String thisNodeName = myNodePath.substring(myNodePath.lastIndexOf("/") + 1);
					int index = childrenNodes.indexOf(thisNodeName);
					waitNodePath = lockDir + "/" + childrenNodes.get(index - 1);
					//重新等待锁
					return waitForLockInner(waitNodePath);
				}
			}
			catch (KeeperException e)
			{
				e.printStackTrace();
				return false;
			}
			catch (InterruptedException e)
			{
				e.printStackTrace();
				return false;
			}
        }
        return true;
    }

	public void unlock() throws Exception
	{
		try
		{
			System.out.println(threadId + " unlock " + myNodePath);
			zk.delete(myNodePath,-1);
			myNodePath = null;
			zk.close();
		}
		catch (InterruptedException e)
		{
			throw new Exception("Error while releasing lock! " + e.getMessage(), e);
		}
		catch (KeeperException e)
		{
			throw new Exception("Error while releasing lock! " + e.getMessage(), e);
		}
	}

	public static void main(String[] args) throws Exception
	{
	        //一个简单的测试
		List<Thread> workers = new ArrayList<Thread>(10);
		for (int i=1; i<10; ++i)
		{
			Thread thread = new Thread(new Runnable()
			{
				String zk = "10.12.10.169:2181,10.12.139.141:2181";
				@Override
				public void run()
				{
					try
					{
						DistributedExclusiveLock lock = new DistributedExclusiveLock(zk, "zkLock");
						if (lock.tryLock());
						{
							String tid = "Thread-" + Thread.currentThread().getId();
							int time = new Random().nextInt(5000);
							System.out.println(tid + " gets lock and is working, sleep for " + time + " ms");
							Thread.sleep(time);
							lock.unlock();
							System.out.println(tid + " releases lock");
						}
					}
					catch (Exception e)
					{
						e.printStackTrace();
					}
				}
			});
			thread.setDaemon(true);
			workers.add(thread);
		}

		for (Thread t : workers)
		{
			t.start();
		}
		Thread.sleep(100000);
	}
}
```

