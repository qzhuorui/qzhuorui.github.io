---
layout: post
title: '探索Window、Activity、View三者关系'
date: 2020-09-02
author: qzhuorui
color: rgb(255,210,32)
tags: Android
---



> 学习Android这一段时间来，总隐隐约约感觉Window和Activity及View间有某种链接作用但一直没专门学习过

# 探索Window、Activity、View三者关系

## 一、Activity的setContentView

Activity是Android开发使用最频繁的API之一，在接触它时，始终认为它就是负责将layout布局中的空间渲染绘制出来的。因为每次想显示一个界面时，都是通过startActivity方式；对于想显示的内容或布局，也只需要在Activity中加一行setContentView即可，剩下的都是Activity自动处理。但是从来没有去创建一个Window来绑定UI或View元素。

直到点开setContentView源码：

![1](/screenshot/探索Window、Activity、View三者关系 /1.png)

显然Activity几乎什么都没做，将操作直接交给了一个Window来处理。getWindow返回的是Activity中的全局变量mWindow，它是Window窗口类型。那么它是什么时候赋值的呢？

在学习Activity启动时，最终代码会调用到ActivityThread中的performLaunchActivity方法，通过反射创建Activity对象，并执行其attach方法。Window就是在这个方法中被创建，代码如下：

![2](/screenshot/探索Window、Activity、View三者关系 /2.png)

在Activity的attach方法中将mWindow赋值给一个PhoneWindow对象，实际上整个Android系统中Window只有一个实现类，就是PhoneWindow。

接下来调用setWindowManager方法，将系统WindowManager传给PhoneWindow，如下：

![3](/screenshot/探索Window、Activity、View三者关系 /3.png)

最终，在PhoneWindow中持有了一个WindowManagerImpl的引用

## 二、PhoneWindow的setContentView

Activity将setContentView的操作交给了PhoneWindow，接下来看看实现过程：

![4](/screenshot/探索Window、Activity、View三者关系 /4.png)

解释：

1. 如果mContentParent == null，则调用installDecor初始化DecorView和mContentParent
2. 将调用setContentView传入的布局添加到mContentParent中

可以看出在PhoneWindow中默认有一个DecorView（实际上是一个FrameLayout），在DecorView中默认自带一个mContentParent （实际上是一个ViewGroup）。我们自己实现的布局是被添加到mContentParent 中的，因此经过setContentView之后，PhoneWindow内部的View关系如下：

![5](/screenshot/探索Window、Activity、View三者关系 /5.gif)

**目前为止PhoneWindow中只是创建出了一个DecorView，并在DecorView中填充了我们在Activity中传入的layout布局**，可是DecorView还没有跟Activity建立任何联系，也没有被绘制到界面上。**那DecorView是何时被绘制到屏幕上的呢？**

刚接触Android学习生命周期时，经常看到介绍Activity执行到onCreate时并不可见，只有执行完onResume之后Activity 中的内容才是屏幕可见状态。造成这种现象的原因就是，onCreate阶段只是初始化了Activity需要显示的内容，而在onResume阶段才会将PhoneWindow中的DecorView真正的绘制到屏幕上！！！

在ActivityThread的handleResumeActivity中，会调用WindowManager的addView方法将DecorView添加到WMS（WindowManagerService）上，如下：

![6](/screenshot/探索Window、Activity、View三者关系 /6.png)

WindowManager的addView结果有两个：

1. DecorView被渲染绘制到屏幕上显示
2. DecorView可以接受屏幕触摸事件

## 三、WindowManager的addView

PhoneWindow只负责处理一些应用窗口通用的逻辑（设置标题栏，导航栏等）。但是真正完成把一个VIew作为窗口添加到WMS的过程是由WindowManager来完成的。

WindowManager是接口类型，上文中我们了解到它真正的实现者是WindowManagerImpl类，看一下它的addView方法：

![7](/screenshot/探索Window、Activity、View三者关系 /7.png)

WindowManagerImpl也是一个空壳，它调用了WindowManagerGlobal的addView方法

WindowManagerGlobal是一个单例，每一个进程中只有一个对象。在其addView方法中，创建了一个最关键的ViewRootImpl对象，然后通过ViewRootImpl的setView方法将view添加到WMS中。

## 四、ViewRootImpl的setView

![8](/screenshot/探索Window、Activity、View三者关系 /8.png)

解释：

1. requestLayout是刷新布局的操作，调用此方法后ViewRootImpl所关联的View也执行measure-layout-draw操作，确保在View被添加到Window上显示到屏幕之前，已经完成测量和绘制操作
2. 调用mWindowSession的addToDisplay方法将View添加到WMS中

WindowSession是WindowManagerGlobal中的单例对象，初始化代码如下：

![9](/screenshot/探索Window、Activity、View三者关系 /9.png)

sWindowSession实际上是IWindowSession类型，是一个Binder类型，真正的实现类是System进程中的Session。上图红框中就是用AIDL获取System进程中Session的对象。其addToDisplay方法，如下：

![10](/screenshot/探索Window、Activity、View三者关系 /10.png)

图中的mService就是WMS。至此，Window已经成功的传递给了WMS。剩下的工作就是全部转移到系统进程中的WMS来完成最终的添加操作。

## 五、再看Activity





























