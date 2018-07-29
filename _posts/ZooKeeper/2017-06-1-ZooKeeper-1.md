---
layout: post
title: java操作Zookeeper
categories: Zookeeper
description: Zookeeper
keywords: Zookeeper
---

# 增删改查节点 #
    
    package qs.zookeeper.base;
    
    import java.util.concurrent.CountDownLatch;
    
    import org.apache.zookeeper.CreateMode;
    import org.apache.zookeeper.WatchedEvent;
    import org.apache.zookeeper.Watcher;
    import org.apache.zookeeper.Watcher.Event.EventType;
    import org.apache.zookeeper.ZooKeeper;
    import org.apache.zookeeper.Watcher.Event.KeeperState;
    import org.apache.zookeeper.ZooDefs.Ids;
       
    public class ZookeeperBase {
    
    	/** zookeeper地址 */
    	static final String CONNECT_ADDR = "192.168.80.88:2181,192.168.80.87:2181,192.168.80.86:2181";
    	/** session超时时间 */
    	static final int SESSION_OUTTIME = 2000;//ms 
    	/** 信号量，阻塞程序执行，用于等待zookeeper连接成功，发送成功信号 */
    	static final CountDownLatch connectedSemaphore = new CountDownLatch(1);
    	
    	public static void main(String[] args) throws Exception{
    		
    		ZooKeeper zk = new ZooKeeper(CONNECT_ADDR, SESSION_OUTTIME, new Watcher(){
    			@Override
    			public void process(WatchedEvent event) {
    				//获取事件的状态
    				KeeperState keeperState = event.getState();
    				EventType eventType = event.getType();
    				//如果是建立连接
    				//KeeperState.ConnectedReadOnly
    				//``KeeperState.Disconnected  
    				//``KeeperState.Expired
    				//KeeperState.NoSyncConnected
    				//``KeeperState.SyncConnected
    				//KeeperState.Unknown
    				if(KeeperState.SyncConnected == keeperState){
    					if(EventType.None == eventType){
    						//如果建立连接成功，则发送信号量，让后续阻塞程序向下执行
    						connectedSemaphore.countDown();
    						System.out.println("zk 建立连接");
    					}
    				}
    			}
    		});
    
    		//进行阻塞
    		connectedSemaphore.await();
    		
    		System.out.println("..");
    		//创建父节点
    		zk.create("/testRoot", "testRoot".getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT,);
    		
    		//创建子节点
    		zk.create("/testRoot/children", "children data".getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
    		
    		//获取节点洗信息
    		
    		byte[] data = zk.getData("/testRoot", false, null);
    		System.out.println(new String(data));
    		System.out.println(zk.getChildren("/testRoot", false));
    		
    		//修改节点的值
    		zk.setData("/testRoot", "modify data root".getBytes(), -1);
    		byte[] data = zk.getData("/testRoot", false, null);
    		System.out.println(new String(data));		
    		
    		//判断节点是否存在
    		System.out.println(zk.exists("/testRoot/children", false));
    		//删除节点
    		zk.delete("/testRoot/children", -1);
    		System.out.println(zk.exists("/testRoot/children", false));
    		
    		zk.close();	
    	}	
    }
    

# watcher监听节点 #

    package qs.zookeeper.watcher;
    
    import java.util.List;
    import java.util.concurrent.CountDownLatch;
    import java.util.concurrent.atomic.AtomicInteger;
    
    import org.apache.zookeeper.CreateMode;
    import org.apache.zookeeper.WatchedEvent;
    import org.apache.zookeeper.Watcher;
    import org.apache.zookeeper.Watcher.Event.EventType;
    import org.apache.zookeeper.Watcher.Event.KeeperState;
    import org.apache.zookeeper.ZooDefs.Ids;
    import org.apache.zookeeper.ZooKeeper;
    import org.apache.zookeeper.data.Stat;
    
    /**
     * Zookeeper Wathcher 
     * 本类就是一个Watcher类（实现了org.apache.zookeeper.Watcher类）
     */
    public class ZooKeeperWatcher implements Watcher {
    
    	/** 定义原子变量 */
    	AtomicInteger seq = new AtomicInteger();
    	/** 定义session失效时间 */
    	private static final int SESSION_TIMEOUT = 10000;
    	/** zookeeper服务器地址 */
    	private static final String CONNECTION_ADDR = "192.168.80.88:2181";
    	/** zk父路径设置 */
    	private static final String PARENT_PATH = "/testWatch";
    	/** zk子路径设置 */
    	private static final String CHILDREN_PATH = "/testWatch/children";
    	/** 进入标识 */
    	private static final String LOG_PREFIX_OF_MAIN = "【Main】";
    	/** zk变量 */
    	private ZooKeeper zk = null;
    	/** 信号量设置，用于等待zookeeper连接建立之后 通知阻塞程序继续向下执行 */
    	private CountDownLatch connectedSemaphore = new CountDownLatch(1);
    
    	/**
    	 * 创建ZK连接
    	 * @param connectAddr ZK服务器地址列表
    	 * @param sessionTimeout Session超时时间
    	 */
    	public void createConnection(String connectAddr, int sessionTimeout) {
    		this.releaseConnection();
    		try {
    			zk = new ZooKeeper(connectAddr, sessionTimeout, this);
    			System.out.println(LOG_PREFIX_OF_MAIN + "开始连接ZK服务器");
    			connectedSemaphore.await();
    		} catch (Exception e) {
    			e.printStackTrace();
    		}
    	}
    
    	/**
    	 * 关闭ZK连接
    	 */
    	public void releaseConnection() {
    		if (this.zk != null) {
    			try {
    				this.zk.close();
    			} catch (InterruptedException e) {
    				e.printStackTrace();
    			}
    		}
    	}
    
    	/**
    	 * 创建节点
    	 * @param path 节点路径
    	 * @param data 数据内容
    	 * @return 
    	 */
    	public boolean createPath(String path, String data) {
    		try {
    			//设置监控(由于zookeeper的监控都是一次性的所以 每次必须设置监控)
    			this.zk.exists(path, true);
    			System.out.println(LOG_PREFIX_OF_MAIN + "节点创建成功, Path: " + 
    							   this.zk.create(	/**路径*/ 
    									   			path, 
    									   			/**数据*/
    									   			data.getBytes(), 
    									   			/**所有可见*/
    								   				Ids.OPEN_ACL_UNSAFE, 
    								   				/**永久存储*/
    								   				CreateMode.PERSISTENT ) + 	
    							   ", content: " + data);
    		} catch (Exception e) {
    			e.printStackTrace();
    			return false;
    		}
    		return true;
    	}
    
    	/**
    	 * 读取指定节点数据内容
    	 * @param path 节点路径
    	 * @return
    	 */
    	public String readData(String path, boolean needWatch) {
    		try {
    			return new String(this.zk.getData(path, needWatch, null));
    		} catch (Exception e) {
    			e.printStackTrace();
    			return "";
    		}
    	}
    
    	/**
    	 * 更新指定节点数据内容
    	 * @param path 节点路径
    	 * @param data 数据内容
    	 * @return
    	 */
    	public boolean writeData(String path, String data) {
    		try {
    			System.out.println(LOG_PREFIX_OF_MAIN + "更新数据成功，path：" + path + ", stat: " +
    								this.zk.setData(path, data.getBytes(), -1));
    		} catch (Exception e) {
    			e.printStackTrace();
    		}
    		return false;
    	}
    
    	/**
    	 * 删除指定节点
    	 * 
    	 * @param path
    	 *节点path
    	 */
    	public void deleteNode(String path) {
    		try {
    			this.zk.delete(path, -1);
    			System.out.println(LOG_PREFIX_OF_MAIN + "删除节点成功，path：" + path);
    		} catch (Exception e) {
    			e.printStackTrace();
    		}
    	}
    
    	/**
    	 * 判断指定节点是否存在
    	 * @param path 节点路径
    	 */
    	public Stat exists(String path, boolean needWatch) {
    		try {
    			return this.zk.exists(path, needWatch);
    		} catch (Exception e) {
    			e.printStackTrace();
    			return null;
    		}
    	}
    
    	/**
    	 * 获取子节点
    	 * @param path 节点路径
    	 */
    	private List<String> getChildren(String path, boolean needWatch) {
    		try {
    			return this.zk.getChildren(path, needWatch);
    		} catch (Exception e) {
    			e.printStackTrace();
    			return null;
    		}
    	}
    
    	/**
    	 * 删除所有节点
    	 */
    	public void deleteAllTestPath() {
    		if(this.exists(CHILDREN_PATH, false) != null){
    			this.deleteNode(CHILDREN_PATH);
    		}
    		if(this.exists(PARENT_PATH, false) != null){
    			this.deleteNode(PARENT_PATH);
    		}		
    	}
    	
    	/**
    	 * 收到来自Server的Watcher通知后的处理。
    	 */
    	@Override
    	public void process(WatchedEvent event) {
    		
    		System.out.println("进入 process 。。。。。event = " + event);
    		
    		try {
    			Thread.sleep(200);
    		} catch (InterruptedException e) {
    			e.printStackTrace();
    		}
    		
    		if (event == null) {
    			return;
    		}
    		
    		// 连接状态
    		KeeperState keeperState = event.getState();
    		// 事件类型
    		EventType eventType = event.getType();
    		// 受影响的path
    		String path = event.getPath();
    		
    		String logPrefix = "【Watcher-" + this.seq.incrementAndGet() + "】";
    
    		System.out.println(logPrefix + "收到Watcher通知");
    		System.out.println(logPrefix + "连接状态:\t" + keeperState.toString());
    		System.out.println(logPrefix + "事件类型:\t" + eventType.toString());
    
    		if (KeeperState.SyncConnected == keeperState) {
    			// 成功连接上ZK服务器
    			if (EventType.None == eventType) {
    				System.out.println(logPrefix + "成功连接上ZK服务器");
    				connectedSemaphore.countDown();
    			} 
    			//创建节点
    			else if (EventType.NodeCreated == eventType) {
    				System.out.println(logPrefix + "节点创建");
    				try {
    					Thread.sleep(100);
    				} catch (InterruptedException e) {
    					e.printStackTrace();
    				}
    				this.exists(path, true);
    			} 
    			//更新节点
    			else if (EventType.NodeDataChanged == eventType) {
    				System.out.println(logPrefix + "节点数据更新");
    				System.out.println("我看看走不走这里........");
    				try {
    					Thread.sleep(100);
    				} catch (InterruptedException e) {
    					e.printStackTrace();
    				}
    				System.out.println(logPrefix + "数据内容: " + this.readData(PARENT_PATH, true));
    			} 
    			//更新子节点
    			else if (EventType.NodeChildrenChanged == eventType) {
    				System.out.println(logPrefix + "子节点变更");
    				try {
    					Thread.sleep(3000);
    				} catch (InterruptedException e) {
    					e.printStackTrace();
    				}
    				System.out.println(logPrefix + "子节点列表：" + this.getChildren(PARENT_PATH, true));
    			} 
    			//删除节点
    			else if (EventType.NodeDeleted == eventType) {
    				System.out.println(logPrefix + "节点 " + path + " 被删除");
    			}
    			else ;
    		} 
    		else if (KeeperState.Disconnected == keeperState) {
    			System.out.println(logPrefix + "与ZK服务器断开连接");
    		} 
    		else if (KeeperState.AuthFailed == keeperState) {
    			System.out.println(logPrefix + "权限检查失败");
    		} 
    		else if (KeeperState.Expired == keeperState) {
    			System.out.println(logPrefix + "会话失效");
    		}
    		else ;
    
    		System.out.println("--------------------------------------------");
    
    	}
    
    	/**
    	 * <B>方法名称：</B>测试zookeeper监控<BR>
    	 * <B>概要说明：</B>主要测试watch功能<BR>
    	 * @param args
    	 * @throws Exception
    	 */
    	public static void main(String[] args) throws Exception {
    
    		//建立watcher
    		ZooKeeperWatcher zkWatch = new ZooKeeperWatcher();
    		//创建连接
    		zkWatch.createConnection(CONNECTION_ADDR, SESSION_TIMEOUT);
    		//System.out.println(zkWatch.zk.toString());
    		
    		Thread.sleep(1000);
    		
    		// 清理节点
    		zkWatch.deleteAllTestPath();
    		
    		if (zkWatch.createPath(PARENT_PATH, System.currentTimeMillis() + "")) {
    			
    			Thread.sleep(1000);
    			
    			
    			// 读取数据
    			System.out.println("---------------------- read parent ----------------------------");
    			//zkWatch.readData(PARENT_PATH, true);
    			
    			// 读取子节点
    			System.out.println("---------------------- read children path ----------------------------");
    			zkWatch.getChildren(PARENT_PATH, true);
    
    			// 更新数据
    			zkWatch.writeData(PARENT_PATH, System.currentTimeMillis() + "");
    			
    			Thread.sleep(1000);
    			
    			// 创建子节点
    			zkWatch.createPath(CHILDREN_PATH, System.currentTimeMillis() + "");
    			
    			Thread.sleep(1000);
    			
    			zkWatch.writeData(CHILDREN_PATH, System.currentTimeMillis() + "");
    		}
    		
    		Thread.sleep(50000);
    		// 清理节点
    		zkWatch.deleteAllTestPath();
    		Thread.sleep(1000);
    		zkWatch.releaseConnection();
    	}
    
    }

# 集群下的zookeeper #

ZKWatcher.java类

    package qs.zookeeper.cluster;
    
    import java.util.List;
    import java.util.Map;
    import java.util.concurrent.ConcurrentHashMap;
    import java.util.concurrent.CopyOnWriteArrayList;
    import java.util.concurrent.CountDownLatch;
    
    import org.apache.zookeeper.CreateMode;
    import org.apache.zookeeper.KeeperException;
    import org.apache.zookeeper.WatchedEvent;
    import org.apache.zookeeper.Watcher;
    import org.apache.zookeeper.Watcher.Event.EventType;
    import org.apache.zookeeper.Watcher.Event.KeeperState;
    import org.apache.zookeeper.ZooDefs.Ids;
    import org.apache.zookeeper.ZooKeeper;
    
    public class ZKWatcher implements Watcher {
    
    	/** zk变量 */
    	private ZooKeeper zk = null;
    	
    	/** 父节点path */
    	static final String PARENT_PATH = "/super";
    	
    	/** 信号量设置，用于等待zookeeper连接建立之后 通知阻塞程序继续向下执行 */
    	private CountDownLatch connectedSemaphore = new CountDownLatch(1);
    	
    	private List<String> cowaList = new CopyOnWriteArrayList<String>();
    	
    	
    	/** zookeeper服务器地址 */
    	public static final String CONNECTION_ADDR = "192.168.80.88:2181,192.168.80.87:2181,192.168.80.86:2181";
    	/** 定义session失效时间 */
    	public static final int SESSION_TIMEOUT = 30000;
    	
    	public ZKWatcher() throws Exception{
    		zk = new ZooKeeper(CONNECTION_ADDR, SESSION_TIMEOUT, this);
    		System.out.println("开始连接ZK服务器");
    		connectedSemaphore.await();
    	}
    
    
    	@Override
    	public void process(WatchedEvent event) {
    		// 连接状态
    		KeeperState keeperState = event.getState();
    		// 事件类型
    		EventType eventType = event.getType();
    		// 受影响的path
    		String path = event.getPath();
    		System.out.println("受影响的path : " + path);
    		
    		
    		if (KeeperState.SyncConnected == keeperState) {
    			// 成功连接上ZK服务器
    			if (EventType.None == eventType) {
    				System.out.println("成功连接上ZK服务器");
    				connectedSemaphore.countDown();
    				try {
    					if(this.zk.exists(PARENT_PATH, false) == null){
    						this.zk.create(PARENT_PATH, "root".getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);		 
    					}
    					List<String> paths = this.zk.getChildren(PARENT_PATH, true);
    					for (String p : paths) {
    						System.out.println(p);
    						this.zk.exists(PARENT_PATH + "/" + p, true);
    					}
    				} catch (KeeperException | InterruptedException e) {
    					e.printStackTrace();
    				}		
    			} 
    			//创建节点
    			else if (EventType.NodeCreated == eventType) {
    				System.out.println("节点创建");
    				try {
    					this.zk.exists(path, true);
    				} catch (KeeperException | InterruptedException e) {
    					e.printStackTrace();
    				}
    			} 
    			//更新节点
    			else if (EventType.NodeDataChanged == eventType) {
    				System.out.println("节点数据更新");
    				try {
    					//update nodes  call function
    					this.zk.exists(path, true);
    				} catch (KeeperException | InterruptedException e) {
    					e.printStackTrace();
    				}
    			} 
    			//更新子节点
    			else if (EventType.NodeChildrenChanged == eventType) {
    				System.out.println("子节点 ... 变更");
    				try {
    					List<String> paths = this.zk.getChildren(path, true);
    					if(paths.size() >= cowaList.size()){
    						paths.removeAll(cowaList);
    						for(String p : paths){
    							this.zk.exists(path + "/" + p, true);
    							//this.zk.getChildren(path + "/" + p, true);
    							System.out.println("这个是新增的子节点 : " + path + "/" + p);
    							//add new nodes  call function
    						}
    						cowaList.addAll(paths);
    					} else {
    						cowaList = paths;
    					}
    					System.out.println("cowaList: " + cowaList.toString());
    					System.out.println("paths: " + paths.toString());
    					
    				} catch (KeeperException | InterruptedException e) {
    					e.printStackTrace();
    				}
    			} 
    			//删除节点
    			else if (EventType.NodeDeleted == eventType) {
    				System.out.println("节点 " + path + " 被删除");
    				try {
    					//delete nodes  call function
    					this.zk.exists(path, true);
    				} catch (KeeperException | InterruptedException e) {
    					e.printStackTrace();
    				}
    			}
    			else ;
    		} 
    		else if (KeeperState.Disconnected == keeperState) {
    			System.out.println("与ZK服务器断开连接");
    		} 
    		else if (KeeperState.AuthFailed == keeperState) {
    			System.out.println("权限检查失败");
    		} 
    		else if (KeeperState.Expired == keeperState) {
    			System.out.println("会话失效");
    		}
    		else ;
    
    		System.out.println("--------------------------------------------");
    	}
    	
    	
    
    }

Client1.java类

    package qs.zookeeper.cluster;
    
    import bjsxt.zookeeper.cluster.ZKWatcher;
    
    public class Client1 {
    
    	public static void main(String[] args) throws Exception{
    		
    		ZKWatcher myWatcher = new ZKWatcher();
    		Thread.sleep(100000000);
    	}
    }

Client2.java类

    package qs.zookeeper.cluster;
    
    import bjsxt.zookeeper.cluster.ZKWatcher;
    
    public class Client1 {
    
    	public static void main(String[] args) throws Exception{
    		
    		ZKWatcher myWatcher = new ZKWatcher();
    		Thread.sleep(100000000);
    	}
    }

Test.java类

    package qs.zookeeper.cluster;
    
    import java.util.concurrent.CountDownLatch;
    
    import org.apache.zookeeper.CreateMode;
    import org.apache.zookeeper.WatchedEvent;
    import org.apache.zookeeper.Watcher;
    import org.apache.zookeeper.Watcher.Event.EventType;
    import org.apache.zookeeper.Watcher.Event.KeeperState;
    import org.apache.zookeeper.ZooDefs.Ids;
    import org.apache.zookeeper.ZooKeeper;
    
    public class Test {
    
    
    	/** zookeeper地址 */
    	static final String CONNECT_ADDR = "192.168.1.106:2181,192.168.1.107:2181,192.168.1.108:2181";
    	/** session超时时间 */
    	static final int SESSION_OUTTIME = 2000;//ms 
    	/** 信号量，阻塞程序执行，用于等待zookeeper连接成功，发送成功信号 */
    	static final CountDownLatch connectedSemaphore = new CountDownLatch(1);
    	
    	public static void main(String[] args) throws Exception{
    		
    		ZooKeeper zk = new ZooKeeper(CONNECT_ADDR, SESSION_OUTTIME, new Watcher(){
    			@Override
    			public void process(WatchedEvent event) {
    				//获取事件的状态
    				KeeperState keeperState = event.getState();
    				EventType eventType = event.getType();
    				//如果是建立连接
    				if(KeeperState.SyncConnected == keeperState){
    					if(EventType.None == eventType){
    						//如果建立连接成功，则发送信号量，让后续阻塞程序向下执行
    						connectedSemaphore.countDown();
    						System.out.println("zk 建立连接");
    					}
    				}
    			}
    		});
    
    		//进行阻塞
    		connectedSemaphore.await();
    		
    //		//创建子节点
    //		zk.create("/super/c1", "c1".getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
    		//创建子节点
    //		zk.create("/super/c2", "c2".getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
    		//创建子节点
    		zk.create("/super/c3", "c3".getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
    		//创建子节点
    //		zk.create("/super/c4", "c4".getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
    		
    //		zk.create("/super/c4/c44", "c44".getBytes(), Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
    		
    		//获取节点信息
    //		byte[] data = zk.getData("/testRoot", false, null);
    //		System.out.println(new String(data));
    //		System.out.println(zk.getChildren("/testRoot", false));
    		
    		//修改节点的值
    //		zk.setData("/super/c1", "modify c1".getBytes(), -1);
    //		zk.setData("/super/c2", "modify c2".getBytes(), -1);
    //		byte[] data = zk.getData("/super/c2", false, null);
    //		System.out.println(new String(data));		
    		
    //		//判断节点是否存在
    //		System.out.println(zk.exists("/super/c3", false));
    //		//删除节点
    //		zk.delete("/super/c3", -1);
    		
    		zk.close();
		
		
		
	  }
    }

# zookeeper的ACL(AUTH) #

    package qs.zookeeper.auth;
    
    import java.util.ArrayList;
    import java.util.List;
    import java.util.concurrent.CountDownLatch;
    import java.util.concurrent.atomic.AtomicInteger;
    
    import org.apache.zookeeper.CreateMode;
    import org.apache.zookeeper.WatchedEvent;
    import org.apache.zookeeper.Watcher;
    import org.apache.zookeeper.Watcher.Event.EventType;
    import org.apache.zookeeper.Watcher.Event.KeeperState;
    import org.apache.zookeeper.ZooDefs.Ids;
    import org.apache.zookeeper.ZooKeeper;
    import org.apache.zookeeper.data.ACL;
    import org.apache.zookeeper.data.Stat;
    /**
     * Zookeeper 节点授权
     */
    public class ZookeeperAuth implements Watcher {
    
    	/** 连接地址 */
    	final static String CONNECT_ADDR = "192.168.80.88:2181";
    	/** 测试路径 */
    	final static String PATH = "/testAuth";
    	final static String PATH_DEL = "/testAuth/delNode";
    	/** 认证类型 */
    	final static String authentication_type = "digest";
    	/** 认证正确方法 */
    	final static String correctAuthentication = "123456";
    	/** 认证错误方法 */
    	final static String badAuthentication = "654321";
    	
    	static ZooKeeper zk = null;
    	/** 计时器 */
    	AtomicInteger seq = new AtomicInteger();
    	/** 标识 */
    	private static final String LOG_PREFIX_OF_MAIN = "【Main】";
    	
    	private CountDownLatch connectedSemaphore = new CountDownLatch(1);
    	
    	@Override
    	public void process(WatchedEvent event) {
    		try {
    			Thread.sleep(200);
    		} catch (InterruptedException e) {
    			e.printStackTrace();
    		}
    		if (event==null) {
    			return;
    		}
    		// 连接状态
    		KeeperState keeperState = event.getState();
    		// 事件类型
    		EventType eventType = event.getType();
    		// 受影响的path
    		String path = event.getPath();
    		
    		String logPrefix = "【Watcher-" + this.seq.incrementAndGet() + "】";
    
    		System.out.println(logPrefix + "收到Watcher通知");
    		System.out.println(logPrefix + "连接状态:\t" + keeperState.toString());
    		System.out.println(logPrefix + "事件类型:\t" + eventType.toString());
    		if (KeeperState.SyncConnected == keeperState) {
    			// 成功连接上ZK服务器
    			if (EventType.None == eventType) {
    				System.out.println(logPrefix + "成功连接上ZK服务器");
    				connectedSemaphore.countDown();
    			} 
    		} else if (KeeperState.Disconnected == keeperState) {
    			System.out.println(logPrefix + "与ZK服务器断开连接");
    		} else if (KeeperState.AuthFailed == keeperState) {
    			System.out.println(logPrefix + "权限检查失败");
    		} else if (KeeperState.Expired == keeperState) {
    			System.out.println(logPrefix + "会话失效");
    		}
    		System.out.println("--------------------------------------------");
    	}
    	/**
    	 * 创建ZK连接
    	 * 
    	 * @param connectString
    	 *ZK服务器地址列表
    	 * @param sessionTimeout
    	 *Session超时时间
    	 */
    	public void createConnection(String connectString, int sessionTimeout) {
    		this.releaseConnection();
    		try {
    			zk = new ZooKeeper(connectString, sessionTimeout, this);
    			//添加节点授权
    			zk.addAuthInfo(authentication_type,correctAuthentication.getBytes());
    			System.out.println(LOG_PREFIX_OF_MAIN + "开始连接ZK服务器");
    			//倒数等待
    			connectedSemaphore.await();
    		} catch (Exception e) {
    			e.printStackTrace();
    		}
    	}
    	
    	/**
    	 * 关闭ZK连接
    	 */
    	public void releaseConnection() {
    		if (this.zk!=null) {
    			try {
    				this.zk.close();
    			} catch (InterruptedException e) {
    			}
    		}
    	}
    	
    	/**
    	 * 
    	 * <B>方法名称：</B>测试函数<BR>
    	 * <B>概要说明：</B>测试认证<BR>
    	 * @param args
    	 * @throws Exception
    	 */
    	public static void main(String[] args) throws Exception {
    		
    		ZookeeperAuth testAuth = new ZookeeperAuth();
    		testAuth.createConnection(CONNECT_ADDR,2000);
    		List<ACL> acls = new ArrayList<ACL>(1);
    		for (ACL ids_acl : Ids.CREATOR_ALL_ACL) {
    			acls.add(ids_acl);
    		}
    
    		try {
    			zk.create(PATH, "init content".getBytes(), acls, CreateMode.PERSISTENT);
    			System.out.println("使用授权key：" + correctAuthentication + "创建节点："+ PATH + ", 初始内容是: init content");
    		} catch (Exception e) {
    			e.printStackTrace();
    		}
    		try {
    			zk.create(PATH_DEL, "will be deleted! ".getBytes(), acls, CreateMode.PERSISTENT);
    			System.out.println("使用授权key：" + correctAuthentication + "创建节点："+ PATH_DEL + ", 初始内容是: init content");
    		} catch (Exception e) {
    			e.printStackTrace();
    		}
    
    		// 获取数据
    		getDataByNoAuthentication();
    		getDataByBadAuthentication();
    		getDataByCorrectAuthentication();
    
    		// 更新数据
    		updateDataByNoAuthentication();
    		updateDataByBadAuthentication();
    		updateDataByCorrectAuthentication();
    
    		// 删除数据
    		deleteNodeByBadAuthentication();
    		deleteNodeByNoAuthentication();
    		deleteNodeByCorrectAuthentication();
    		//
    		Thread.sleep(1000);
    		
    		deleteParent();
    		//释放连接
    		testAuth.releaseConnection();
    	}
    	/** 获取数据：采用错误的密码 */
    	static void getDataByBadAuthentication() {
    		String prefix = "[使用错误的授权信息]";
    		try {
    			ZooKeeper badzk = new ZooKeeper(CONNECT_ADDR, 2000, null);
    			//授权
    			badzk.addAuthInfo(authentication_type,badAuthentication.getBytes());
    			Thread.sleep(2000);
    			System.out.println(prefix + "获取数据：" + PATH);
    			System.out.println(prefix + "成功获取数据：" + badzk.getData(PATH, false, null));
    		} catch (Exception e) {
    			System.err.println(prefix + "获取数据失败，原因：" + e.getMessage());
    		}
    	}
    
    	/** 获取数据：不采用密码 */
    	static void getDataByNoAuthentication() {
    		String prefix = "[不使用任何授权信息]";
    		try {
    			System.out.println(prefix + "获取数据：" + PATH);
    			ZooKeeper nozk = new ZooKeeper(CONNECT_ADDR, 2000, null);
    			Thread.sleep(2000);
    			System.out.println(prefix + "成功获取数据：" + nozk.getData(PATH, false, null));
    		} catch (Exception e) {
    			System.err.println(prefix + "获取数据失败，原因：" + e.getMessage());
    		}
    	}
    
    	/** 采用正确的密码 */
    	static void getDataByCorrectAuthentication() {
    		String prefix = "[使用正确的授权信息]";
    		try {
    			System.out.println(prefix + "获取数据：" + PATH);
    			
    			System.out.println(prefix + "成功获取数据：" + zk.getData(PATH, false, null));
    		} catch (Exception e) {
    			System.out.println(prefix + "获取数据失败，原因：" + e.getMessage());
    		}
    	}
    
    	/**
    	 * 更新数据：不采用密码
    	 */
    	static void updateDataByNoAuthentication() {
    
    		String prefix = "[不使用任何授权信息]";
    
    		System.out.println(prefix + "更新数据： " + PATH);
    		try {
    			ZooKeeper nozk = new ZooKeeper(CONNECT_ADDR, 2000, null);
    			Thread.sleep(2000);
    			Stat stat = nozk.exists(PATH, false);
    			if (stat!=null) {
    				nozk.setData(PATH, prefix.getBytes(), -1);
    				System.out.println(prefix + "更新成功");
    			}
    		} catch (Exception e) {
    			System.err.println(prefix + "更新失败，原因是：" + e.getMessage());
    		}
    	}
    
    	/**
    	 * 更新数据：采用错误的密码
    	 */
    	static void updateDataByBadAuthentication() {
    
    		String prefix = "[使用错误的授权信息]";
    
    		System.out.println(prefix + "更新数据：" + PATH);
    		try {
    			ZooKeeper badzk = new ZooKeeper(CONNECT_ADDR, 2000, null);
    			//授权
    			badzk.addAuthInfo(authentication_type,badAuthentication.getBytes());
    			Thread.sleep(2000);
    			Stat stat = badzk.exists(PATH, false);
    			if (stat!=null) {
    				badzk.setData(PATH, prefix.getBytes(), -1);
    				System.out.println(prefix + "更新成功");
    			}
    		} catch (Exception e) {
    			System.err.println(prefix + "更新失败，原因是：" + e.getMessage());
    		}
    	}
    
    	/**
    	 * 更新数据：采用正确的密码
    	 */
    	static void updateDataByCorrectAuthentication() {
    
    		String prefix = "[使用正确的授权信息]";
    
    		System.out.println(prefix + "更新数据：" + PATH);
    		try {
    			Stat stat = zk.exists(PATH, false);
    			if (stat!=null) {
    				zk.setData(PATH, prefix.getBytes(), -1);
    				System.out.println(prefix + "更新成功");
    			}
    		} catch (Exception e) {
    			System.err.println(prefix + "更新失败，原因是：" + e.getMessage());
    		}
    	}
    
    	/**
    	 * 不使用密码 删除节点
    	 */
    	static void deleteNodeByNoAuthentication() throws Exception {
    
    		String prefix = "[不使用任何授权信息]";
    
    		try {
    			System.out.println(prefix + "删除节点：" + PATH_DEL);
    			ZooKeeper nozk = new ZooKeeper(CONNECT_ADDR, 2000, null);
    			Thread.sleep(2000);
    			Stat stat = nozk.exists(PATH_DEL, false);
    			if (stat!=null) {
    				nozk.delete(PATH_DEL,-1);
    				System.out.println(prefix + "删除成功");
    			}
    		} catch (Exception e) {
    			System.err.println(prefix + "删除失败，原因是：" + e.getMessage());
    		}
    	}
    
    	/**
    	 * 采用错误的密码删除节点
    	 */
    	static void deleteNodeByBadAuthentication() throws Exception {
    
    		String prefix = "[使用错误的授权信息]";
    
    		try {
    			System.out.println(prefix + "删除节点：" + PATH_DEL);
    			ZooKeeper badzk = new ZooKeeper(CONNECT_ADDR, 2000, null);
    			//授权
    			badzk.addAuthInfo(authentication_type,badAuthentication.getBytes());
    			Thread.sleep(2000);
    			Stat stat = badzk.exists(PATH_DEL, false);
    			if (stat!=null) {
    				badzk.delete(PATH_DEL, -1);
    				System.out.println(prefix + "删除成功");
    			}
    		} catch (Exception e) {
    			System.err.println(prefix + "删除失败，原因是：" + e.getMessage());
    		}
    	}
    
    	/**
    	 * 使用正确的密码删除节点
    	 */
    	static void deleteNodeByCorrectAuthentication() throws Exception {
    
    		String prefix = "[使用正确的授权信息]";
    
    		try {
    			System.out.println(prefix + "删除节点：" + PATH_DEL);
    			Stat stat = zk.exists(PATH_DEL, false);
    			if (stat!=null) {
    				zk.delete(PATH_DEL, -1);
    				System.out.println(prefix + "删除成功");
    			}
    		} catch (Exception e) {
    			System.out.println(prefix + "删除失败，原因是：" + e.getMessage());
    		}
    	}
    
    	/**
    	 * 使用正确的密码删除节点
    	 */
    	static void deleteParent() throws Exception {
    		try {
    			Stat stat = zk.exists(PATH_DEL, false);
    			if (stat == null) {
    				zk.delete(PATH, -1);
    			}
    		} catch (Exception e) {
    			e.printStackTrace();
    		}
    	}
    
    }





