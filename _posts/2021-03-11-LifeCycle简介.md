---
layout: post
title: 'LifeCycle使用简介'
date: 2021-03-11
author: qzhuorui
color: rgb(98,170,255)
tags: Android
---



> 简单记录下LifeCycle


# LifeCycle

> 目的：把Activity生命周期的方法延伸给另外一个类。让另外一个类去处理生命周期方法，缩小Activity代码量。解耦，让组件不需要关注页面的生命周期。

Android主要提供了两个类**LifecycleOwner**（被观察者类）和**LifecycleObserver**（观察者）。通过**观察者模式**，实现对页面生命周期的回调与监听。

对于具有生命周期性质的页面空间，Android都已实现 **LifecycleOwner** 接口。想要监听生命周期时，我们只需要实现  **观察者**  就好。

使用：

LifecycleOwner接口中只有一个**getLifecycle(LifecycleObserver observer)**方法。 通过该方法实现”观察者模式“。

通过getLifecycle().addObserver()方法将观察者与被观察者绑定起来即可。

观察者中根据观察需求，在对应方法上使用注解接口监听。

```java
@OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
```

