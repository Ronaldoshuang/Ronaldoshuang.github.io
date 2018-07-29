---
layout: post
title: ZkClient操作Zookeeper
categories: Zookeeper
description: Zookeeper
keywords: Zookeeper
---

# 增删改查节点 #

    package qs.zkclient.base;
    
    import java.util.List;
    import java.util.concurrent.TimeUnit;
    
    import org.I0Itec.zkclient.ZkClient;
    import org.I0Itec.zkclient.ZkConnection;
    
    public class ZkClientBase {
    
    	/** zookeeper地址 */
    	static final String CONNECT_ADDR = "192.168.1.171:2181,192.168.1.172:2181,192.168.1.173:2181";
    	/** session超时时间 */
    	static final int SESSION_OUTTIME = 5000;//ms 
    	
    	
    	public static void main(String[] args) throws Exception {
    		ZkClient zkc = new ZkClient(new ZkConnection(CONNECT_ADDR), 5000);
    		//1. create and delete方法 
    		zkc.createEphemeral("/temp");
    		zkc.createPersistent("/super/c1", true);
    		Thread.sleep(10000);
    		zkc.delete("/temp");
    		zkc.deleteRecursive("/super");
    		
    		//2. 设置path和data 并且读取子节点和每个节点的内容
    //		zkc.createPersistent("/super", "1234");
    //		zkc.createPersistent("/super/c1", "c1内容");
    //		zkc.createPersistent("/super/c2", "c2内容");
    //		List<String> list = zkc.getChildren("/super");
    //		for(String p : list){
    //			System.out.println(p);
    //			String rp = "/super/" + p;
    //			String data = zkc.readData(rp);
    //			System.out.println("节点为：" + rp + "，内容为: " + data);
    //		}
    		
    		//3. 更新和判断节点是否存在
    //		zkc.writeData("/super/c1", "新内容");
    //		System.out.println(zkc.readData("/super/c1"));
    //		System.out.println(zkc.exists("/super/c1"));
    		
    		//4.递归删除/super内容
    //		zkc.deleteRecursive("/super");		
    	}
    }

# zkclient的watcher #

    package qs.zkclient.watcher;
    
    import java.util.List;
    
    import org.I0Itec.zkclient.IZkChildListener;
    import org.I0Itec.zkclient.IZkDataListener;
    import org.I0Itec.zkclient.IZkStateListener;
    import org.I0Itec.zkclient.ZkClient;
    import org.I0Itec.zkclient.ZkConnection;
    import org.apache.zookeeper.Watcher.Event.KeeperState;
    
    public class ZkClientWatcher2 {
    
    	/** zookeeper地址 */
    	static final String CONNECT_ADDR = "192.168.1.171:2181,192.168.1.172:2181,192.168.1.173:2181";
    	/** session超时时间 */
    	static final int SESSION_OUTTIME = 5000;//ms 
    	
    	
    	public static void main(String[] args) throws Exception {
    		ZkClient zkc = new ZkClient(new ZkConnection(CONNECT_ADDR), 5000);
    		
    		zkc.createPersistent("/super", "1234");
    		
    		//对父节点添加监听子节点变化。
    		//------------------子节点变化----------------------
    		zkc.subscribeChildChanges("/super", new IZkChildListener() {
    			
    			@Override
    			public void handleChildChange(String arg0, List<String> arg1) throws Exception {
    				// TODO Auto-generated method stub
    				
    			}
    		});
    //		//----------------节点状态变化--------------------------
    //		zkc.subscribeStateChanges(new IZkStateListener() {
    //			
    //			@Override
    //			public void handleStateChanged(KeeperState arg0) throws Exception {
    //				// TODO Auto-generated method stub
    //				
    //			}
    //			
    //			@Override
    //			public void handleNewSession() throws Exception {
    //				// TODO Auto-generated method stub
    //				
    //			}
    //		});
    		//--------------节点数据变化----------------------------
    		zkc.subscribeDataChanges("/super", new IZkDataListener() {
    			@Override
    			public void handleDataDeleted(String path) throws Exception {
    				System.out.println("删除的节点为:" + path);
    			}
    			
    			@Override
    			public void handleDataChange(String path, Object data) throws Exception {
    				System.out.println("变更的节点为:" + path + ", 变更内容为:" + data);
    			}
    			
    		});
    		Thread.sleep(3000);
    		zkc.writeData("/super", "456", -1);
    		Thread.sleep(1000);
    
    		zkc.delete("/super");
    		Thread.sleep(Integer.MAX_VALUE);
    		
    		
    	}
    }

