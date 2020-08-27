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

它用来缓存移除屏幕之外的ViewHolder，默认情况下缓存个数是2，不过可以通过setViewCacheSize方法来改变缓存的容量大小。如果mCachedViews的容量已满，则会根据FIFO的规则将旧ViewHolder抛弃，然后添加新的ViewHolder，如下：

![12](/screenshot/RecycleView学习/12.gif)

通常情况下刚被移出屏幕的ViewHolder有可能接下来马上就会使用到，所以ViewHolder不会立即将其设置为无效ViewHolder，而是会将它们保存到cache中，但又不能将所有移除屏幕的ViewHolder都视为有效ViewHolder，所以它的默认容量只有2个

#### c.第三级缓存ViewCacheExtension

这是ViewHolder预留给开发人员的一个抽象类，在这个类中只有一个抽象方法，如下：

![13](/screenshot/RecycleView学习/13.png)

开发人员可以通过继承ViewCacheExtension，并复写抽象方法getViewForPositionAndType来实现自己的缓存机制。只是一般情况下我们不会自己实现也不建议自己去添加缓存逻辑，因为这个类的使用门槛较高。

#### d.第四级缓存RecycledViewPool

RecycledViewPool同样是用来缓存屏幕外的ViewHolder，当mCachedViews中的个数已满（默认为2），则从mCachedViews中淘汰出来的ViewHolder会先缓存到RecycledViewPool中。ViewHolder在被缓存到RecycledViewPool时，会将内部的数据清理，因此从RecycledViewPool中取出来的ViewHolder需要重新调用onBindViewHolder绑定数据。这就同最早的ListView中的使用ViewHolder复用convertView的道理是一致的，因此ViewHolder也算是将ListView优点完美继承了。

RecycledViewPool还有一个重要功能，多个RV之间可以共享一个RecycledViewPool，这对于多tab界面的优化效果会很显著。

**主要注意** ：RecycledViewPool是根据type来获取ViewHolder，每个type默认最大缓存5个。

因此多个RV共享RecycledViewPool时，必须确保共享的RV使用的Adapter是同一个，或view type是不会冲突的。

## 三、RV是如何从缓存中获取ViewHolder的

在上面介绍onLayout阶段时，有提到layoutChunk方法中通过调用layoutState.next方法拿到某个子ItemView，然后添加到RV中。

看下layoutState.next代码：

![14](/screenshot/RecycleView学习/14.png)

继续跟下去：

![15](/screenshot/RecycleView学习/15.png)

可以看出最终调用tryGetViewHolderForPositionByDeadline方法来查找相应位置上的ViewHolder，在这个方法中会从上面的4级缓存中依次查找：

![16](/screenshot/RecycleView学习/16.png)

如图红框处所示，如果在各级缓存中都没有找到相应的ViewHolder，则会使用Adapter中的createViewHolder方法创建一个新的ViewHolder

## 四、何时将ViewHolder存入缓存

看下ViewHolder被存入各级缓存的场景

### 1.第一次layout

当调用setLayoutManager和setAdapter之后，RV会经历第一次layout并被显示到屏幕上，如下：

![17](/screenshot/RecycleView学习/17.png)

此时并不会有任何ViewHolder的缓存，所有的ViewHolder都是通过createViewHolder创建的

### 2.刷新列表

如果通过手势下拉刷新，获取到新的数据data之后，我们会调用notifyXXX方法通知RV数据发生改变，这回RV会先将屏幕内的所有ViewHolder保存在Scrap中，如下：

![18](/screenshot/RecycleView学习/18.png)

当缓存执行完之后，后续通过Recycler就可以从缓存中获取相应position的ViewHolder（姑且称为旧ViewHolder），然后将刷新后的数据设置到这些ViewHolder上，如下：

![19](/screenshot/RecycleView学习/19.png)

最后再将新的ViewHolder绘制到RV上：

![20](/screenshot/RecycleView学习/20.png)

# 总结

本次我们深入学习了RecycleView源码中的2块核心实现：

- RecycleView是如何经过测量，布局，最终绘制到屏幕上，其中大部分工作是通过委托给LayoutManager来实现的
- RecycleView的缓存复用机制，主要是通过内部类Recycler来实现



























