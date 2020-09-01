---
layout: post
title: 'Android Bitmap学习'
date: 2020-08-28
author: qzhuorui
color: rgb(255,210,32)
tags: Android
---



>主要学习了解下Bitmap

# Android Bitmap学习

> Android App中都会用到Bitmap，它是程序中内存消耗的大户，当Bitmap使用内存超过可用空间，就会报OOM。因此我们需要了解如何正确使用

## 一、Bitmap占用内存分析

Bitmap用来描述一张图的长，宽，颜色等信息。通常下我们可以使用BitmapFactory来将某一路径下的图片解析为Bitmap对象

当一张图加载到内存之后，具体需要占用多大内存呢？

### 1.getAllocationByteCount探索

我们可以通过Bitmap.getAllocationByteCount()方法获取Bitmap占的字节大小

![1](/screenshot/AndroidBitmap学习/1.png)

打印结果为： `bitmap size is 1440000` 

默认情况下BitmapFactory使用Bitmap.Config.ARGB_8888的存储方式来加载图片内容，而在这种存储模式下，每一个像素需要占用4个字节。因此内存大小可用如下公式来算：`宽 * 高 * 4 = 600 * 600 * 4 = 1440000` 

### 2.屏幕自适应

如果我们在保证代码不修改的前提下，将图片移动到（注意是移动而不是拷贝！）res/drawable-hdpi目录下，重新运行代码，则打印为：

`bitmap size is 2560000` 

可以看出只是移动了图片的位置，Bitmap所占用的空间上涨了77%，这为啥？

实际上BitmapFactory在解析图片的过程中，会根据 **当前屏幕密度** 和 **图片所在的drawable目录** 来做一个对比，根据这对比值进行缩放操作。公式如下：

1. 缩放比例scale = 当前设备屏幕密度 / 图片所在drawable目录对应屏幕密度
2. Bitmap实际大小 = 宽 * scale * 高 * scale * Config对应存储像素数

在Android中，各个drawable对应的屏幕密度如下：

![2](/screenshot/AndroidBitmap学习/2.png)

调试设备为Nexus 4，屏幕密度为320。如果将图放在hdpi下，最终计算公式为：

`600 * (320 / 240) * 600 * (320 / 240) * 4 = 2560000`

### 3.assets中的图片大小

Android中的图片不仅可以保存在drawable目录中，还可以保存在assets目录下，然后通过AssetManager获取图片的输入流。

那这种方式加载生成的Bitmap多大呢？同上图片，这次放在assets中

![3](/screenshot/AndroidBitmap学习/3.png)

打印结果：`bitmap size is 1440000`

可以看出，加载assets目录中的图片，系统并不会对其进行缩放操作

## 二、Bitmap加载优化

从上面例子看出，一张65Kb大小的图片被加载到内存后，竟然占了2560000个字节，也就是2.5M左右。因此适当时候，我们需要对加载的图片进行优化！

### 1.修改图片加载的Config

修改占用空间少的存储方式，可以快速有效降低图片占用内存。

比如通过BitmapFactory.Options的inPreferredConfig选项，将存储方式设置为Bitmap.Config.RGB_565。这种存储方式一个像素占用2个字节，所以最终占用内存直接减半！

![4](/screenshot/AndroidBitmap学习/4.png)

另外Options中还有一个inSampleSize参数，可以实现Bitmap采样压缩，这个参数的含义是宽高维度上每隔inSampleSize个像素进行一次采集

![5](/screenshot/AndroidBitmap学习/5.png)

因为宽高都会进行采样，所以最终图片会被缩略4倍

## 三、Bitmap复用

### 1.场景描述

如果在Android某个页面创建很多个Bitmap，比如有两张图片A和B，通过点击某一个按钮需要在ImageView上切换显示这两张图片，实现效果如下：

![6](/screenshot/AndroidBitmap学习/6.gif)

代码如下：

![7](/screenshot/AndroidBitmap学习/7.png)

但是在每次调用switchImage切换图片时，都需要通过BitmapFactory创建一个新的Bitmap对象。当方法执行完毕之后，这个Bitmap又会被GC回收，这就造成不断地创建和销毁较大的内存对象，从而导致频繁GC（或叫内存抖动）。像Android APP这种面向用户的产品，会影响到用户体验的。可以在AS Profiler中查看内存情况，多次切换内存图片后，显示的效果如下：

![8](/screenshot/AndroidBitmap学习/8.png)

### 2.使用Options.inBitmap优化

实际上经过第一次显示后，内存中已经存在了一个Bitmap对象。每次切换图片只是显示的内容不同，我们可以重复利用已经占用内存的Bitmap空间，具体做法就是使用Options.inBitmap参数。将getBitmap方法修改如下：

![9](/screenshot/AndroidBitmap学习/9.png)

解释：

- 图①处创建一个可以用来服用的Bitmap对象
- 图②处将options.inBitmap赋值为之前创建的reuseBitmap对象，从而避免重新分配内存

重新运行代码，并查看Profiler中内存情况，可以发现不管我们切换多少次图片，内存占用始终处于一个水平线状态

![10](/screenshot/AndroidBitmap学习/10.png)

**注意**：

在上述getBitmap方法中，复用inBitmap之前，需要调用canUseForInBitmap方法来判断reuseBitmap是否可以被复用。这是因为Bitmap的复用有一定的限制：

- 在Android4.4版本之前，只能重用相同大小的Bitmap内存区域
- 4.4之后可以重用任何Bitmap的内存区域，只要这块内存比将要分配内存的Bitmap大就行

canUseForInBitmap代码如下：

![11](/screenshot/AndroidBitmap学习/11.png)

在每次加载前，除了inBitmap参数外，我们还需要将Options.inMutable置为true，如果不置为true，BitmapFactory将不会重复利用Bitmap内存，并输出相应的warning日志

## 四、BitmapRegionDecoder图片分片显示

有时想加载的图片很大或很长，比如手机滚动截屏的图。

针对这种情况，在不压缩图片的前提下， **不建议一次性将整张图加载到内存** ，而是采用分片加载的方式来显示图片部分内容，然后根据手势操作，放大缩小或移动图片显示区域。

图片分片加载显示主要使用Android SDK中的BitmapRegionDecoder来实现。

### 1.BitmapRegionDecoder基本使用

首先需要使用BitmapRegionDecoder将图片加载到内存中，图片可以以绝对路径，文件描述符，输入流的方式传递给BitmapRegionDecoder，如下：

![12](/screenshot/AndroidBitmap学习/12.png)

运行后，显示效果如下：

![13](/screenshot/AndroidBitmap学习/13.png)

在此基础上，我们可以通过自定义View，添加touch事件来动态地设置Bitmap需要显示的区域Rect。

## 五、Bitmap缓存

当需要在界面上同时展示一大堆图片时，比如ListView，RecycleView等，由于用户不断的上下滑动，某个Bitmap可能会被短时间内加载并销毁多次。这种情况下**通过适当的缓存，可以有效地减缓GC频率保证图片加载效率，提高界面响应速度和流畅性** 。

最常用的缓存方式就是LruCache，基本使用如下：

![14](/screenshot/AndroidBitmap学习/14.png)

解释：

- 图①处指定LruCache的最大空间为20M，当超过20M时，LruCache会根据内部缓存策略将多余Bitmap移除
- 图②处指定了插入Bitmap时的大小，当我们向LruCache中插入数据时，LruCache并不知道每一个对象会占用多大内存，因此需要手动指定，并根据缓存数据的类型不同也会有不同的计算方式

# 总结

本次学习总结了以下几个问题：

1. 一张图片被加载成Bitmap后实际占用内存是多大
2. 通过Options.inBitmap可以实现Bitmap的复用，但有一定限制
3. 当界面需要展示多张图，尤其是列表，可以考虑使用Bitmap缓存
4. 如果要展示图过大，可以考虑分片加载策略



































