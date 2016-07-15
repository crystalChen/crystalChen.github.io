---
layout: post
categories: [JUC]
tags: [JUC]
code: true
title: 通过CountDownLatch浅析AQS（待续）
---

上次想做爬虫，遇到问题在stackoverflow搜索的时候发现了CountDownLatch，之前没有接触过，于是去了解了一下，知道了这是一个倒计数锁。  
      
API描述：

&emsp;&emsp;一个同步辅助类，在完成一组正在其他线程中执行的操作之前，它允许一个或多个线程一直等待。    
&emsp;&emsp;用给定的计数初始化CountDownLatch。由于调用了 countDown()方法，所以在当前计数到达零之前，await()方法会一直受阻塞。之后，会释放所有等待的线程，await的所有后续调用都将立即返回。这种现象只出现一次——计数无法被重置。如果需要重置计数，请考虑使用CyclicBarrier。
&emsp;&emsp;CountDownLatch 是一个通用同步工具，它有很多用途。将计数 1 初始化的 CountDownLatch 用作一个简单的开/关锁存器，或入口：在通过调用countDown() 的线程打开入口前，所有调用 await 的线程都一直在入口处等待。用 N 初始化的 CountDownLatch 可以使一个线程在 N 个线程完成某项操作之前一直等待，或者使其在某项操作完成 N 次之前一直等待。  
<b>总结一下：倒计数锁，await方法会阻塞线程直到count减到0为止。</b>  
网上说count为N则必须要有N个线程的说法是错误的。API明确说了可以是N次操作，N和线程个数没有关系，亲测。  
自己猜了猜如何实现的，以为是synchronized + CAS + queue，可是它是JUC包下的，主要通过内部子类继承AQS来实现逻辑，AQS才是关键所在，看了好久的AQS源码看的都不是很清晰，但是毕竟看了那么久，写个笔记先，以后可以接着看。   
 
要看懂AQS，要先了解以下知识：  
1.自旋锁  
2.CLH lock queue  
3.LockSupport类的park()和unpark()  

#### 1.自旋锁(spinlock)  
自旋锁是指当一个线程尝试获取某个锁时，如果该锁已被其他线程占用，就一直循环检测锁是否被释放，而不是进入线程挂起或睡眠状态。
自旋锁适用于锁保护的临界区很小的情况，临界区很小的话，锁占用的时间就很短。比方程序中一直访问某个变量的状态，如：while(state == true){...}

#### 2.CLH Lock Queue  
CLH锁是一种基于链表的可扩展、高性能、公平的自旋锁，申请线程只在本地变量上自旋，它不断轮询前驱的状态，如果发现前驱释放了锁就结束自旋。CLH锁只有prev指针，没有next指针。[CLH论文地址](http://people.csail.mit.edu/shanir/publications/CLH.pdf)。示意图：

           +------+  prev +-----+       +-----+
      head |      | <---- |     | <---- |     |  tail
           +------+       +-----+       +-----+

而AQS使用的是CLH的变种，不再是自旋，是阻塞的，同时为了唤醒后续的结点，提供了next指针。
#### 3.LockSupport类的park()和unpark()
LockSupport实际上是调用了Unsafe类里的函数，归结到Unsafe里，只有两个函数：  

	public native void unpark(Thread jthread);        
	public native void park(boolean isAbsolute, long time);

unpark函数可以先于park调用。比如线程B调用unpark函数（一次多次都是作一个标记,可以理解设置boolean值），当线程A调用park时，看到许可的标记，那么它会马上再继续运行。  
不同于notify只会唤醒一个线程，如果错误地有两个线程在同一个对象上wait等待，那么又悲剧了。而notifyAll是唤醒所有。  
[Doug Lea的AQS论文地址](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)