---
layout: post
title: Curator操作Zookeeper
categories: Zookeeper
description: Zookeeper
keywords: Zookeeper
---

# 增删改查节点 #

    package qs.curator.base;
    
    import java.util.List;
    import java.util.concurrent.ExecutorService;
    import java.util.concurrent.Executors;
    
    import org.apache.curator.RetryPolicy;
    import org.apache.curator.framework.CuratorFramework;
    import org.apache.curator.framework.CuratorFrameworkFactory;
    import org.apache.curator.framework.api.BackgroundCallback;
    import org.apache.curator.framework.api.CuratorEvent;
    import org.apache.curator.retry.ExponentialBackoffRetry;
    import org.apache.curator.retry.RetryNTimes;
    import org.apache.zookeeper.CreateMode;
    import org.apache.zookeeper.ZooKeeper.States;
    import org.apache.zookeeper.data.Stat;
    
    public class CuratorBase {
    	
    	/** zookeeper地址 */
    	static final String CONNECT_ADDR = "192.168.1.171:2181,192.168.1.172:2181,192.168.1.173:2181";
    	/** session超时时间 */
    	static final int SESSION_OUTTIME = 5000;//ms 
    	
    	public static void main(String[] args) throws Exception {
    		
    		//1 重试策略：初试时间为1s 重试10次
    		RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 10);
    		//2 通过工厂创建连接
    		CuratorFramework cf = CuratorFrameworkFactory.builder()
    					.connectString(CONNECT_ADDR)
    					.retryPolicy(retryPolicy)
    					.sessionTimeoutMs(SESSION_OUTTIME)
    					.connectionTimeoutMs(SESSION_OUTTIME)
    //					.namespace("super")
    					.build();
    		//3 开启连接
    		cf.start();
    		
    //		System.out.println(States.CONNECTED);
    //		System.out.println(cf.getState());
    		
    		// 新加、删除
    		/**
    		//4 建立节点 指定节点类型（不加withMode默认为持久类型节点）、路径、数据内容
    		cf.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath("/super/c1","c1内容".getBytes());
    		//5 删除节点
    		cf.delete().guaranteed().deletingChildrenIfNeeded().forPath("/super");
    		*/
    		// 读取、修改
    		/**
    		//创建节点
    //		cf.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath("/super/c1","c1内容".getBytes());
    //		cf.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath("/super/c2","c2内容".getBytes());
    		//读取节点
    //		String ret1 = new String(cf.getData().forPath("/super/c2"));
    //		System.out.println(ret1);
    	
    		//修改节点
    //		cf.setData().forPath("/super/c2", "修改c2内容".getBytes());
    //		String ret2 = new String(cf.getData().forPath("/super/c2"));
    //		System.out.println(ret2);	
    		*/
    		// 绑定回调函数
    		/**
    		ExecutorService pool = Executors.newCachedThreadPool();
    		cf.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT)
    		.inBackground(new BackgroundCallback() {
    			@Override
    			public void processResult(CuratorFramework cf, CuratorEvent ce) throws Exception {
    				System.out.println("code:" + ce.getResultCode());
    				System.out.println("type:" + ce.getType());
    				System.out.println("线程为:" + Thread.currentThread().getName());
    			}
    		}, pool)
    		.forPath("/super/c3","c3内容".getBytes());
    		Thread.sleep(Integer.MAX_VALUE);
    		*/
    		
    		
    		// 读取子节点getChildren方法 和 判断节点是否存在checkExists方法
    		/**
    		List<String> list = cf.getChildren().forPath("/super");
    		for(String p : list){
    			System.out.println(p);
    		}
    		
    		Stat stat = cf.checkExists().forPath("/super/c3");
    		System.out.println(stat);
    		
    		Thread.sleep(2000);
    		cf.delete().guaranteed().deletingChildrenIfNeeded().forPath("/super");
    		*/
    		
    		
    		//cf.delete().guaranteed().deletingChildrenIfNeeded().forPath("/super");
    		
    	}
    }

# watcher监听节点 #

## CuratorWatcher1 ##


    package qs.curator.watcher;
    
    import java.util.List;
    import java.util.concurrent.ExecutorService;
    import java.util.concurrent.Executors;
    
    import org.apache.curator.RetryPolicy;
    import org.apache.curator.framework.CuratorFramework;
    import org.apache.curator.framework.CuratorFrameworkFactory;
    import org.apache.curator.framework.api.BackgroundCallback;
    import org.apache.curator.framework.api.CuratorEvent;
    import org.apache.curator.framework.recipes.cache.NodeCache;
    import org.apache.curator.framework.recipes.cache.NodeCacheListener;
    import org.apache.curator.retry.ExponentialBackoffRetry;
    import org.apache.zookeeper.CreateMode;
    import org.apache.zookeeper.data.Stat;
    
    public class CuratorWatcher1 {
    	
    	/** zookeeper地址 */
    	static final String CONNECT_ADDR = "192.168.1.171:2181,192.168.1.172:2181,192.168.1.173:2181";
    	/** session超时时间 */
    	static final int SESSION_OUTTIME = 5000;//ms 
    	
    	public static void main(String[] args) throws Exception {
    		
    		//1 重试策略：初试时间为1s 重试10次
    		RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 10);
    		//2 通过工厂创建连接
    		CuratorFramework cf = CuratorFrameworkFactory.builder()
    					.connectString(CONNECT_ADDR)
    					.sessionTimeoutMs(SESSION_OUTTIME)
    					.retryPolicy(retryPolicy)
    					.build();
    		
    		//3 建立连接
    		cf.start();
    		
    		//4 建立一个cache缓存
    		final NodeCache cache = new NodeCache(cf, "/super", false);
    		cache.start(true);
    		cache.getListenable().addListener(new NodeCacheListener() {
    			/**
    			 * <B>方法名称：</B>nodeChanged<BR>
    			 * <B>概要说明：</B>触发事件为创建节点和更新节点，在删除节点的时候并不触发此操作。<BR>
    			 * @see org.apache.curator.framework.recipes.cache.NodeCacheListener#nodeChanged()
    			 */
    			@Override
    			public void nodeChanged() throws Exception {
    				System.out.println("路径为：" + cache.getCurrentData().getPath());
    				System.out.println("数据为：" + new String(cache.getCurrentData().getData()));
    				System.out.println("状态为：" + cache.getCurrentData().getStat());
    				System.out.println("---------------------------------------");
    			}
    		});
    		
    		Thread.sleep(1000);
    		cf.create().forPath("/super", "123".getBytes());
    		
    		Thread.sleep(1000);
    		cf.setData().forPath("/super", "456".getBytes());
    		
    		Thread.sleep(1000);
    		cf.delete().forPath("/super");
    		
    		Thread.sleep(Integer.MAX_VALUE);
    		
    		
    
    	}
    }


## CuratorWatcher2 ##

    package qs.curator.watcher;
    
    import java.util.List;
    import java.util.concurrent.ExecutorService;
    import java.util.concurrent.Executors;
    
    import org.apache.curator.RetryPolicy;
    import org.apache.curator.framework.CuratorFramework;
    import org.apache.curator.framework.CuratorFrameworkFactory;
    import org.apache.curator.framework.api.BackgroundCallback;
    import org.apache.curator.framework.api.CuratorEvent;
    import org.apache.curator.framework.recipes.cache.NodeCache;
    import org.apache.curator.framework.recipes.cache.NodeCacheListener;
    import org.apache.curator.framework.recipes.cache.PathChildrenCache;
    import org.apache.curator.framework.recipes.cache.PathChildrenCacheEvent;
    import org.apache.curator.framework.recipes.cache.PathChildrenCacheListener;
    import org.apache.curator.framework.recipes.cache.PathChildrenCache.StartMode;
    import org.apache.curator.retry.ExponentialBackoffRetry;
    import org.apache.zookeeper.CreateMode;
    import org.apache.zookeeper.data.Stat;
    
    public class CuratorWatcher2 {
    	
    	/** zookeeper地址 */
    	static final String CONNECT_ADDR = "192.168.1.171:2181,192.168.1.172:2181,192.168.1.173:2181";
    	/** session超时时间 */
    	static final int SESSION_OUTTIME = 5000;//ms 
    	
    	public static void main(String[] args) throws Exception {
    		
    		//1 重试策略：初试时间为1s 重试10次
    		RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 10);
    		//2 通过工厂创建连接
    		CuratorFramework cf = CuratorFrameworkFactory.builder()
    					.connectString(CONNECT_ADDR)
    					.sessionTimeoutMs(SESSION_OUTTIME)
    					.retryPolicy(retryPolicy)
    					.build();
    		
    		//3 建立连接
    		cf.start();
    		
    		//4 建立一个PathChildrenCache缓存,第三个参数为是否接受节点数据内容 如果为false则不接受
    		PathChildrenCache cache = new PathChildrenCache(cf, "/super", true);
    		//5 在初始化的时候就进行缓存监听
    		cache.start(StartMode.POST_INITIALIZED_EVENT);
    		cache.getListenable().addListener(new PathChildrenCacheListener() {
    			/**
    			 * <B>方法名称：</B>监听子节点变更<BR>
    			 * <B>概要说明：</B>新建、修改、删除<BR>
    			 * @see org.apache.curator.framework.recipes.cache.PathChildrenCacheListener#childEvent(org.apache.curator.framework.CuratorFramework, org.apache.curator.framework.recipes.cache.PathChildrenCacheEvent)
    			 */
    			@Override
    			public void childEvent(CuratorFramework cf, PathChildrenCacheEvent event) throws Exception {
    				switch (event.getType()) {
    				case CHILD_ADDED:
    					System.out.println("CHILD_ADDED :" + event.getData().getPath());
    					break;
    				case CHILD_UPDATED:
    					System.out.println("CHILD_UPDATED :" + event.getData().getPath());
    					break;
    				case CHILD_REMOVED:
    					System.out.println("CHILD_REMOVED :" + event.getData().getPath());
    					break;
    				default:
    					break;
    				}
    			}
    		});
    
    		//创建本身节点不发生变化
    		cf.create().forPath("/super", "init".getBytes());
    		
    		//添加子节点
    		Thread.sleep(1000);
    		cf.create().forPath("/super/c1", "c1内容".getBytes());
    		Thread.sleep(1000);
    		cf.create().forPath("/super/c2", "c2内容".getBytes());
    		
    		//修改子节点
    		Thread.sleep(1000);
    		cf.setData().forPath("/super/c1", "c1更新内容".getBytes());
    		
    		//删除子节点
    		Thread.sleep(1000);
    		cf.delete().forPath("/super/c2");		
    		
    		//删除本身节点
    		Thread.sleep(1000);
    		cf.delete().deletingChildrenIfNeeded().forPath("/super");
    		
    		Thread.sleep(Integer.MAX_VALUE);
    		
    
    	}
    }


# 集群下的操作zookeeper #

CuratorWatcher.java类

    package qs.curator.cluster;
    
    import java.util.List;
    import java.util.Map;
    import java.util.concurrent.ConcurrentHashMap;
    import java.util.concurrent.CopyOnWriteArrayList;
    import java.util.concurrent.CountDownLatch;
    
    import org.apache.curator.RetryPolicy;
    import org.apache.curator.framework.CuratorFramework;
    import org.apache.curator.framework.CuratorFrameworkFactory;
    import org.apache.curator.framework.recipes.cache.PathChildrenCache;
    import org.apache.curator.framework.recipes.cache.PathChildrenCacheEvent;
    import org.apache.curator.framework.recipes.cache.PathChildrenCacheListener;
    import org.apache.curator.framework.recipes.cache.PathChildrenCache.StartMode;
    import org.apache.curator.retry.ExponentialBackoffRetry;
    import org.apache.zookeeper.CreateMode;
    import org.apache.zookeeper.KeeperException;
    import org.apache.zookeeper.WatchedEvent;
    import org.apache.zookeeper.Watcher;
    import org.apache.zookeeper.Watcher.Event.EventType;
    import org.apache.zookeeper.Watcher.Event.KeeperState;
    import org.apache.zookeeper.ZooDefs.Ids;
    import org.apache.zookeeper.ZooKeeper;
    
    public class CuratorWatcher {
    
    	/** 父节点path */
    	static final String PARENT_PATH = "/super";
    	
    	/** zookeeper服务器地址 */
    	public static final String CONNECT_ADDR = "192.168.1.171:2181,192.168.1.172:2181,192.168.1.173:2181";	/** 定义session失效时间 */
    	
    	public static final int SESSION_TIMEOUT = 30000;
    	
    	public CuratorWatcher() throws Exception{
    		//1 重试策略：初试时间为1s 重试10次
    		RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 10);
    		//2 通过工厂创建连接
    		CuratorFramework cf = CuratorFrameworkFactory.builder()
    					.connectString(CONNECT_ADDR)
    					.sessionTimeoutMs(SESSION_TIMEOUT)
    					.retryPolicy(retryPolicy)
    					.build();
    		//3 建立连接
    		cf.start();
    		
    		//4 创建跟节点
    		if(cf.checkExists().forPath(PARENT_PATH) == null){
    			cf.create().withMode(CreateMode.PERSISTENT).forPath(PARENT_PATH,"super init".getBytes());
    		}
    
    		//4 建立一个PathChildrenCache缓存,第三个参数为是否接受节点数据内容 如果为false则不接受
    		PathChildrenCache cache = new PathChildrenCache(cf, PARENT_PATH, true);
    		//5 在初始化的时候就进行缓存监听
    		cache.start(StartMode.POST_INITIALIZED_EVENT);
    		cache.getListenable().addListener(new PathChildrenCacheListener() {
    			/**
    			 * <B>方法名称：</B>监听子节点变更<BR>
    			 * <B>概要说明：</B>新建、修改、删除<BR>
    			 * @see org.apache.curator.framework.recipes.cache.PathChildrenCacheListener#childEvent(org.apache.curator.framework.CuratorFramework, org.apache.curator.framework.recipes.cache.PathChildrenCacheEvent)
    			 */
    			@Override
    			public void childEvent(CuratorFramework cf, PathChildrenCacheEvent event) throws Exception {
    				switch (event.getType()) {
    				case CHILD_ADDED:
    					System.out.println("CHILD_ADDED :" + event.getData().getPath());
    					System.out.println("CHILD_ADDED :" + new String(event.getData().getData()));
    					break;
    				case CHILD_UPDATED:
    					System.out.println("CHILD_UPDATED :" + event.getData().getPath());
    					System.out.println("CHILD_UPDATED :" + new String(event.getData().getData()));
    					break;
    				case CHILD_REMOVED:
    					System.out.println("CHILD_REMOVED :" + event.getData().getPath());
    					System.out.println("CHILD_REMOVED :" + new String(event.getData().getData()));
    					break;
    				default:
    					break;
    				}
    			}
    		});
    	}
    
    }


Client1.java类

    package qs.curator.cluster;

    public class Client1 {
    
    	public static void main(String[] args) throws Exception{
    		
    		CuratorWatcher watcher = new CuratorWatcher();
    		Thread.sleep(100000000);
    	}
    }
    
Client2.java类

    package qs.curator.cluster;
 
    public class Client1 {
    
    	public static void main(String[] args) throws Exception{
    		
    		CuratorWatcher watcher = new CuratorWatcher();
    		Thread.sleep(100000000);
    	}
    }

Test.java类

    package qs.curator.cluster;
    
    import org.apache.curator.RetryPolicy;
    import org.apache.curator.framework.CuratorFramework;
    import org.apache.curator.framework.CuratorFrameworkFactory;
    import org.apache.curator.retry.ExponentialBackoffRetry;
    import org.apache.zookeeper.CreateMode;
    
    public class Test {
    
    	/** zookeeper地址 */
    	static final String CONNECT_ADDR = "192.168.1.171:2181,192.168.1.172:2181,192.168.1.173:2181";
    	/** session超时时间 */
    	static final int SESSION_OUTTIME = 5000;//ms 
    	
    	public static void main(String[] args) throws Exception {
    		
    		//1 重试策略：初试时间为1s 重试10次
    		RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 10);
    		//2 通过工厂创建连接
    		CuratorFramework cf = CuratorFrameworkFactory.builder()
    					.connectString(CONNECT_ADDR)
    					.sessionTimeoutMs(SESSION_OUTTIME)
    					.retryPolicy(retryPolicy)
    					.build();
    		//3 开启连接
    		cf.start();
    
    		
    		Thread.sleep(3000);
    		System.out.println(cf.getChildren().forPath("/super").get(0));
    		
    		//4 创建节点
    //		Thread.sleep(1000);
    		cf.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath("/super/c1","c1内容".getBytes());
    		Thread.sleep(1000);
    //		cf.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT).forPath("/super/c2","c2内容".getBytes());
    //		Thread.sleep(1000);
    //		
    //		
    //		
    //		//5 读取节点
    //		Thread.sleep(1000);
    //		String ret1 = new String(cf.getData().forPath("/super/c1"));
    //		System.out.println(ret1);
    //
    //		
    //		//6 修改节点
    		Thread.sleep(1000);
    		cf.setData().forPath("/super/c2", "修改的新c2内容".getBytes());
    		String ret2 = new String(cf.getData().forPath("/super/c2"));
    		System.out.println(ret2);	
    //		
    //
    //		
    //		//7 删除节点
    //		Thread.sleep(1000);
    //		cf.delete().forPath("/super/c1");
  
    	}
    }


# Curator提供的分布式锁 #

    package qs.curator.lock;
    
    import java.text.SimpleDateFormat;
    import java.util.Date;
    import java.util.concurrent.CountDownLatch;
    import java.util.concurrent.locks.ReentrantLock;
    
    import org.apache.curator.RetryPolicy;
    import org.apache.curator.framework.CuratorFramework;
    import org.apache.curator.framework.CuratorFrameworkFactory;
    import org.apache.curator.framework.recipes.locks.InterProcessMutex;
    import org.apache.curator.retry.ExponentialBackoffRetry;
    
    
    public class Lock2 {
    
    	/** zookeeper地址 */
    	static final String CONNECT_ADDR = "192.168.1.171:2181,192.168.1.172:2181,192.168.1.173:2181";
    	/** session超时时间 */
    	static final int SESSION_OUTTIME = 5000;//ms 
    	
    	static int count = 10;
    	public static void genarNo(){
    		try {
    			count--;
    			System.out.println(count);
    		} finally {
    		
    		}
    	}
    	
    	public static void main(String[] args) throws Exception {
    		
    		//1 重试策略：初试时间为1s 重试10次
    		RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 10);
    		//2 通过工厂创建连接
    		CuratorFramework cf = CuratorFrameworkFactory.builder()
    					.connectString(CONNECT_ADDR)
    					.sessionTimeoutMs(SESSION_OUTTIME)
    					.retryPolicy(retryPolicy)
    //					.namespace("super")
    					.build();
    		//3 开启连接
    		cf.start();
    		
    		//4 分布式锁
    		final InterProcessMutex lock = new InterProcessMutex(cf, "/super");
    		//final ReentrantLock reentrantLock = new ReentrantLock();
    		final CountDownLatch countdown = new CountDownLatch(1);
    		
    		for(int i = 0; i < 10; i++){
    			new Thread(new Runnable() {
    				@Override
    				public void run() {
    					try {
    						countdown.await();
    						//加锁
    						lock.acquire();
    						//reentrantLock.lock();
    						//-------------业务处理开始
    						//genarNo();
    						SimpleDateFormat sdf = new SimpleDateFormat("HH:mm:ss|SSS");
    						System.out.println(sdf.format(new Date()));
    						//System.out.println(System.currentTimeMillis());
    						//-------------业务处理结束
    					} catch (Exception e) {
    						e.printStackTrace();
    					} finally {
    						try {
    							//释放
    							lock.release();
    							//reentrantLock.unlock();
    						} catch (Exception e) {
    							e.printStackTrace();
    						}
    					}
    				}
    			},"t" + i).start();
    		}
    		Thread.sleep(100);
    		countdown.countDown();	 
    	}
    }


# Curator提供的栅栏Barrier #

CuratorBarrier1.java

    package qs.curator.barrier;
    
    import java.util.Random;
    
    import org.apache.curator.RetryPolicy;
    import org.apache.curator.framework.CuratorFramework;
    import org.apache.curator.framework.CuratorFrameworkFactory;
    import org.apache.curator.framework.recipes.barriers.DistributedDoubleBarrier;
    import org.apache.curator.retry.ExponentialBackoffRetry;
    
    public class CuratorBarrier1 {
    
    	/** zookeeper地址 */
    	static final String CONNECT_ADDR = "192.168.1.171:2181,192.168.1.172:2181,192.168.1.173:2181";
    	/** session超时时间 */
    	static final int SESSION_OUTTIME = 5000;//ms 
    	
    	public static void main(String[] args) throws Exception {
    		
    		
    		
    		for(int i = 0; i < 5; i++){
    			new Thread(new Runnable() {
    				@Override
    				public void run() {
    					try {
    						RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 10);
    						CuratorFramework cf = CuratorFrameworkFactory.builder()
    									.connectString(CONNECT_ADDR)
    									.retryPolicy(retryPolicy)
    									.build();
    						cf.start();
    						DistributedDoubleBarrier barrier = new DistributedDoubleBarrier(cf, "/super", 5);
    						Thread.sleep(1000 * (new Random()).nextInt(3)); 
    						System.out.println(Thread.currentThread().getName() + "已经准备");
    						barrier.enter();
    						System.out.println("同时开始运行...");
    						Thread.sleep(1000 * (new Random()).nextInt(3));
    						System.out.println(Thread.currentThread().getName() + "运行完毕");
    						barrier.leave();
    						System.out.println("同时退出运行...");
    						
    
    					} catch (Exception e) {
    						e.printStackTrace();
    					}
    				}
    			},"t" + i).start();
    		}
    	}
    }

CuratorBarrier2.java

    package qs.curator.barrier;
    
    import java.text.SimpleDateFormat;
    import java.util.Date;
    
    import org.apache.curator.RetryPolicy;
    import org.apache.curator.framework.CuratorFramework;
    import org.apache.curator.framework.CuratorFrameworkFactory;
    import org.apache.curator.framework.recipes.atomic.AtomicValue;
    import org.apache.curator.framework.recipes.atomic.DistributedAtomicInteger;
    import org.apache.curator.framework.recipes.barriers.DistributedBarrier;
    import org.apache.curator.framework.recipes.barriers.DistributedDoubleBarrier;
    import org.apache.curator.framework.recipes.queue.DistributedQueue;
    import org.apache.curator.retry.ExponentialBackoffRetry;
    import org.apache.curator.retry.RetryNTimes;
    
    public class CuratorBarrier2 {
    
    	/** zookeeper地址 */
    	static final String CONNECT_ADDR = "192.168.1.171:2181,192.168.1.172:2181,192.168.1.173:2181";
    	/** session超时时间 */
    	static final int SESSION_OUTTIME = 5000;//ms 
    	
    	static DistributedBarrier barrier = null;
    	
    	public static void main(String[] args) throws Exception {
    		
    		
    		
    		for(int i = 0; i < 5; i++){
    			new Thread(new Runnable() {
    				@Override
    				public void run() {
    					try {
    						RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 10);
    						CuratorFramework cf = CuratorFrameworkFactory.builder()
    									.connectString(CONNECT_ADDR)
    									.sessionTimeoutMs(SESSION_OUTTIME)
    									.retryPolicy(retryPolicy)
    									.build();
    						cf.start();
    						barrier = new DistributedBarrier(cf, "/super");
    						System.out.println(Thread.currentThread().getName() + "设置barrier!");
    						barrier.setBarrier();	//设置
    						barrier.waitOnBarrier();	//等待
    						System.out.println("---------开始执行程序----------");
    					} catch (Exception e) {
    						e.printStackTrace();
    					}
    				}
    			},"t" + i).start();
    		}
    
    		Thread.sleep(5000);
    		barrier.removeBarrier();	//释放
    		
    		
    	}
    }


# Curator提供的AtomicInteger #

    package qs.curator.atomicinteger;
    
    import org.apache.curator.RetryPolicy;
    import org.apache.curator.framework.CuratorFramework;
    import org.apache.curator.framework.CuratorFrameworkFactory;
    import org.apache.curator.framework.recipes.atomic.AtomicValue;
    import org.apache.curator.framework.recipes.atomic.DistributedAtomicInteger;
    import org.apache.curator.retry.ExponentialBackoffRetry;
    import org.apache.curator.retry.RetryNTimes;
    
    public class CuratorAtomicInteger {
    
    	/** zookeeper地址 */
    	static final String CONNECT_ADDR = "192.168.1.171:2181,192.168.1.172:2181,192.168.1.173:2181";
    	/** session超时时间 */
    	static final int SESSION_OUTTIME = 5000;//ms 
    	
    	public static void main(String[] args) throws Exception {
    		
    		//1 重试策略：初试时间为1s 重试10次
    		RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 10);
    		//2 通过工厂创建连接
    		CuratorFramework cf = CuratorFrameworkFactory.builder()
    					.connectString(CONNECT_ADDR)
    					.sessionTimeoutMs(SESSION_OUTTIME)
    					.retryPolicy(retryPolicy)
    					.build();
    		//3 开启连接
    		cf.start();
    		//cf.delete().forPath("/super");
    		
    
    		//4 使用DistributedAtomicInteger
    		DistributedAtomicInteger atomicIntger = 
    				new DistributedAtomicInteger(cf, "/super", new RetryNTimes(3, 1000));
    		
    		AtomicValue<Integer> value = atomicIntger.add(1);
    		System.out.println(value.succeeded());
    		System.out.println(value.postValue());	//最新值
    		System.out.println(value.preValue());	//原始值
    		
    	}
    }
