---
layout: post
title: Activity四种启动模式
categories: Android应用层
description: 自学 Python 核心编程过程中对课后习题的答案记录。
keywords: launchMode, 启动模式, activity, 活动
---

&emsp;&emsp;在AndroidManifest.xml中配置activity时，android:launchMode属性可用来指定启动activity的模式：standard、singleTop、singleTask、singleInstance。

这四种模式一般配合Intent属性变量FLAG_ACTIVITY_XXX使用，比如FLAG_ACTIVITY_NEW_TASK，本文暂时撇开FLAG_ACTIVITY_XXX，只讨论这四种模式的启动结果。
#### standard
写一个FirstActivity，其AndroidManifest.xml中启动模式配置如下：
``` java
<activity
	android:name=".FirstActivity"
	android:label="@string/app_name"
	android:launchMode="standard" >
	<intent-filter>
		<action android:name="android.intent.action.MAIN" />
		<category android:name="android.intent.category.LAUNCHER" />
	</intent-filter>
</activity>
```
假设布局中只有一个按钮，点击按钮启动FirstActivity自己：
``` java
Intent intent = new Intent(FirstActivity.this, FirstActivity.class);
startActivity(intent);
```
默认的启动模式就是Standard，此处为了描述与其他模式的区别，特别地加上android:launchMode="standard" 属性。
运行代码之前，用命令dumpsys activity activities查看栈信息(此处省略多余的信息，只贴上相关部分)：
```java
* Hist #1: ActivityRecord{53640264 u0 com.andevele.www.launchmode/.FirstActivity}
```
FirstActivity位于栈顶，栈下是Launcher。除launcher之外，栈中只有一个FirstActivity实例。单击按钮启动FirstActivity自身后的情况：
```java
* Hist #2: ActivityRecord{53633f40 u0 com.andevele.www.launchmode/.FirstActivity}
* Hist #1: ActivityRecord{535746e0 u0 com.andevele.www.launchmode/.FirstActivity}
```

栈中存在两个FirstActivity，被启动的位于栈顶，启动者位于其下方，共有2个FirstActivity活动实例。

结论：Standard模式每启动一次，就创建一个新的Activity对象，启动多次就会创建多个对象，每个对象都有不同的内存地址，互不相等。

#### singleTop
情景1：A仍然为标准模式，B为SingleTop模式，A启动B，B中点击按钮再启动B，多次重复启动B
```java
* Hist #2: ActivityRecord{5373be00 u0 com.andevele.www.launchmode/.SecondActivity}
    packageName=com.andevele.www.launchmode processName=com.andevele.www.launchmode
* Hist #1: ActivityRecord{535746e0 u0 com.andevele.www.launchmode/.FirstActivity}
    packageName=com.andevele.www.launchmode processName=com.andevele.www.launchmode
```
如上图，观察栈中B活动的实例个数仍就为1个，在B中点击多次按钮重复启动B，都不会创建新的实例

情景2：A启动B，B启动C，C再启动B，其中，B为SingleTop模式，按照上面的理解，栈中的B实例是不是也只有一个呢？
```java
* Hist #4: ActivityRecord{5364b6ec u0 com.andevele.www.launchmode/.SecondActivity}
* Hist #3: ActivityRecord{53640264 u0 com.andevele.www.launchmode/.ThirdActivity}
* Hist #2: ActivityRecord{535746e0 u0 com.andevele.www.launchmode/.SecondActivity}
* Hist #1: ActivityRecord{53715c50 u0 com.andevele.www.launchmode/.FirstActivity}
```
栈顶是第二个活动SecondActivity即B的一个实例，在A的上面还有一个B的实例，在看ActivityRecord后面的内存地址是不一样的，总共有2个B实例，看来这和上面的理解不一样

结论：当活动为SingleTop模式时，若栈中已有一个实例位于栈顶，就用该实例；若栈中已有一个实例但不在栈顶，每次启动还会创建一个新的实例。

#### singleTask
情景1：A仍然为标准模式，B为singleTask模式，A启动B，B中点击按钮再启动B，多次重复启动B
这种情况下和SingleTop一样，无论启动多少次，栈中只有一个实例B
情景2：B为singleTask模式，A---->B---->C---->B
看看这种情况和SingleTop的是否一样：
```java
* Hist #2: ActivityRecord{536a3c88 u0 com.andevele.www.launchmode/.SecondActivity}
* Hist #1: ActivityRecord{5364edf0 u0 com.andevele.www.launchmode/.FirstActivity}
```
看来结果是不一样的，栈顶是B，C竟然不在栈中了。当A---->B---->C后，C位于栈顶，然后C启动B时，Android系统会在栈中查找是否有B的实例，发现有，就会重新启动B，并把位于该实例之上的C移出栈。

结论：当活动为singleTask模式时，若栈中已有一个实例位于栈顶，就用该实例；若栈中已有一个实例但不在栈顶，系统会把位于该实例之上的所有其他实例都移出栈，再将该实例放到栈顶。
这种模式和SingleTop的区别就在于是否有其他活动对象位于被启动对象之上，若有，就会把这些活动统统移出栈。
#### singleInstance
该模式和其他三个模式最根本的区别在于：被启动的活动对象会放到一个新的栈中。什么意思？先看一个情景1：
B为singleInstance模式，A---->B
启动后，栈中信息如下：
```java
  * TaskRecord{536fb518 #14 A com.andevele.www.launchmode U 0}
    numActivities=1 rootWasReset=false userId=0
    affinity=com.andevele.www.launchmode
    intent={cmp=com.andevele.www.launchmode/.SecondActivity}
    realActivity=com.andevele.www.launchmode/.SecondActivity
    askedCompatMode=false
    lastThumbnail=null lastDescription=null
    lastActiveTime=4635631 (inactive for 5s)
    * Hist #2: ActivityRecord{53630f68 u0 com.andevele.www.launchmode/.SecondActivity}
        packageName=com.andevele.www.launchmode processName=com.andevele.www.launchmode
        launchedFromUid=10038 userId=0
        app=ProcessRecord{5362a2ac 7871:com.andevele.www.launchmode/u0a10038}
        Intent { cmp=com.andevele.www.launchmode/.SecondActivity }
        frontOfTask=true task=TaskRecord{536fb518 #14 A com.andevele.www.launchmode U 0}
        taskAffinity=com.andevele.www.launchmode
        realActivity=com.andevele.www.launchmode/.SecondActivity
        baseDir=/data/app/com.andevele.www.launchmode-2.apk
        dataDir=/data/data/com.andevele.www.launchmode
        stateNotNeeded=false componentSpecified=true isHomeActivity=false
        compat={320dpi} labelRes=0x7f050000 icon=0x7f020000 theme=0x7f060001
        config={1.0 310mcc260mnc zh_CN ldltr sw360dp w360dp h615dp 320dpi nrml long port finger qwerty/v/v dpad/v s.8}
        launchFailed=false haveState=false icicle=null
        state=RESUMED stopped=false delayedResume=false finishing=false
        keysPaused=false inHistory=true visible=true sleeping=false idle=true
        fullscreen=true noDisplay=false immersive=false launchMode=3
        frozenBeforeDestroy=false thumbnailNeeded=false forceNewConfig=false
        thumbHolder: 536fb518 bm=null desc=null
        waitingVisible=false nowVisible=true lastVisibleTime=-4s985ms
  * TaskRecord{536bc9e4 #13 A com.andevele.www.launchmode U 0}
    numActivities=1 rootWasReset=false userId=0
    affinity=com.andevele.www.launchmode
    intent={act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.andevele.www.la
unchmode/.FirstActivity}
    realActivity=com.andevele.www.launchmode/.FirstActivity
    askedCompatMode=false
    lastThumbnail=android.graphics.Bitmap@538225bc lastDescription=null
    lastActiveTime=4635617 (inactive for 5s)
    * Hist #1: ActivityRecord{5370a6d4 u0 com.andevele.www.launchmode/.FirstActivity}
        packageName=com.andevele.www.launchmode processName=com.andevele.www.launchmode
        launchedFromUid=0 userId=0
        app=ProcessRecord{5362a2ac 7871:com.andevele.www.launchmode/u0a10038}
        Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.andevele.w
ww.launchmode/.FirstActivity }
        frontOfTask=true task=TaskRecord{536bc9e4 #13 A com.andevele.www.launchmode U 0}
        taskAffinity=com.andevele.www.launchmode
        realActivity=com.andevele.www.launchmode/.FirstActivity
        baseDir=/data/app/com.andevele.www.launchmode-2.apk
        dataDir=/data/data/com.andevele.www.launchmode
        stateNotNeeded=false componentSpecified=true isHomeActivity=false
        compat={320dpi} labelRes=0x7f050000 icon=0x7f020000 theme=0x7f060001
        config={1.0 310mcc260mnc zh_CN ldltr sw360dp w360dp h615dp 320dpi nrml long port finger qwerty/v/v dpad/v s.8}
        launchFailed=false haveState=true icicle=Bundle[mParcelledData.dataSize=1160]
        state=STOPPED stopped=true delayedResume=false finishing=false
        keysPaused=false inHistory=true visible=false sleeping=false idle=true
        fullscreen=true noDisplay=false immersive=false launchMode=0
        frozenBeforeDestroy=false thumbnailNeeded=false forceNewConfig=false
        thumbHolder: 536bc9e4 bm=android.graphics.Bitmap@538225bc desc=null
        waitingVisible=false nowVisible=false lastVisibleTime=-16s468ms
```
整个系统栈中有2个栈(此处先省略launcher所在的栈):TaskRecord{536fb518、TaskRecord{536bc9e4
前者存在一个B实例，后者存在一个A实例，当A启动B时，系统会新创建以一个任务栈单独存放B，和等其他活动都隔开存放，而在前面三种模式中都会放在相同的任务栈中存放，这就是最根本的区别。

当多次启动B时，B所在的栈中还是只有一个实例，这种说法和singleTop/singleTask一样，只不过位于不同的栈而已；

再看看情景2：B为singleInstance模式，A---->B---->C---->B
当A---->B---->C，此时C和A所在的栈(我们称为栈1)在上面，B所在的栈(称为栈2)在下面或者说在整个系统栈的栈底。
然后C再启动B时，应该发生什么？

先从全局来看，栈2又来到了系统栈的栈顶了，即栈1位于栈2的下方了。然后从局部来看，栈2中B的实例还是1个。从下面的栈信息可以验证这一点：
```java
Main stack:
* TaskRecord{537238f8 #16 A com.andevele.www.launchmode U 0}
  numActivities=1 rootWasReset=false userId=0
  affinity=com.andevele.www.launchmode
  intent={flg=0x400000 cmp=com.andevele.www.launchmode/.SecondActivity}
  realActivity=com.andevele.www.launchmode/.SecondActivity
  askedCompatMode=false
  lastThumbnail=android.graphics.Bitmap@538225bc lastDescription=null
  lastActiveTime=5534402 (inactive for 3s)
  * Hist #3: ActivityRecord{5357e80c u0 com.andevele.www.launchmode/.SecondActivity}
      packageName=com.andevele.www.launchmode processName=com.andevele.www.launchmode
      launchedFromUid=10038 userId=0
      app=ProcessRecord{536a29b0 8476:com.andevele.www.launchmode/u0a10038}
      Intent { cmp=com.andevele.www.launchmode/.SecondActivity }
      frontOfTask=true task=TaskRecord{537238f8 #16 A com.andevele.www.launchmode U 0}
      taskAffinity=com.andevele.www.launchmode
      realActivity=com.andevele.www.launchmode/.SecondActivity
      baseDir=/data/app/com.andevele.www.launchmode-1.apk
      dataDir=/data/data/com.andevele.www.launchmode
      stateNotNeeded=false componentSpecified=true isHomeActivity=false
      compat={320dpi} labelRes=0x7f050000 icon=0x7f020000 theme=0x7f060001
      config={1.0 310mcc260mnc zh_CN ldltr sw360dp w360dp h615dp 320dpi nrml long port finger qwerty/v/v dpad/v s.8}
      launchFailed=false haveState=false icicle=null
      state=RESUMED stopped=false delayedResume=false finishing=false
      keysPaused=false inHistory=true visible=true sleeping=false idle=true
      fullscreen=true noDisplay=false immersive=false launchMode=3
      frozenBeforeDestroy=false thumbnailNeeded=false forceNewConfig=false
      thumbHolder: 537238f8 bm=android.graphics.Bitmap@538225bc desc=null
      waitingVisible=false nowVisible=true lastVisibleTime=-2s962ms
* TaskRecord{536e8ea4 #15 A com.andevele.www.launchmode U 0}
  numActivities=2 rootWasReset=false userId=0
  affinity=com.andevele.www.launchmode
  intent={act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.andevele.www.la
chmode/.FirstActivity}
  realActivity=com.andevele.www.launchmode/.FirstActivity
  askedCompatMode=false
  lastThumbnail=android.graphics.Bitmap@5380ab3c lastDescription=null
  lastActiveTime=5534379 (inactive for 3s)
  * Hist #2: ActivityRecord{536b18e4 u0 com.andevele.www.launchmode/.ThirdActivity}
      packageName=com.andevele.www.launchmode processName=com.andevele.www.launchmode
      launchedFromUid=10038 userId=0
      app=ProcessRecord{536a29b0 8476:com.andevele.www.launchmode/u0a10038}
      Intent { flg=0x400000 cmp=com.andevele.www.launchmode/.ThirdActivity }
      frontOfTask=false task=TaskRecord{536e8ea4 #15 A com.andevele.www.launchmode U 0}
      taskAffinity=com.andevele.www.launchmode
      realActivity=com.andevele.www.launchmode/.ThirdActivity
      baseDir=/data/app/com.andevele.www.launchmode-1.apk
      dataDir=/data/data/com.andevele.www.launchmode
      stateNotNeeded=false componentSpecified=true isHomeActivity=false
      compat={320dpi} labelRes=0x7f050000 icon=0x7f020000 theme=0x7f060001
      config={1.0 310mcc260mnc zh_CN ldltr sw360dp w360dp h615dp 320dpi nrml long port finger qwerty/v/v dpad/v s.8}
      launchFailed=false haveState=true icicle=Bundle[mParcelledData.dataSize=1160]
      state=STOPPED stopped=true delayedResume=false finishing=false
      keysPaused=false inHistory=true visible=false sleeping=false idle=true
      fullscreen=true noDisplay=false immersive=false launchMode=0
      frozenBeforeDestroy=false thumbnailNeeded=false forceNewConfig=false
      thumbHolder: 536e8ea4 bm=android.graphics.Bitmap@5380ab3c desc=null
      waitingVisible=false nowVisible=false lastVisibleTime=-4m5s166ms
  * Hist #1: ActivityRecord{53712c80 u0 com.andevele.www.launchmode/.FirstActivity}
      packageName=com.andevele.www.launchmode processName=com.andevele.www.launchmode
      launchedFromUid=0 userId=0
      app=ProcessRecord{536a29b0 8476:com.andevele.www.launchmode/u0a10038}
      Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.andevele.w
.launchmode/.FirstActivity }
      frontOfTask=true task=TaskRecord{536e8ea4 #15 A com.andevele.www.launchmode U 0}
      taskAffinity=com.andevele.www.launchmode
      realActivity=com.andevele.www.launchmode/.FirstActivity
      baseDir=/data/app/com.andevele.www.launchmode-1.apk
      dataDir=/data/data/com.andevele.www.launchmode
      stateNotNeeded=false componentSpecified=true isHomeActivity=false
      compat={320dpi} labelRes=0x7f050000 icon=0x7f020000 theme=0x7f060001
      config={1.0 310mcc260mnc zh_CN ldltr sw360dp w360dp h615dp 320dpi nrml long port finger qwerty/v/v dpad/v s.8}
      launchFailed=false haveState=true icicle=Bundle[mParcelledData.dataSize=1160]
      state=STOPPED stopped=true delayedResume=false finishing=false
      keysPaused=false inHistory=true visible=false sleeping=false idle=true
      fullscreen=true noDisplay=false immersive=false launchMode=0
      frozenBeforeDestroy=false thumbnailNeeded=false forceNewConfig=false
      thumbHolder: 536e8ea4 bm=android.graphics.Bitmap@5380ab3c desc=null
      waitingVisible=false nowVisible=false lastVisibleTime=-4m8s583ms
```
Main stack表示真个系统栈，所有的信息都位于这个大栈(可以称为祖父栈)中，大栈中有分为1个或多个小的任务栈TaskRecord。

结论：当活动为singleInstance模式时，和singleTask情况几乎一样，无论启动多少次，栈中只有一个实例。区别在于：系统不会把位于该活动之上的其他活动实例都移出栈，而是把他们所在的任务栈都放到该活动之下。

#### 退栈的顺序
如果B为singleTask模式，A---->B---->C---->B，按下返回键退出栈的顺序？
- C启动B后栈中只有A和B，B先出栈，接着是A

如果B为singleInstance模式，A---->B---->C，按下返回键退出栈的顺序？
- C先退出，接着A位于栈顶，B所在的栈位于A的栈的下方，然后A退栈，最后B退栈。C退出时为何没有先显示B？因为C和A是同一个栈，当C退出时，从局部考虑，和C同栈的A会先显示到栈顶。

如果A---->B---->C---->B，按下返回键退出栈的顺序？
- 当C再次启动B时，B所在的栈位于栈顶，按下返回键时，B先退栈，B所在的栈也退出，而后A和C所在的栈位于系统栈的栈顶，同样A和C所在的小栈中，C位于栈顶，C先退出，最后是A。

对于singleInstance模式来说，无论是进栈还是出栈，都先以任务栈为整体来考虑的。


