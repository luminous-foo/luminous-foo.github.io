---
title: java多线程
date: 2020-12-28 22:20:52
tags:
- java
- 随笔
categories:
 - 复习
---


## 一、线程创建4种方式

### 1、继承Thread

### 2、实现Runnable接口

### 3、使用FutureTask接受实现Callable接口类的构造，然后传入线程类

### 4、使用线程池

<!--more-->

![FutureTask](java多线程/20160713174739239)



## 二、常用线程池

	newFixedThreadPool
	newSingleThreadExecutor
	newCachedThreadPool
	创建一个大小无限的线程池。此线程池支持定时以及周期性执行任务的需求
	newScheduledThreadPool
​	



## 三、线程池7大参数

​		corePoolSize:线程池中的常驻核心线程数
​		maximunPoolSize:线程池能够容纳同时执行的最大线程数
​		keepAliveTime:多余的空闲线程的存活时间(多余线程池线程数据超过corePoolSize时，空闲时间达到keepAliveTime)
​		Unit:keepAliveTime的单位
​		workQueue:任务队列，被提交但尚未被执行的任务
​		threadFactory:表示生成线程池中工作线程的线程工厂，用来创建线程一般用默认的即可
​		handler:拒绝策略，当队列满了并且工作线程大于等于线程池的最大线程数maximunPoolSize时如何来如何拒绝请求执行的runnable的策略
​		

		线程池工作原理
			1.创建了线程池后，等待提交过来的任务请求。
			2.当调用execute()方法添加一个请求任务，线程池会做如下判断
				2.1如果正在运行的线程数量小于corePoolSize，那么马上创建线程运行这个任务
				2.2如果正在运行的线程数量大于等于corePoolSize，那么将这个任务放入队列
				2.3如果这时候队列满了且正在运行的线程数量还小于maximumPoolSize，那么还是要创建非核心线程立刻运行这个任务
				2.4如果队列满了且正在运行的线程数量大于或等于maximumPoolSize，那么线程池会启动饱和拒绝策略来执行
			3.当一个线程完成任务时，它会从队列中取下一个任务来执行
			4.当一个线程无事可做超过一定的时间（keepAliveTime）时，线程会判断：
				如果当前线程数大于corePoolSize，那么这个线程被停掉
				所以线程池的所有任务完成后它最终会收缩到corePoolSize的大小
		线程池的拒绝策略（等待队列已经满了，再也塞不下新任务了，线程池的max线程也达到了，无法继续为新任务服务，这时候用拒绝策略合理的处理这个问题）
			1.AbortPolicy:直接抛出RejectedExecutionException
			2.CallerRunsPolicy:"调用者运行"一种调节机制，该策略既不会抛弃任务，也不会抛异常。而实将某些任务回退到调用者，从而降低新的任务流量
			3.DiscardOldersPolicy:抛弃队列中等待最久的任务，然后把当前任务加入队列中尝试再次提交当前任务
			4.DiscardPolicy:直接丢弃任务，不予处理也不抛弃异常。
		线程池参数设置：CPU密集型:核心数+1  IO密集型:2*核心数，核心数/（1-阻尼系数（0.8-0.9））
##  四、使用Lock代替Synchronized

​	Synchronized	Lock
​	wait			(Condition)c.await
​	notify			(Condition)c.signal
​	使用Lock可以让多线程之间按顺序执行
​	区别：

​		  1原始构成：Synchronized是关键字属于JVM层面（monitorenter monitorexit）Lock是具体类是API层面
​	      2使用方法：Synchronized不需要手动释放锁，执行完系统会自动让线程释放对锁的占用 Lock需要手动释放锁否则可能死锁
​		  3等待是否可中断：Synchronized不可中断，除非抛出异常，Lock可中断
​		  4加锁是否公平
​		  5锁绑定多个条件Condition(精准唤醒)