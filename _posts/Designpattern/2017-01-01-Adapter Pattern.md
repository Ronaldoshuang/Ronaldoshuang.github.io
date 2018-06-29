---
layout: post
title: 适配器模式
categories: Designpattern
description: 适配器模式
keywords: java, 适配器模式
---

**Adaptee.java（被适配的类，相当于例子中的，PS/2键盘）**

    public class Adaptee {
    	
    	public void request(){
    		System.out.println("可以完成客户请求的需要的功能！");
    	}
    }


**Adapter.java（适配器）**

    public class Adapter extends Adaptee implements Target {
    	
    	
    	@Override
    	public void handleReq() {
    		super.request();
    	}
    	
    }


**Adapter2.java（适配器二）**

    public class Adapter2  implements Target {
    	
    	private Adaptee adaptee;
    	
    	@Override
    	public void handleReq() {
    		adaptee.request();
    	}
    
    	public Adapter2(Adaptee adaptee) {
    		super();
    		this.adaptee = adaptee;
    	}
    }


**Target.java（接口）**

    public interface Target {
    	void handleReq();
    }


**Client.java（客户端类，相当于例子中的笔记本，只有USB接口）**

    public class Client {
    	
    	public void test1(Target t){
    		t.handleReq();
    	}
    	
    	public static void main(String[] args) {
    		Client  c = new Client();
    		
    		Adaptee a = new Adaptee();
    		
    //		Target t = new Adapter();
    
    		Target t = new Adapter2(a);
    		
    		c.test1(t);
    		
    	}
    	
    }

