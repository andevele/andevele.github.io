---
layout: post
title: 内存泄漏及优化
categories: Android性能
description: 描述了Android app中内存泄漏常见的问题并给出优化方法
keywords: 内存泄漏, memory leak
---

&emsp;&emsp;内存泄漏(memory leak)在app开发中可能经常遇到，但不易被发现，因为少数的泄漏并不会引起系统直接崩溃。但频繁地操作或日积月累后引起内存溢出，从而出现app卡死或者直接退出的现象。因此，充分了解内存泄漏并给出优化对app性能来说非常重要。


#### 定义
不再被使用的对象持续占有内存或无用对象的内存得不到及时释放，从而造成内存空间的浪费称为内存泄漏。简单地说，不再被使用的对象的内存不能被回收就会引起内存泄漏。
Android内存泄漏常见的场合：
* 静态变量
* Handler的使用
* 线程
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



除非干掉app进程，但总不能以杀死进程的方法来解决


``` java
Intent intent = new Intent(FirstActivity.this, FirstActivity.class);
startActivity(intent);
```
默认的启动模式就是Standard，此处为了描述与其他模式的区别，特别地加上android:launchMode="standard" 属性。


