---
layout: post
title: 内存泄漏及优化
categories: Android性能
description: 描述了Android app中内存泄漏常见的问题并给出优化方法
keywords: 内存泄漏, memory leak
---

&emsp;&emsp;内存泄漏(memory leak)在app开发中可能经常遇到，但不一定立即带来直接的危害，因为少数的泄漏并不会直接导致系统崩溃。但频繁地操作或日积月累后导致更多的内存无法释放，从而引起内存溢出，出现app卡死或者系统崩溃。因此，了解内存泄漏并给出优化对app性能来说非常重要。


#### 定义
不再被使用的对象持续占有内存或无用对象的内存得不到及时释放，从而造成内存空间的浪费称为内存泄漏。简单地说，不再被使用的对象的内存不能被回收引起内存泄漏。
Android常见内存泄漏：
* 静态变量
* 线程
* Handler的使用
* bitmap
* 单例模式
* 属性动画
*  

##### 静态变量
静态变量引用了实例对象比如activity，当activity退出时，静态变量还保持着activity对象的引用，activity对象所占用的内存不能释放
如下案例，

``` java
public class MainActivity extends AppCompatActivity {
    private Button btnStart;
    private static Context mContext;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mContext = this;
        btnStart = (Button) findViewById(R.id.btn_start);
        btnStart.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent = new Intent(MainActivity.this,SecondActivity.class);
                startActivity(intent);
                finish();
            }
        });
    }
}
```
点击按钮启动SecondActivity并退出MainActivity，静态变量mContext引用了MainActivity对象，静态变量的生命周期和app进程一致，即便MainActivity退出mContext仍然保持着MainActivity对象的引用，该对象的内存不能释放。  
下面用Android Device Monitor、MAT查看是否真的有内存泄漏，在Android Studio打开Android Device Monitor，如下图  

![](/images/posts/android/post2_memory_leak/initial_heap.png '初始堆对象大小')

查看data Object一行，看Total Size大小，其值就是当前进程中所有Java数据对象的内存总量，可以通过此值判断是否有泄漏。
不断地操作app中有可能出现泄漏的地方，同时观察data object的Total Size值
* 正常情况下Total Size值会稳定在一个有限的范围内，即时值变大，过了一会又会回落。虚拟机不断地进行GC过程，无用的对象都被回收了，内存占用量会会落到一个稳定的水平。也就是说程序中的代码良好，没有造成对象不被垃圾回收的情况。
* 不正常情况下，随着操作的不断进行，Total Size值会不断地增加，即便有回落，也会回落一个很高的值，而且值还会增加，说明有部分对象没有被GC回收，明显有内存泄漏。

如上图红色方框标注，app初始堆大小为305.523KB,然后点击按钮启动SecondActivity(通过代码分析该过程存在泄漏的可能性)，再点击Cause GC发现内存先突然增高然后回落到一个值，如下图

![](/images/posts/android/post2_memory_leak/final_heap.png '启动secondactivity并退出mainactivity后堆大小')

堆内存先曾加到390多，又回落到346.766，虽然回落了，但比初始的305.523还是增加了约40KB，而且刷新Cause GC后发现不会变小，说明这40KB的内存没有被回收，有内存没有释放，肯定有泄漏。这种方法能够帮助分析确定释放有内存泄漏，但不能确定具体何处泄漏

这就要用到一些工具辅助分析，比如MAT。在AS或Eclipse的Android Deveice Monitor中，先点击Update Heap按钮，再把容易引起泄漏的过程操作一遍(本文就是再启动secondactivity)，然后点击Dump Hprof file获取到hprof
文件，弹出对话框保存到起来。  
MAT是eclipse中的一个内存分析工具，在eclipse中可以直接安装MAT插件，安装好后，点击Dump Hprof file后等几秒钟会自动弹出MAT图形界面；如果在AS中，保存好hprof文件后，不会自动弹出MAT界面，还需要下载MAT工具包。

[点击下载MAT](https://www.eclipse.org/mat/)  

下载好后解压，点击MemoryAnalyzer.exe启动MAT，如下图

![](/images/posts/android/post2_memory_leak/mat.png 'mat主界面')

然后点击File-->Open Heap File打开hprof文件，出现错误：

![](/images/posts/android/post2_memory_leak/hprof_error.png 'hprof文件打开错误')

出错原因在于文件格式错误，需要用sdk工具hprof-conv转换。
在cmd中cd到sdk的platform-tools目录，然后用命令
hprof-conv  xxx.hprof  yyy.hprof
把xxx.hprof源文件转换成yyy.hprof，xxx是转换前的名字，yyy是转换后的名称

我的写法：
H:\Program Files (x86)\Android\android-sdk\platform-tools>hprof-conv &emsp;&emsp; E:\Android内存泄漏\leak.com.zhulf.www.memoryleak.hp
rof  &emsp;&emsp;  E:\Android内存泄漏\leak.hprof

再用MAT工具打开yyy.hprof文件，成功后显示如下

![](/images/posts/android/post2_memory_leak/hprof_dialog.png 'hprof文件打开显示对话框')

点击finish按钮如下

![](/images/posts/android/post2_memory_leak/hprof_success.png 'hprof文件打开成功')

如果在eclipse中操作，点击Help-->Eclipse Marketplace，在输入框中输入MAT搜索，搜到后点击install按钮安装MAT，之后点击Dump Hprof file后等几秒钟会自动弹出MAT图形界面，和上图一样。

在Overview中选择Dominator Tree(支配树)如下

![](/images/posts/android/post2_memory_leak/dominator_1.png 'dominator树_1')

图中列出了很多对象，在红框处输入本案例所关心的包名memoryleak，检索出相关信息  
注：调试内存泄漏时，只要检查所关心的类，无关的类不需要检查

![](/images/posts/android/post2_memory_leak/dominator_2.png 'dominator树_2')

Shallow Heap: 对象本身占用内存的大小，不包含对其他对象的引用
Retained Heap: 该对象自己的shallow size，加上从该对象能直接或间接访问到对象的shallow size之和

一般选择Retained Heap最大的那个对象，表示引用的对象较多，占的内存较大，存在内存被占用的可能性。右键选择Paths To GC Roots-->exclude weak/soft references，此处是排除弱引用、软引用，弱引用和软引用易被GC回收，排除了能够被
回收的对象后，就有可能看到没有被回收的对象，此处给出GC Roots的含义：  
JAVA虚拟机通过可达性（Reachability)来判断对象是否存活，基本思想：以”GCRoots”的对象作为起始点向下搜索，搜索形成的路径称为引用链，当一个对象到GC Roots没有任何引用链相连（即不可达的），则该对象被判定为可以被回收的对象，反之不能被回收。  
再来看另外一篇博客关于GC的定义：  
The so-called GC (Garbage Collector) roots are objects special for garbage collector. Garbage collector collects those objects that are not GC roots and are not accessible by references from GC roots.  
这段话引用自[这个博客](https://www.yourkit.com/docs/java/help/gc_roots.jsp) 
中文翻译为：
所谓的GC根是垃圾收集器中特有的对象，垃圾收集器收集那些不是GC根的对象，以及不能通过GC根的引用树访问的对象。
也就是说，如果可以通过GC roots树访问到的对象，该对象要么是RC Roots对象，要么是“被引用”的对象，而这些“被引用”的对象是没有被回收的，是重点要观察的对象，是有可能发生泄漏的对象。

排除软引用和弱引用后如下图

![](/images/posts/android/post2_memory_leak/gc_roots.png 'gc roots引用')

红色框标注的mContext变量出现在第一行MainActivity对象的path树中，就表明mContext引用了MainActivity对象，MainActivity对象仍然存在，其占有的内存没有被回收，这就可以断定MainActivity对象存在内存泄漏，或者说该apk存在内存泄漏。  
mContext右边的一串字符
``` java
class leak.com.zhulf.www.memoryleak.MainActivity @ 0x12cef000
```
表示该mContext变量属于MainActivity这个类的，代码中可以看到mContext是MainActivity声明的静态变量。如果mContext是其他类中
声明的静态变量，那么右边一串字符就会是其他类的名称。

解决方法  
把mContext改成实例变量，当对象退出时实例变量都会被回收。尽量不要用静态变量引用实例对象，很容易导致泄漏。
此时再用MAT测试一下，排除弱引用和弱引用后发现该对象没有被其他变量引用。

##### 线程
Activity中使用线程不当也会带来内存泄漏，看一个例子
``` java
package leak.com.zhulf.www.memoryleak;

import android.content.Intent;
import android.os.Bundle;
import android.support.v7.app.AppCompatActivity;
import android.view.View;
import android.widget.Button;

public class MainActivity extends AppCompatActivity {
    private Button btnStart;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        btnStart = (Button) findViewById(R.id.btn_start);
        btnStart.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent = new Intent(MainActivity.this, SecondActivity.class);
                startActivity(intent);
                finish();
            }
        });
		new Thread(new InnerThread()).start();
    }

    private class InnerThread implements Runnable {

        @Override
        public void run() {
            try {
                Thread.sleep(100000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

```
启动MainActivity同时启动线程，线程等待100s，然后单击按钮销毁MainActivity，但MainActivity并没有销毁，为何？用上一节提到的查看堆大小的方式发现存在内存泄漏，再用MAT查看泄漏原因

![](/images/posts/android/post2_memory_leak/thread_leak.png 'MAT抓取线程泄漏')

根据内部类的特点，内部类会持有一个指向外部类对象的引用。如上图，InnerThread就是内部类，this$0就代表指向外部类对象的引用。第三行的taget就是线程对象持有内部类InnerThread的一个引用。活着的线程被认为是GC ROOTS，被GC ROOTS直接引用或者间接引用的对象是不能被释放的。当前线程持有了一个InnerThread对象引用，而InnerThread对象又持有一个MainActivity对象引用，导致MainActivity没有释放。
解决方法
* 把InnerThread修改成静态内部类，根据java内部类特性，静态内部类不持有外部类引用
* 把InnerThread作为外部类使用



