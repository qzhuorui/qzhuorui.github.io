---
layout: post
title: 'Android App的安装过程'
date: 2020-09-04
author: qzhuorui
color: rgb(255,210,32)
tags: Android
---



> 通过分析apk的安装过程学习下PackagemanagerService(PMS)

# Android App的安装过程

在分析安装过程之前，需要先了解下Android项目是如何 编译 -> 打包，生成最终的 .apk格式的安装包。引用Google的一张图：

![1](/screenshot/Android App的安装过程/1.png)

一个完整的Android项目可能包含多个module，而从宏观上看每一个module中的内容可以分为2部分：**Resource资源文件，Java或Kotlin源代码** 。因此**整个项目的编译打包过程也是针对这2部分来完成** 。

## 一、编译阶段

### 1.Resource资源文件

资源文件包括项目中res目录下的各种XML文件，动画，drawable图片，音视频等。AAPT工具负责编译项目中的这些资源文件，所有资源文件会被编译处理， **XML文件（drawable图片除外）会被编译成二进制文件** ，所以解压apk之后无法直接打开XML文件。但是**assets和raw目录下的资源并不会被编译，会被原封不动的打包到apk压缩包中** 。

资源文件编译之后的产物包括2部分：resource.arsc文件和一个R.java。前者保存的时一个资源索引表，后者定义了各个资源ID常量。这两者结合就可以在代码中找到对应的资源引用。

如下R.java文件：

![2](/screenshot/Android App的安装过程/2.png)

可以看出，R.java中的资源ID是一个4字节的无符号整数，用16进制表示。最高的1字节表示Package ID，次高的1字节表示Type ID，最低的2字节表示Entry ID

resource.arsc相当于一个资源索引表，可以理解为一个map映射表。其中map的key就是R.java中的资源ID，而value就是其对应的资源所在的路径。实际上resource.arsc里面还有别的信息，这里就不展开了

### 2.源码部分

项目中的源码首先会通过javac编译为.class字节码文件，然后这些.class文件连同依赖的三方库中的.class文件一同被dx工具优化为.dex文件。如果有分包，那么也可能会产生多个.dex文件。

> 实际上源码文件也包括AIDL接口文件编译之后生成的.java文件，Android项目中如果包含.aidl接口文件，这些.aidl文件会被编译成.java文件

## 二、打包阶段

最后使用工具APK Builder将编译之后的resource和.dex文件一起打包到apk中，实际上被打包到apk中的还有一些其他资源，比如AndroidManifest.xml清单文件和第三方库中使用的动态.so文件

apk创建好之后，还不能直接使用。需要使用工具jarsigner对其进行签名，因为Android系统不会安装没有进行签名的程序。签名之后会生成META_INF文件夹，此文件夹中保存着跟签名相关的各个文件。

- CERT.SF：生成每个文件相对的密钥
- MANIFEST.MF：数字签名信息
- xxx.SF：只是JAR文件的签名文件
- xxx.DSA：对输出文件的签名和密钥

PMS在安装过程中会检查apk中的签名证书的合法性，这个后面说。

常量来说，签名之后的apk应该是可以正常安装使用了，但是**实际打包过程还会多一步apk优化操作** 。就是使用工具zipalign对apk中的未压缩资源（图片，视频等）进行**对齐操作，让资源按照4字节的边界进行对其** 。这种思想同Java对象内存分布中的对齐空间非常类似，主要是**为了加快资源的访问速度** 。如果每个资源的开始位置都是上一个资源之后的4n字节，那么访问下一个资源就不用遍历，直接跳到4n字节处判断是不是一个新资源即可。

至此一个完整的apk安装包就创建成功，一个完整的apk解压缩之后的内容如下：

![3](/screenshot/Android App的安装过程/3.png)

整个编译打包流程可用下图描述：

![4](/screenshot/Android App的安装过程/4.png)

接下来看下PMS是如何将其安装到手机设备中的。

## 三、PMS安装过程概览

当我们点击某一个App安装包进行安装时，首先会弹出一个系统界面指示我们进行安装操作。这个界面是Android Framework中预置的一个Activity—PackageInstallerActivity.java。 **当点击安装后，PackageInstallerActivity最终会将所安装的apk信息通过PackageInstallerSession传给PMS** ，具体方法在commitLocked方法中，如下：

![5](/screenshot/Android App的安装过程/5.png)

图中的mPm就是系统服务PackageManagerService。installStage方法就是正式开始apk的安装过程。

整个apk的安装过程可以分为2大步：

1. 拷贝安装包
2. 装载代码

### 1.拷贝安装包

从installStage方法开始，代码如下：

![6](/screenshot/Android App的安装过程/6.png)

- ①处创建了类型为INIT_COPY的Message
- ②处创建InstallParams，并传入安装包的相关数据

Message发送出去之后，由PMS的内部类PackageHandler接收并处理，如下：

![7](/screenshot/Android App的安装过程/7.png)

- ①处从Message中取出HandlerParams对象，实际类型是InstallParams类型
- ②处调用connectToService方法连接安装apk的Service服务

#### 1.PackageHandler的connectToService

![8](/screenshot/Android App的安装过程/8.png)

通过隐式Intent绑定Service，实际绑定的Service类型是DefaultContainerService类型。当绑定Service成功之后，会在onServiceConnection方法中发送一个绑定操作的Message，如下：































