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

































