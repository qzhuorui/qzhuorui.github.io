---
layout: post
title: 'startActivity启动过程分析'
date: 2020-09-01
author: qzhuorui
color: rgb(255,210,32)
tags: Android
---



> 学习下startActivity的具体流程，版本为Android-28

# startActivity启动过程分析

在手机桌面上点击某一个icon之后，实际上最终就是通过startActivity去打开某一个Activity页面。

在Android中一个App就相当于一个进程， **所以startActivity操作中还需要判断** ， **目标Activity的进程是否已经创建** ，如果没有，则在显示之前还需要将进程Process提前创建出来。

假设是从ActivityA跳到另一个App中的ActivityB，过程如下：

![1](/screenshot/startActivity启动过程分析/1.png)

整个**startActivity的流程分为3大部分** ，也**涉及3个进程之间的交互** ：

1. ActivityA -> ActivityManagerService（简称AMS）
2. ActivityManagerService -> ApplicationThread
3. ApplicationThread -> Activity

## 一、ActivityA -> ActivityManagerService

这过程并不复杂，用一张图表示：

![2](/screenshot/startActivity启动过程分析/2.png)

看下源码都做哪些操作：

### 1.Activity的startActivity

![3](/screenshot/startActivity启动过程分析/3.png)

最终调用startActivityForResult方法，传入的-1表示不需要获取startActivity的结果。

### 2.Activity的startActivityForResult

具体代码如下：

![4](/screenshot/startActivity启动过程分析/4.png)

startActivityForResult也很简单，调用Instrumentation.execStartActivity方法，剩下的交给Instrumentation类去处理

解释：

- Instrumentation类主要用来监控应用程序与系统交互
- 蓝框中的mMainThread是ActivityThread类型，ActivityThread可以理解为一个进程，在这就是A所在的进程
- mMainThread获取一个ApplicationThread的引用，这个引用就是用来实现进程间通信的，具体来说就是AMS所在系统进程通知应用程序进程来进行的一系列操作，这点稍后说

### 3.Instrumentation的execStartActivity

方法如下：

![5](/screenshot/startActivity启动过程分析/5.png)

在Instrumentation中，会通过ActivityManager.getServer获取AMS的实例，然后通过调用其startActivity方法，实际上这里就是通过AIDL来调用AMS的startActivity方法，至此， **startActivity的工作重心就从进程A转移到了系统进程AMS中** ！

## 二、ActivityManagerService -> ApplicationThread

接下来看下AMS中是如何一步步执行到B进程的

> 提前了解下：刚才在看Instrumentation时，提到一个ApplicationThread类，这个类是负责进程间通信的，这里AMS最终其实就是调用了B进程中的一个ApplicationThread引用，从而间接地通知B进程进行相应的操作

AMS --> ApplicationThread 流程，里面就干了2件事：

1. 综合处理launchMode和Intent中的Flag标志位，并根据结果生成一个目标ActivityB的对象（ActivityRecord）
2. 判断是否需要为目标ActivityB创建一个新的进程（ProcessRecord），新的任务栈（TaskRecord）

接下来就从AMS的startActivity方法开始看起：

### 1.AMS的startActivity

![6](/screenshot/startActivity启动过程分析/6.png)

从上图看出，经过多个方法调用，最终通过obtainStarter方法获取了ActivityStarter类型的对象，然后调用其execute方法。在execute方法中会再次调用其内部的startActivityMayWait方法

### 2.ActivityStarter的startActivityMayWait

ActivityStarter这个类看名字就知道它专门负责一个Activity的启动操作。它的主要作用包括解析Intent，创建ActivityRecord，如果有可能还要创建TaskRecord。startActivityMayWait方法的部分实现如下：

![7](/screenshot/startActivity启动过程分析/7.png)

从上图可以看出获取目标Activity信息的操作由mSupervisor来实现，它是ActivityStackSupervisor类型，从名字可以看出它**是主要负责Activity所处栈的管理类** 。

> 上图中resolveIntent中实际上是调用系统PackageManagerService来获取最佳Activity。有时我们通过隐式Intent启动Activity时，系统可能存在多个Activity可以处理Intent，此时会弹出一个选择框让用户选择具体打开哪个Activity，就是此处的逻辑处理结果。

在startActivityMayWait方法中调用了一个重载的startActivity方法，而最终会调用ActivityStarter中的**startActivityUnchecked方法来获取启动Activity的结果** 。

### 3.ActivityStarter的startActivityUnchecked

![8](/screenshot/startActivity启动过程分析/8.png)

解释：

- 图①处计算启动Activity的Flag值
- 图②处处理Task和Activity的进栈操作
- 图③处启动栈中顶部的Activity

#### a.computeLaunchingTaskFlags具体

![9](/screenshot/startActivity启动过程分析/9.png)

这个方法主要是计算启动Activity的Flag，不同的Flag决定了启动Activity最终会被放置到哪一个Task集合中

- ①处mInTask是TaskRecord类型，此处为null，代表Activity需要加入的栈不存在，因此需要判断是否需要新建Task
- ②处的mSourceRecord是ActivityRecord类型，它是用来描述“初始Activity”，什么是“初始Activity”？比如A启动了B，A就是初始Activity。当我们使用Context或Application启动Activity时，此SourceRecord为null
- ③处表示初始Activity如果是在SingleInstance栈中的Activity，这种需要添加NEW_TASK的标识。因此SingleInstance栈只能允许保存一个Activity
- ④处表示如果Launch Mode设置了singleTask或singleInstance，则也要创建一个新栈

### 4.ActivityStackSupervisor的startActivityLocked

![10](/screenshot/startActivity启动过程分析/10.png)

方法中会调用insertTaskTop方法尝试将Task和Activity入栈。如果Activity是以newTask的模式启动或TASK堆栈中不存在该Task id，则Task会重新入栈，并且放在栈的顶部。需要注意的是：Task先入栈，之后才是Activity入栈，他们是包含关系

这里了解下Stack，Task，Activity的关系，：它们都是在AMS内部维护的数据结构，关系如下：

![11](/screenshot/startActivity启动过程分析/11.png)

### 5.ActivityStack的resumeFocusedStackTopActivityLocked

![12](/screenshot/startActivity启动过程分析/12.png)

经过一系列调用，最终又回到了ActivityStackSupervisor中的startSpecificActivityLocked方法

#### a.ActivityStackSupervisor的startSpecificActivityLocked

![13](/screenshot/startActivity启动过程分析/13.png)

解释：

- ①处根据进程名称和Application和uid来判断目标进程是否已经创建，如果没有则代表未创建
- ②处调用AMS创建Activity所在进程

不管是目标进程已经存在还是新建目标进程，最终都会调用图中红线标记的realStartActivityLocked方法来执行启动Activity的操作

### 6.ActivityStackSupervisor的realStartActivityLocked

![14](/screenshot/startActivity启动过程分析/14.png)

这个方法在Android-27和28版本的区别很大，从28开始Activity的启动交给了事务（Transaction）来完成

- ①处创建Activity启动事务，并传入app.thread参数，它是ApplicationThread类型。ApplicationThread是为了实现进程间通信，是ActivityThread的一个内部类
- ②处执行Activity启动事务

Activity 启动事务的执行是由ClientLifecycleManager来完成的，具体如下：

![15](/screenshot/startActivity启动过程分析/15.png)

可以看出实际上是调用了启动事务ClientTransaction的schedule方法，而这个transaction实际上是在创建ClientTransaction时传入的app.thread对象，也就是ApplicationThread对象，如下：

![16](/screenshot/startActivity启动过程分析/16.png)

解释：

- 这里传入的app.thread会赋值给ClientTransaction的成员变量mClient，ClientTransaction会调用mClient.scheduleTransaction(this)来执行事务
- 这个app.thread是ActivityThread的内部类ApplicationThread，所以事务最终是调用app.thread的scheduleTransaction执行

到此为止，startActivity操作就从AMS转移到另一个进程B中的ApplicationThread中，剩下的就是AMS通过进程间通信机制通知ApplicationThread执行Activity-B的生命周期方法。

## 三、ApplicationThread -> Activity





















