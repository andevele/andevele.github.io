---
layout: post
title: 如何停止线程
categories: Java Android
description: 描述了java中线程如何停止并给出Android中线程停止的方法
keywords: 线程, 停止
---

&emsp;&emsp;如何正确地停止线程在java多线程开发中是一个重要的技术点，处理不当的话会带来各种异常结果。


#### 线程定义
先看线程定义，线程是现代操作系统调度的最小单元，也叫轻量级进程。一个进程至少有一个线程，也可以创建多个线程，这些线程可以共享进程所拥有的资源
java中停止线程总体来说有四种方法
* run方法自行运行结束退出
* 标志位 
* 线程的interrupt方法
* 线程stop方法  
第四种stop方法在jdk1.2中已经被标明过期、作废的方法，因为stop强制停止线程，会带来数据不一致性的问题，所以一般情况下不使用stop方法。第一种“run自行运行结束退出”，这不是程序员主动停止的方式，而是线程自有的特性，因此，可以回答停止线程只有2种方法：标志位和interrupt，这两种情况都是需要控制程序流程的，是一种主动行为。当然，如果非要纠结，这2种方法最后都会在run运行结束后自行退出线程，这也就回归到了第一种方法，但这样说也没有什么意义，正常开发过程中，还是要用到主动停止的行为。


##### run方法自行运行结束退出
``` java 
	@Override
	public void run() {
		doSomething();
	}
```
run方法执行完任务后自动退出，这没什么可说的，就像普通方法一样，任务执行完成就退出了。但大多数情况下，需要主动停止线程，这就要用其他技术实现。


##### 线程interrupt方法
要中断一个java线程，可用Thread对象的interrupt()成员方法，但该方法并不会立即执行中断操作，而是给线程设置一个值为true的中断标志(中断标志是一个布尔型的变量)，然后再根据线程状态决定如何退出。  
那么线程状态有哪些呢？  
线程生命周期分为初始、可运行或就绪、运行、阻塞、终止五个状态
如果线程处于运行状态，那么interrupt方法设置中断标志为true，被设置为true的线程继续运行并不会停止，是否退出由自行决定
如果线程处于阻塞状态，比如在调用wait、sleep、join方法时进入阻塞状态，那么调用interrupt方法后，java虚拟机先将该线程的中断标志清除，重新设为false，然后抛出InterruptedException异常，此时是否退出线程根据具体功能决定  

可以看出，interrupt并不能停止线程，只是给线程设置了中断标志，是否停止线程还需要配合程序流程或功能需求来决定。

###### 处于运行状态调用interrupt方法
先看一个案例，线程在运行时中断，看看如何停止线程
``` java
class ThreadInterrupt extends Thread {
	@Override
	public void run() {
		for (int i = 0; i < 500000; i++) {
			if(this.isInterrupted()) {
				System.out.println("线程已经停止了，不需要再执行循环了");
				break;
			}
			System.out.println("i =" + (i + 1));
		}
		System.out.println("for循环下面的语句");
		System.out.println("===线程执行结束==");
	}
}

```

``` java
public class ThreadInterruptTest {
	public static void main(String[] args) {
		// TODO Auto-generated method stub
		ThreadInterrupt thread = new ThreadInterrupt();
		thread.start();
		try {
			Thread.sleep(2000);
		} catch (InterruptedException e) {
			System.out.println("===抛出异常,在catch语句中===");
			e.printStackTrace();
		}
		thread.interrupt();
		System.out.println("===main end===");
	}
}
```
运行结果：
===main end===
线程已经停止了，不需要再执行循环了
for循环下面的语句
===线程执行结束==

子线程ThreadInterrupt对象启动后开始执行循环语句，此时主线程开始睡眠，子线程调用isInterrupted判断是否中断，线程还没有中断，返回false
当主线程睡眠2秒后，子线程执行interrupt方法，中断标志位被设置为true，isInterrupted返回true，条件语句if(this.isInterrupted())成立，break跳出for循环，开始执行for下面的
两条打印语句，执行完后run方法结束，线程正常退出。

由此可知，线程处于运行状态时，在中断标志被设为true后，线程是否退出，还需要结合其他方法来退出线程，用break跳出循环就是一种方式，跳出循环后没有其他多余的循环语句了，只是简单的两句打印，执行后run方法也就结束了，线程自行退出。



###### 处于阻塞状态调用interrupt方法



##### 标志位
``` java 
	@Override
	public void run() {
		while(isRun) {
			doSomething();
		}		
	}
```
假如线程中有一个循环语句，当isRun为true时运行，为false时退出循环，如果循环后面也没有其他循环语句，退出循环后自行运行结束，线程停止。


##### 线程stop方法




