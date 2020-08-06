---
layout: post
title: 'Android是如何通过Activity进行交互的'
date: 2020-08-06
author: qzhuorui
color: rgb(154,133,255)
tags: Android
---



# Android是如何通过Activity进行交互的？

> 主要总结下使用startActivity时需要注意的点

## 一、taskAffinity

对于Activity的启动模式而言，我们可以通过设置不同的启动模式，来实现调配不同的Task。但是taskAffinity在一定程度上也会影响任务栈的调配流程。

每一个Activity都有一个Affinity属性，如果不在清单文件中指定，默认为当前应用的包名。taskAffinity主要有以下几点需要注意：

### 1.taskAffinity会默认使Activity在新的栈中分配吗？

通过一个例子来验证一下，在一个Android项目TaskAffinity中，创建两个Activity：First和Second，除了Activity类名之外，其他都是默认配置。这种情况下，点击First中的Button，从First跳转到Second。通过命令 `adb shell dumpsys activity activities` 上述命令会将系统中所有存活中的Activity信息打印到控制台，这时我们可以看到在一个任务栈中存在两个Activity实例，并且Second处于栈顶。

接下来修改下Second的taskAffinity，将其改为“second.affinity"，使它和First的taskAffinity不同。这时再查看任务栈中的情况，可以看到虽然First和Second的taskAffinity不同，但是他们都被创建在一个任务栈中。

但是如果我们再将Second的launchMode改为singleTask，再次运行就会发现两个Activity会被分配到不同的任务栈中。

**结论：** 单纯使用taskAffinity不能导致Activity被创建在新的任务栈中，需要配合singleTask或singleInstance！！！

### 2.taskAffinity + allowTaskReparenting

allowTaskReparenting赋予Activity**在各个Task中间转移的特性** 。一个在**后台任务栈中的Activity A** ，当有**其他任务进入前台** ，并且**taskAffinity与A相同** ，则会**自动将A添加到当前启动的任务栈中** 。

举例：

1. 在外卖APP中下好订单后，跳转到Alipay进行支付。当在Alipay中支付成功之后，页面停留在Alipay支付成功界面。
2. 按下Home间，在主页面重新打开Alipay，页面上显示的并不是Alipay主页面，而是之前的支付成功页面
3. 再次进入外卖app，可以发现支付成功页面已经消失

上面的现象原因就是allowTaskReparenting属性，通过代码演示下：

分别创建两个project过程：First和TaskAffinityReparent

- 在First中有三个Activity：FirstA,FirstB,FirstC。打开顺序依次是A-B-C。其中FirstC的taskAffinity为"first.affinity"，且allowTaskReparenting属性设置为true。FirstA和FirstB为默认值
- TaskAffinityReparent中只有一个Activity--ReparentActivity，并且TaskAffinity也等于"first.affinity"

先打开First app，并从FirstA开始跳转到B-C。然后按下Home键，使其进入后台。

接下来打开TaskAffinityReparent app，屏幕上本应显示ReparentActivity的页面，但实际却显示的时First C中的页面，通过命令可以看到，First C被移动到与ReparentActivity在同一个任务栈中。此时First C位于栈顶，再次点返回键，才会显示ReparentActivity页面

## 通过Binder传递数据的限制

### 1.Binder传递数据限制

Activity界面跳转时，使用Intent传递数据是最常用的操作。但是Intent传值偶尔也会导致程序崩溃。如下代码：

![2](/screenshot/Android是如何通过Activity进行交互的/1.png)

在startFirstB方法中，跳转FirstB页面，并通过Intent传递Bean类中的数据。但是执行上述代码会报错：

![3](/screenshot/Android是如何通过Activity进行交互的/2.png)

log中的意思是Intent传递数据过大，最终原因是Android系统对使用**Binder传数据进行了限制** 。通常为1M，但根据不同版本，厂商，也会有变化。

**解决方法** ：

1. 减少通过Intent传递的数据，将非必须字段使用transient关键字修饰，修饰上面的 `byte[] data` 从而避免将其序列化。
2. 将对象转化为JSON字符串，减少数据体积。因为JVM加载类通常会伴随额外的空间来保存类相关信息，将类中数据转化为JSON字符串可以减少数据大小。比如使用Gson.toJson方法。大多时候将类转化为JSON后还是会超出Binder限制。这时候需要考虑本地持久化来实现数据共享，或者使用EventBus来实现数据传递。

## process造成多个Application

我们通常会在自定义的Application中做一些初始化的操作，比如app分包，推送初始化，库的全局配置等。但实际上， **Activity可以在不同的进程中启动，而每一个不同的进程都会创建出一个Application** ，因此有可能造成Application的onCreate()被多次执行。

比如在Manifest中为某个Activity配置process属性，这将导致它会在一个新的进程中创建。当从一个Activity在跳转到它时，Application会再被创建。

**解决：** 

1. onCreate()中判断进程的名称，只有在符合的进程中，才执行初始化操作
2. 抽象出一个与Application生命周期同步的类，并根据不同的进程创建相应的Application实例

# 总结

主要说了：

1. taskAffinity实现任务栈的调配
2. 通过Binder传递数据的限制
3. 多进程应用可能会造成的问题

