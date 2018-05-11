---
layout: post
title: 如何停止线程
categories: Java
description: 描述了java中线程如何停止并给出Android中线程停止的方法
keywords: 线程, 停止
---

如何正确地停止线程在java多线程开发中是一个重要的技术点，处理不当的话会带来各种异常结果。


### 线程定义
先看线程定义，线程是现代操作系统调度的最小单元，也叫轻量级进程。一个进程至少有一个线程，也可以创建多个线程，这些线程可以共享进程所拥有的资源，java中停止线程总体来说有四种方法：
* run方法自己运行结束退出
* 线程的interrupt方法
* 标志位 
* 线程stop方法  
  
第四种stop方法在jdk1.2中已经被标明过期、作废的方法，stop会强制停止线程，可能带来数据不一致性的问题，所以一般情况下不使用stop方法。  

第一种“run自行运行结束退出”，这不是程序员主动停止的方式，而是线程自有的特性。因此可以回答停止线程只有2种方法：标志位和interrupt，这两种情况都是需要控制程序流程的，是一种主动行为。  

当然，如果非要纠结，这2种方法最后都会在run运行结束后自行退出线程，这也就回归到了第一种方法，但这样说也没有什么意义，正常开发过程中，如要控制线程停止，还是要用到主动停止的行为。


### run方法自行运行结束退出
``` java 
@Override
public void run() {
	doSomething();
}
```
run方法执行完任务后自动退出，这没什么可说的，就像普通方法一样，任务执行完成就退出了。


### 线程interrupt方法
要中断一个java线程，可用Thread对象的interrupt()成员方法，但该方法并不会立即执行中断操作，而是给线程设置一个值为true的中断标志(中断标志是一个布尔型的变量)，然后再根据线程状态决定如何退出。  
那么线程状态有哪些呢？  

线程生命周期分为初始、可运行或就绪、运行、阻塞、终止五个状态。  

如果线程处于运行状态，那么interrupt方法设置中断标志为true，被设置为true的线程继续运行并不会停止，是否退出由程序流程决定 

如果线程处于阻塞状态，比如在调用wait、sleep、join方法时进入阻塞状态，那么调用interrupt方法后，java虚拟机先将该线程的中断标志清除，重新设为false，然后抛出InterruptedException异常，此时是否退出线程根据具体功能决定。  

可以看出，interrupt并不能停止线程，只是给线程设置了中断标志，是否停止线程还需要配合功能需求来决定。

#### 处于运行状态调用interrupt方法
	
看一个案例，线程在运行时中断，看如何停止线程
``` java
class ThreadInterrupt extends Thread {
  @Override
  public void run() {
	for (int i = 0; i < 500000; i++) {
	  if(this.isInterrupted()) {
	    System.out.println("线程已经停止了，不需要再执行循环了");
        //doSomething();
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
i =382081  
i =382082  
i =382083  
i =382084  
===main end===  
线程已经停止了，不需要再执行循环了  
我在for循环下面，将被执行  
===线程执行结束==  

子线程ThreadInterrupt对象启动后执行循环语句，主线程开始睡眠，子线程调用isInterrupted判断是否中断，线程还没有中断，返回false。当主线程睡眠2秒后，子线程执行interrupt方法，中断标志位被设置为true，isInterrupted返回true，if(this.isInterrupted())成立，break跳出循环，开始执行for下面的两条打印语句，执行完后结束退出。

线程处于运行状态时，在中断标志被设为true后，线程是否退出需要结合其他方法，用break跳出循环就是一种方式。

这个例子在退出循环后会执行后面的打印语句，如果后面的语句还是循环，那线程又退不出去了，为了解决此问题，采用抛出异常方法，同时尽量不要把任务放在异常捕获语句之后，ThreadInterrupt修改后如下，main方法不变  
``` java
class ThreadInterrupt extends Thread {
  @Override
  public void run() {
    super.run();
    try {
      for (int i = 0; i < 500000; i++) {
        if(this.isInterrupted()) {
          System.out.println("线程已经被设置为中断,不需要再执行循环了");
          //break;
          //doSomething();
          throw new InterruptedException();
        }
        System.out.println("i =" + (i + 1));
      }
      System.out.println("我在for循环下面，将被执行");
    }catch(InterruptedException e) {
       System.out.println("我在catch语句中");
       e.printStackTrace();
       System.out.println("===线程执行结束==");
    }
  }
}
```
运行结果：  
i =320129  
i =320130  
i =320131  
i =320132  
===main end===  
线程已经被设置为中断,不需要再执行循环了  
我在catch语句中  
java.lang.InterruptedException  
===线程执行结束==  
	at ThreadInterrupt.run(ThreadInterruptTest.java:10)  

当判断isInterrupted返回true时，抛出异常，直接跳到catch语句中，for之后的语句都没有打印，把该做的事情放在catch之前，一旦进入catch后执行结束退出。


#### 处于阻塞状态调用interrupt方法

再看看阻塞状态下的线程如何停止，阻塞状态有多种方式：
* 等待阻塞：运行的线程执行wait()方法，JVM会把该线程放入等待池中。
* 同步阻塞：运行的线程在获取对象的同步锁时，若该同步锁被别的线程占用，则JVM会把该线程放入锁池中。
* 其他阻塞：运行的线程执行sleep()或join()方法，或者发出了I/O请求时，JVM会把该线程置为阻塞状态。当sleep()状态超时、join()等待线程终止或者超时、或者I/O处理完毕时，线程重新转入就绪状态。

看一个sleep的例子,main方法不变
``` java 
class ThreadInterrupt extends Thread {
	@Override
	public void run() {
        System.out.println("睡眠之前,线程是否处于中断状态? " + this.isInterrupted());
        try {
            Thread.sleep(200000);
        } catch (InterruptedException e) {
            // TODO Auto-generated catch block
            System.out.println("在沉睡中被中断，进入catch语句,线程是否处于中断状态?" + this.isInterrupted());
            e.printStackTrace();
        }
        System.out.println("===线程执行结束==");
    }
}
```
运行结果：  
睡眠之前,线程是否处于中断状态? false  
===main end===  
在沉睡中被中断，进入catch语句,线程是否处于中断状态?false  
java.lang.InterruptedException: sleep interrupted  
===线程执行结束==  
	at java.lang.Thread.sleep(Native Method)  
		at ThreadInterrupt.run(ThreadInterruptTest.java:47)  

子线程一开始还没有被中断，所以isInterrupted返回false，然后开始睡眠，睡眠时间总长为200000ms，睡到2000ms时被中断，子线程抛出异常并进入catch语句，最终运行结束退出。为何在catch语句中isInterrupted返回还是false呢？按道理，线程被中断后，中断标志位应该为true才对。其原因是当线程在睡眠状态下被中断时，中断标志位被置为true，在抛出异常之前JVM先把标志位再置为false，然后再抛出异常，所以在catch中调用isInterrupted为false


再看一个循环语句的睡眠例子,main方法还是不变
``` java
class ThreadInterrupt extends Thread {
	@Override
	public void run() {
		for (int i = 0; i < 10; i++) {
			if(this.isInterrupted()) {
				System.out.println("线程已经停止了，不需要再执行循环了");
				break;
			}
			
			System.out.println("i =" + (i + 1));
			
			try {
				Thread.sleep(800);
			} catch (InterruptedException e) {
				// TODO Auto-generated catch block
				System.out.println("我在睡眠时被中断了");
				//Thread.currentThread().interrupt();//语句1
			}
			
		}
		System.out.println("===线程执行结束==");
	}
}
```
如果把语句1处屏蔽掉，看看打印结果  
i =1  
i =2  
i =3  
===main end===  
我在睡眠时被中断了  
i =4  
i =5  
i =6  
i =7  
i =8  
i =9  
i =10  
===线程执行结束==  


子线程没隔8ms打印i的值，当子线程开始执行interrupt后，抛出异常，按道理循环也中断了，但循环并没有结束，而是继续打印i的值，为何？根据上一个例子结论可知，线程睡眠时被中断，中断标志位会被清除，所以isInterrupted返回为false，if条件不成立，循环继续进行。那如何修改成睡眠被中断时停止循环并退出？只需要把语句1处的代码恢复即可：
``` java 
try {
		Thread.sleep(800);
	} catch (InterruptedException e) {
		// TODO Auto-generated catch block
		System.out.println("我在睡眠时被中断了");
		Thread.currentThread().interrupt();//语句1恢复
	}
```
打印结果为：  
i =1  
i =2  
i =3  
===main end===  
我在睡眠时被中断了  
线程已经停止了，不需要再执行循环了  
===线程执行结束==  

if语句判断成立，break跳出循环，最终run方法执行完退出线程。



### 标志位
假如线程中有一个循环语句，当条件为true时运行，为false时退出循环，如果循环后面也没有其他循环语句，退出循环后自行运行结束，线程停止，可以用如下伪代码表示 
``` java 
	@Override
	public void run() {
		while(isRun) {
			doSomething();
		}		
	}
```
除了中断标志位，下面的例子用一个boolean变量来控制是否中断任务并终止线程。

```java
class NormalThread extends Thread {

	private boolean isRun = true;

	@Override
	public void run() {
		// TODO Auto-generated method stub
		int i = 0;
		while(isRun && !this.isInterrupted()) {
			i++;
		}
		System.out.println("=======循环的次数=== " + i);
	}
	
	public void cancelTask(boolean isRun) {
		this.isRun = isRun;
	}
	
}
```
``` java
public class ThreadInterruptInNormal {

	public static void main(String[] args) {
		// TODO Auto-generated method stub
		NormalThread normalThread = new NormalThread();
		normalThread.start();
		try {
			Thread.sleep(1000);
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			System.out.println("=======主线程中断异常===");
			e.printStackTrace();
		}
		normalThread.cancelTask(false);
		//normalThread.interrupt();
		System.out.println("=======main方法执行结束===");
	}

}
```
打印结果：  
=======main方法执行结束===  
=======循环的次数=== 1367966380  

循环结果不同机器有不同结果，主线程睡眠一秒后子线程执行了取消任务动作，isRun为false，循环中断，然后run执行完退出。



###  线程stop方法
stop方法太过于暴力，强行把执行到一半的线程终止，会引起数据不一致的问题。比如下面暴力停止的例子
``` java
class SynchronizedObject {

	private String username = "a";
	private String password = "aa";

	public String getUsername() {
		return username;
	}

	public void setUsername(String username) {
		this.username = username;
	}

	public String getPassword() {
		return password;
	}

	public void setPassword(String password) {
		this.password = password;
	}

	synchronized public void printString(String username, String password) {
		try {
			this.username = username;
			Thread.sleep(600);
			this.password = password;
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}

```
``` java
class MyThread extends Thread {

	private SynchronizedObject object;

	public MyThread(SynchronizedObject object) {
		super();
		this.object = object;
	}

	@Override
	public void run() {
		object.printString("b", "bb");
	}
}
```
``` java
public class StopThread {
	public static void main(String[] args) {
		try {
			SynchronizedObject object = new SynchronizedObject();
			MyThread thread = new MyThread(object);
			thread.start();
			Thread.sleep(500);
			//thread.stop();//语句stop
			Thread.sleep(500);
			System.out.println(object.getUsername() + " " + object.getPassword());
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}
```
先把语句stop屏蔽掉，运行结果为：  
b  bb  

这个也是我们所期望的结果，因为主线程两次休眠500等于是休眠了1000ms，而子线程早已经休眠好了600ms，所以username和password变量得以顺利赋值。但如果把语句stop打开，再运行看看结果：  

b  aa

结果不同了，说明stop语句强行终止了线程导致数据不一致。具体原因：子线程一开始运行时就把username赋值为b，此时主线程还在第一次休眠的500ms中，等到过了500ms时刻，子线程还在剩下的100ms中休眠，但主线程开始执行了stop方法停止了子线程导致：  
this.password = password;  
这句话没有执行，所以password变量还是初始值aa


### 小结
以上讲了4种方法停止线程，第一种是自行结束线程，第4种stop方法已经过期、丢弃，常用的有第2、第3种方法，都是通过标志变量来判断，一个是中断标志位，一个是boolean变量，这是安全停止线程的方式。