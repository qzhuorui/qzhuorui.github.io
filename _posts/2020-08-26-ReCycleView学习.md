---
layout: post
title: 'RecycleView学习'
date: 2020-08-26
author: qzhuorui
color: rgb(154,133,255)
tags: Android
---



> 本次学习总结RecycleView，了解为啥RecycleView可以完美替代ListVIew

# RecycleView学习

RecycleView是作为ListView和GridView的加强版出现的，目的是在有限的屏幕之上展示大量的内容，因此RecycleView 的复用机制的实现是它的一个核心部分

RecycleView 常规使用方式如下：

![1](/screenshot/RecycleView学习/1.png)

解释说明：

- setLayoutManager：必选项！，设置RecycleView的布局管理器，决定RecycleView的显示风格。常用的有线性布局管理器（LinearLayoutManager），网格布局管理器（GridLayoutManager），瀑布流布局管理器（StaggeredGridLayoutManager）
- setAdapter：必选项！，设置RecycleView的数据适配器。当数据发生改变时，以通知者的身份，通知RecycleView数据改变进行列表刷新操作
- addItemDecoration：非必选项，设置RecycleView中Item的装饰器，经常用来设置Item的分隔线
- setItemAnimator：非必选项，设置RecycleView中Item的动画

这次我们主要学习RecycleView：

1. 如何一步步将每一个ItemView显示到屏幕上，
2. 再分析在显示和滑动过程中，是如何通过缓存复用来提升整体性能的。

RecycleView 本质上也是一个自定义控件，因此可以沿着分析onMeasure -> onLayout -> onDraw这3个方法的路线来深入研究

## 一、绘制流程分析

### 1.onMeasure

RecycleView的onMeasure方法如下：

![2](/screenshot/RecycleView学习/2.png)

- ①中，表示在XML布局文件中，RecycleView的宽高被设置为match_parent或具体值，那么直接将skipMeasure置为true，并调用mLayout（传入的LayoutManager）的onMeasure方法测量自身的宽高即可
- ②中，表示在XML布局文件中，RecycleView的宽高设置为wrap_content，则会执行下面的dispatchLayoutStep2()，其实就是测量RecycleView的子View的大小，最终确定RecycleView的实际宽高

上图中还有一个dispatchLayoutStep1()方法，它不是这次学习的目标，但是它和RecycleView的动画息息相关。后面需要了解学习！！

### 2.onLayout

RecycleView的onLayout方法如下：

![3](/screenshot/RecycleView学习/3.png)

很简单，只是调用了一层dispatchLayout()方法，此方法具体如下：

![4](/screenshot/RecycleView学习/4.png)

如果在onMeasure阶段没有执行dispatchLayoutStep2()方法去测量子View，则会在onLayout阶段重新执行

dispatchLayoutStep2()源码如下：

![5](/screenshot/RecycleView学习/5.png)

可以看出，核心逻辑是调用了mLayout的onLayoutChildren方法。这个方法是LayoutManager中的一个空方法，主要作用是测量RecycleView的子View大小，并确定它们所在的位置。LinearLayoutManager，GridLayoutManager，和StaggeredLayoutManager都分别复写这个方法，并实现了不同方式的布局

以LinearLayoutManager为例，展开分析，实现如下：

![6](/screenshot/RecycleView学习/6.png)

解释声明：

1. 在onLayoutChildren中调用fill方法，完成子View的测量布局工作
2. 在fill中通过while循环判断是否还有剩余足够空间来绘制一个完整的子View
3. layoutChunk方法中是子View测量布局的真实实现，每次执行完成后需要重新计算remainingSpace

layoutChunk是一个非常核心的方法，这个方法执行一次就填充一个ItemView到RecycleView，部分代码如下：

![7](/screenshot/RecycleView学习/7.png)

解释：

- 图①处从缓存（Recycler）中取出子ItemView，然后调用addView或addDisappearingView将子ItemView添加到RecycleView中
- 图②处测量被添加的RecycleView中的子ItemView的宽高
- 图③处根据所设置的Decoration，Margins等所有选项确定子ItemView的显示位置

### 3.onDraw

测量和布局都完成之后，就剩下最后的绘制操作了，如下：

![8](/screenshot/RecycleView学习/8.png)

这个方法很简单，如果有添加ItemDecoration，则循环调用所有的Decoration的onDraw方法，将其显示。至于所有的子ItemView则是通过Android渲染机制递归的调用子ItemVIew的draw方法显示到屏幕上。

### 小结：

RecycleView会将测量onMeasure和布局onLayout的工作委托给LayoutManager来执行，不同的LayouManager会有不同风格的布局显示，这是一种策略模式，如图表示：

![9](/screenshot/RecycleView学习/9.png)

## 二、缓存复用原理Recycler

缓存复用是RecycleView中另一个非常重要的机制，这套机制主要实现RecycleView的缓存以及复用

核心代码是在Recycler中完成的，它是RecycleView中的一个内部类，主要用来缓存屏幕内ViewHolder以及部分屏幕外ViewHolder，部分代码如下：

![10](/screenshot/RecycleView学习/10.png)

Recycler的缓存机制就是通过上图中的这些数据容器来实现的，实际上Recycler的缓存也是分级处理的，根据访问优先级从上到下可以分为4级，如下：

![11](/screenshot/RecycleView学习/11.png)

### 1.各级缓存功能

RecycleView之所以分成这么多块，是为了在功能上进行一些区分，并分别对应不同的使用场景

#### a.第一级缓存mAttachedScrap&mChangedScrap

是两个名为Scrap的ArrayList，这两者主要用来缓存屏幕内的ViewHolder。为什么屏幕内的ViewHolder需要缓存？

引出一个场景：通过下拉刷新列表中的内容，当刷新被触发时，只需要在原有的ViewHolder基础上进行重新绑定新的数据data即可，而这些旧的ViewHolder就是被保存在mAttachedScrap和mChangedScrap中。实际上当我们调用RecycleView的notifyXXX方法时，就会向这两个列表进行填充，将旧ViewHolder缓存起来。

#### b.第二级缓存mCachedViews

























