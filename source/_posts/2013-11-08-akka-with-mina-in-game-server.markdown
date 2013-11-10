---
layout: post
title: "游戏项目中的akka"
date: 2013-11-08 22:18
comments: true
categories: 
---


为什么选择Akka?

从模型的角度来说：

 * akka是基于actor模型，游戏中的每个玩家也是actor，所以他们是天然联系在一起的。

从技术的角度上来看:

 * akka实例占用内存极少，在一台机器上可创建百万数量级的actor实例。
 * akka能够充分利用多核cpu的能量，提高机器的效率
 * akka的消息模型是单向有序的，可以解决客户端与服务器发包乱序的问题。
 
 

对于java游戏服务器中，都会选择个一个io框架，或者是mina，或者是netty。但不管如何，其核心模式
（主线程+工作线程），主线程负责监听来自客户端请求，创建socket；工作线程负责业务处理。akka的引入，将会替代工作线程，进行业务处理。


在实际情况中，决定使用mina+akka来实现这个思路，为此需要解决如下问题：

 * 如何替代工作线程
 * 如何创建akka，销毁akka实例。
 * 

解决的入口是Mina的IoHandlerAdapter类，通过此类，可以创建实例，销毁，调用akka实例。

步骤如下：

 * 创建AkkaIoActor接口，替代IoHandlerAdapter.messageReceieved函数
 * 创建AkkaIoActorImpl，将逻辑处理转移给akka。
 * 创建AkkaIoActorFactory，负责创建、销毁AkkaIoActor实例。
 * 在IoHandlerAdapter类，集成AkkaIoActor。
 * 
相关的假代码如下：

```java

public class AkkaIoHandler extends IoHandlerAdapter {

   
    //使用Spring注入各种资源

	public AkkaIoHandler(){
	}
	
	@Override
	public void sessionOpened(IoSession ioSession) throws Exception {
        //用AkkaIoAcotrFactory创建Akka实例
        //讲akka实例与ioSession关联

	}

	@Override
	public void messageReceived(IoSession ioSession, Object message) throws Exception {
		//找到对应akka实例
        //调用akka进行逻辑处理
	}
	

	@Override
	public void exceptionCaught(IoSession session, Throwable cause) throws Exception {
	    //销毁akka实例，并与session解除关联
	}

	@Override
	public void sessionClosed(IoSession ioSession) throws Exception {
        //销毁akka实例，并与session解除关联
	}
```

整体思路大概就是这样。相关的性能测试数据后续再补充。