---
layout: post
title: 'Android如何通过View进行渲染'
date: 2020-09-03
author: qzhuorui
color: rgb(255,210,32)
tags: Android
---



>承上Activity、Window、View之间的关系

# Android如何通过View进行渲染

在另一篇文章中学习Activity、Window、View之间的关系时，我们了解了ViewRootImpl在整个流程中起着承上启下的作用。

1. ViewRootImpl中通过Binder通信机制，远程调用WindowSession将View添加到Window中
2. ViewRootImpl在添加View之前，又需要调用requestLayout方法，执行完整的View树的渲染操作

## 一、屏幕绘制

### 1.ViewRootImpl requestLayout流程

![1](/screenshot/Android如何通过View进行渲染/1.png)

requestLayout第一次被调用是在setView方法中，从名字也可以看出，这个方法主要目的就是请求布局操作，其中包括VIew的测量，布局，绘制等。具体如下：

![2](/screenshot/Android如何通过View进行渲染/2.png)

说明：

1. 检查是否为合法线程，一般情况下就是检查是否为主线程
2. 将请求布局标识符设置为true，这个参数决定了后续是否需要执行measure和layout操作。

最后执行scheduleTraversals方法，如下：

![3](/screenshot/Android如何通过View进行渲染/3.png)

说明：

1. 向主线程消息队列中插入SyncBarrier Message。该方法发送了一个没有target的Message到Queue中，在next方法中获取消息时，如果发现没有target的Message，则在一定的时间内跳过同步消息，优先执行异步消息。这里通过调用此方法，保证UI绘制操作优先执行
2. 调用Choreographer的postCallback方法，实际上也是发送一个Message到主线程消息队列

Choreographer的postCallback的执行流程如下：

![4](/screenshot/Android如何通过View进行渲染/4.png)

可以看出最终通过Handler发送到MessageQueue中的Message被设置为异步类型的消息。

mTraversalRunable是一个实现Runnable接口的TraversalRunable类型对象，其run方法如下：

![5](/screenshot/Android如何通过View进行渲染/5.png)

可以看出，在run方法中调用了doTraversal方法，并最终调用了performTraversal()方法，这个方法就是真正的开始View绘制流程：measure -> layout -> draw

### 2.ViewRootImpl的performTraversal方法

这个方法是一个比较重的方法，查看源码发现一共将近900行。但是抽取下核心，这个方法实际上只做了3件事：

![6](/screenshot/Android如何通过View进行渲染/6.png)

很明显，实际就是执行了我们在定义View中的3个主要过程

以测量performMeasure实现举例

### 3.ViewRootImpl 的 measureHierarchy

View的测量是一层递归调用，递归执行子View的测量工作之后，最后决定父视图的宽和高。但是这个递归的起源是在哪呢？就是DecorView！！！因为在measureHierarchy方法中最终是调用performMeasure方法来进行测量工作的，所以直接看performMeasure方法的实现，如下：

![7](/screenshot/Android如何通过View进行渲染/7.png)

在这个方法中，通过getRootMeasureSpec方法获取了根View的MeasureSpec，实际上MeasureSpec中的宽高此处获取的值是Window的宽高。关于MeasureSpec介绍可以看下之前的自定义View内容

### 4.ViewRootImpl 的performMeasure 

![8](/screenshot/Android如何通过View进行渲染/8.png)

这个很简单，只是执行了mView的measure方法，这个mView就是DecorView。其DecorView的measure方法中，会调用onMeasure方法，而DecorView是继承自FrameLayout的，因此最终会执行FrameLayout中的onMeasure方法，并递归调用子View的onMeasure方法

> performLayout也是类似的过程，不再赘述

### 5.ViewRootImpl 的 performDraw

![9](/screenshot/Android如何通过View进行渲染/9.png)

可以看出，在performDraw方法中，调用的ViewRootImpl的draw方法。在draw方法中进行UI绘制操作，Android提供了2种绘制方式：

1. 表示APP开启了硬件加速功能，所以会启用硬件加速绘制
2. 表示使用软件绘制

> ViewRootImpl 中有一个非常重要的对象Surface，之所以说ViewRootImpl的一个核心功能就是负责UI渲染，原因就在于在ViewRootImpl中会将我们在draw方法中绘制的UI元素，绑定到这个Surface上。如果说Canvas是画板，那么Surface就是画板上的画纸，Surface中的内容最终会被传递给底层的SurfaceFlinger，最终将Surface中的内容进行合成并显示在屏幕上

### 6.软绘制 drawSoftware

![10](/screenshot/Android如何通过View进行渲染/10.png)

1. 调用DecorView 的draw方法将UI元素绘制到画布Canvas对象中，具体可以绘制的内容在自定义View时总结过
2. 请求将Canvas中的内容显示到屏幕上，实际上就是将Canvas中的内容提交给SurfaceFlinger进行合成处理

默认情况下，软件绘制没有采用GPU渲染的方式，drawSoftware工作完全由CPU来完成

DecorView并没复写draw方法，因此实际是调用的顶层View的draw方法，如下：

![11](/screenshot/Android如何通过View进行渲染/11.png)

解释：

1. 绘制View的背景
2. 绘制View自身内容
3. 对draw事件进行分发，在View中是空实现，实际调用的是ViewGroup中的实现，并递归调用子View的draw事件

## 二、启用硬件加速

### 1.是否启用硬件加速

可以在ViewRootImpl的draw方法中，通过如下方法判断是否启用硬件加速：

![12](/screenshot/Android如何通过View进行渲染/12.png)

可以在清单文件中，指定Application或某一个Activity支持硬件加速

![13](/screenshot/Android如何通过View进行渲染/13.png)

此外还可以进行粒度更小的硬件加速设置，比如设置某个View支持硬件加速：

![14](/screenshot/Android如何通过View进行渲染/14.png)

之所有有这么多区分，是因为并不是所有的2D绘制操作都支持硬件加速，当在自定义View中使用如下API，可能导致程序工作不正常：

Canvas：

1. clipPath()
2. clipRegion()
3. drawPicture()
4. drawPosText()
5. drawTextOnPath()
6. drawVertices()

Paint：

1. setLinearText()
2. setMaskFilter()
3. setRasterizer()

### 2.硬件加速优势

















