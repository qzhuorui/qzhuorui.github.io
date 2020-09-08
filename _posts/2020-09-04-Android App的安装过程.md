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

![9](/screenshot/Android App的安装过程/9.png)

MCS_BOUND的Message接收者还是PackageHandler，具体如下：

![10](/screenshot/Android App的安装过程/10.png)

mPendingInstalls是一个等待队列，里面保存所有需要安装的apk解析出来的PackageParams参数，从mPendingInstalls中取出第一个需要安装的HandlerParams对象，并调用其startCopy方法，在startCopy方法中会继续调用一个抽象方法handleStartCopy处理安装请求。通过之前的分析，我们知道HandlerParams实际类型是InstallParams类型，因此最终调用的是InstallParams的handlerStartCopy方法

#### 2.InstallParams 的handlerStartCopy方法

这个方法是整个安装包拷贝的核心方法，如下：

![11](/screenshot/Android App的安装过程/11.png)

解释：

1. 设置安装标志位，决定安装在手机内部存储空间还是sdcard中
2. 判断apk安装位置

如果安装位置合法，则执行③处逻辑，创建一个InstallArgs，实际上是其子类FileInstallArgs类型，然后调用其copyApk方法进行安装包的拷贝操作。

#### 3.FileInstallArgs 的copyApk方法

![12](/screenshot/Android App的安装过程/12.png)

可以看出在copyApk方法中调用了doCopyApk方法，doCopyApk方法中主要做了3件事：

1. 创建存储安装包的目标路径，实际上是/data/app/ 应用包名目录
2. 调用服务的copyPackage方法将安装包apk拷贝到目标路径中
3. 将apk中的动态库.so文件也拷贝到目标路径中

上图中的IMediaContainerService实际上就是在开始阶段进行连接操作的DefaultContainerService对象，其内部copyPackage方法本质上就是执行IO流操作，具体如下：

![13](/screenshot/Android App的安装过程/13.png)

最终安装包在data/app目录下以base.apk的方式保存， **至此安装包拷贝工作已完成** 。

### 2.装载代码

代码拷贝结束之后，就开始进入真正的安装步骤

代码回到上述的HandlerParams中的startCopy方法：

![14](/screenshot/Android App的安装过程/14.png)

可以看出当安装包拷贝操作结束之后，继续调用handleReturnCode方法来处理返回结果，最终调用processPendingInstall方法处理安装过程，代码具体如下：

![15](/screenshot/Android App的安装过程/15.png)

解释：

1. 执行预安装操作，主要是检查安装包的状态，确保安装环境正常，如果安装环境有问题会清理拷贝文件
2. 真正的安装阶段，installPackageTraceLI方法中添加跟踪Trace，然后调用installPackageLI方法进行安装
3. 处理安装完成之后的操作

installPackageLI是apk安装阶段的核心代码，核心代码如下：

![16](/screenshot/Android App的安装过程/16.png)

1. 调用PackageParser的parsePackage方法解析apk文件，主要是解析AndroidManifest.xml文件，将结果记录在PackageParser.Package中。我们在清单文件中声明的Activity，Service等组件就是在这一步中被记录到系统Framework中，后续才可以通过startActivity或startService启动相应的活动或服务
2. 对apk中的签名信息进行验证操作。collectCertificates做签名验证，collectManifestDigest主要是做包的项目清单摘要的收集，主要适合用来比较两个包是否一样。如果已经安装了一个debug版本的apk，再次使用一个release版本的apk进行覆盖安装时，会在这一步验证失败，最终导致安装失败
3. 执行dex优化，实际为dex2oat操作，用来将apk中的dex文件转化为oat文件
4. 调用installNewPackageLI方法执行新apk的安装操作

installNewPackageLI方法是负责完成最后的apk安装过程，如下：

![17](/screenshot/Android App的安装过程/17.png)

解释：

1. scanPackageLI继续扫描解析apk安装包文件，保存apk相关信息到PMS中，并创建apk的data目录，路径为/data/data/应用包名
2. updateSettingLI如果安装成功，更新系统设置中的应用信息，比如应用的权限信息
3. deletePackageLI如果安装失败，则将安装包以及各种缓存文件删除

> 至此整个apk的安装过程结束，实际上安装成功之后，还会发送一个App安装成功的广播ACTION_PACKAGE_ADD。手机桌面应用注册这个广播当收到之后，就将app的启动icon显示在桌面

# 总结

主要总结学习了一个Android项目从编译成apk文件，然后被安装到手机设备上的简要过程。其中编译分为2部分：资源+源码。并且生成apk之后还要进行签名，对齐等操作。apk安装也分为2步：安装包拷贝和代码装载。



















