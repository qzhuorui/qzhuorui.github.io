---
layout: post
title: '理解ClassLoader加载机制'
date: 2020-08-20
author: qzhuorui
color: rgb(255,210,32)
tags: JVM/ART
---



> 总结学习下类加载器ClassLoader



# 理解ClassLoader加载机制

首先需要知道一个完整的Java程序是由多个.class文件组成，在程序运行过程中，需要**将这些.class文件加载到JVM中**才可以使用。

而负责加载这些.class文件的就是这次学习的类加载器（Class Loader）

## Java中的类何时被加载器加载

在Java程序启动的时候，并不会一次性加载程序中所有的.class文件，而是在程序的运行过程中，动态地加载相应的类到内存中。

通常情况下，Java程序中的.class文件会在以下2种情况下被ClassLoader主动加载到内存中：

1. 调用类构造器
2. 调用类中的静态（static）变量或方法

## Java中ClassLoader

JVM中自带3个类加载器：

1. 启动类加载器BootstrapClassLoader
2. 扩展类加载器ExtClassLoader(JDK1.9之后改名为PlatformClassLoader)
3. 系统加载器APPClassLoader

以上3者在JVM中有各自分工，但是又互相依赖

### 1.APPClassLoader系统类加载器

部分源码如下：

![1](/screenshot/理解ClassLoader加载机制/1.png)

可以看出，APPClassLoader主要加载系统属性“java.class.path"配置下类文件，也就是环境变量CLASS_PATH配置的路径。因此APPClassLoader是面向用户的类加载器，我们自己写的代码以及使用第三方jar包通常都是由它来加载

### 2.ExtClassLoader扩展类加载器

部分源码如下：

![2](/screenshot/理解ClassLoader加载机制/2.png)

可以看出，ExtClassLoader加载系统属性"java.ext.dirs"配置下类文件，可以打印出这个属性看看具体有哪些文件：

`System.out.println(System.getProperty("java.ext.dirs"));`

`/Library/Java/JavaVirtualMachines/jdk_1.8.0/Contents/Home/jre/lib/ext`

### 3.BootstrapClassLoader启动类加载器

BootstrapClassLoader同上面的两种ClassLoader不太一样。首先，它并不是使用Java代码实现的，而是由C/C++编写，它本身属于虚拟机的一部分。因此我们无法在Java代码中直接获取它的引用。如果尝试在Java层获取BootstrapClassLoader的引用，系统会返回null。

BootstrapClassLoader加载系统属性“sun.boot.class.path”配置下类文件，可以打印出这个属性来查看具体有哪些文件，结果是全是JRE目录下的jar包或者.class文件。

## 双亲委派模式（Parents Delegation Model）

既然JVM中已经有了这3种ClassLoader，那么JVM又是如何知道该使用哪一个类加载器去加载相应的类呢？答案就是：**双亲委派模式**

### 双亲委派模式

双亲委派模式就是，当类加载器收到加载类或资源的请求时，通常都是先委托给父类加载器加载，也就是说，只有当父类加载器找不到指定类或资源时，自身才会执行实际的类加载过程。

其具体实现代码是在ClassLoader.java中的loadClass方法中，如下：

![3](/screenshot/理解ClassLoader加载机制/3.png)

解释说明：

1. 判断该Class是否已加载，如果已加载，则直接将该Class返回
2. 如果该Class没有被加载过，则判断parent是否为空，如果不为空则将加载的任务委托给parent
3. 如果parent==null，则直接调用BootstrapClassLoader加载该类
4. 如果parent或者BootstrapClassLoader都没有加载成功，则调用当前ClassLoader的findClass方法继续尝试加载

那这个parent是啥呢，看下ClassLoader的构造器，如下：

![4](/screenshot/理解ClassLoader加载机制/4.png)

可以看出，在每一个ClassLoader中都有一个ClassLoader类型的parent引用，并且在构造器中传入值。如果我们继续查看源码，可以看到AppClassLoader传入的parent就是ExtClassLoader，而ExtClassLoader并没有传入任何parent，也就是null

举例说明，执行以下代码：

`Test test = new Test();`

默认情况下，JVM首先使用AppClassLoader去加载Test类。

1. AppClassLoader将加载的任务委派给它的父类加载器（parent）-ExtClassLoader
2. ExtClassLoader的parent为null，所以直接将加载任务委派给BootstrapClassLoader
3. BootstrapClassLoader在jdk.lib目录下无法找到Test类，因此返回的Class为null
4. 因为parent和BootstrapClassLoader都没有成功加载Test类，所以AppClassLoader会调用自身的findClass方法来加载Test

最终Test类就是被AppClassLoader加载到内存中，验证代码如下：

![5](/screenshot/理解ClassLoader加载机制/5.png)

![6](/screenshot/理解ClassLoader加载机制/6.png)

可以看出，Test的ClassLoader为AppClassLoader类型，而AppClassLoader的parent为ExtClassLoader类型。ExtClassLoader的parent为Null

> ”双亲委派“机制只是Java推荐的机制，并不是强制的机制。我们可以继承java.lang.ClassLoader类，实现自己的类加载器。如果想保持双亲委派模型，就应该重写findClass(name)方法；想破坏双亲委派模型，可以重写loadClass(name)方法

## 自定义ClassLoader

JVM中预置的3种ClassLoader只能加载特定目录下的.class文件，如果我们想加载其他特殊位置下的jar包或类时（比如，要加载网络或者磁盘上一个.class文件），默认的ClassLoader就不能满足需求，所以需要自定义ClassLoader来加载特定目录下的.class文件

### 1.自定义ClassLoader步骤

1. 重写一个类继承抽象类ClassLoader
2. 重写findClass方法
3. 在findClass方法中，调用defineClass方法将字节码转换成Class对象，并返回

用伪代码来描述下：

![7](/screenshot/理解ClassLoader加载机制/7.png)

### 2.自定义ClassLoader实践

首先在电脑上创建一个测试类Secret.java

```java
public class Secret{
    public void printSecret(){
        System.out.println("我是女生");
    }
}
```

接下来创建DiskClassLoader继承ClassLoader，重写findClass方法，并在其中调用defineClass

![8](/screenshot/理解ClassLoader加载机制/8.png)

最后写一个测试自定义DiskClassLoader的测试类，来验证下

![9](/screenshot/理解ClassLoader加载机制/9.png)

解释说明：

1. 代表需要动态加载的class的路径
2. 代表需要动态加载的类名
3. 代表需要动态调用的方法名称

> 注意：上述动态加载.class文件的思路，经常被用作热修复和插件化开发的框架中，包括QQ空间热修复方案，微信Tink等原理都是由此而来。客户端只要从服务端下载一个加密的.class文件，然后在本地通过事先定义好的加密方式进行解密，最后再使用自定义ClassLoader动态加载解密后的.class文件，并动态调用相应的方法

## Android中的ClassLoader

本质上，Android和传统JVM是一样的，也需要通过ClassLoader将目标类加载到内存，类加载器之间也符合双亲委派模型。但是在Android中，ClassLoader的加载细节**有略微差别** 。

在Android虚拟机里是无法直接运行.class文件的，Android会将所有的.class文件转换成一个.dex文件，并且Android将加载.dex文件的实现封装在BaseDexClassLoader中， **而我们一般只使用它的两个子类：PathClassLoader和DexClassLoader**

### 1.PathClassLoader

PathClassLoader用来加载系统apk和被安装到手机中的apk内的dex文件。它的2个构造函数如下：

![10](/screenshot/理解ClassLoader加载机制/10.png)

参数说明：

1. dexPath：dex文件路径，或者包含dex文件的jar包路径
2. librarySearchPath：C/C++ native库的路径

PathClassLoader里面除了这2个构造方法外就没有其他代码了，具体的实现都是在BaseDexClassLoader里面，其dexPath比较受限制，一般是已经安装应用的apk文件路径

当一个app被安装到手机后，apk里面的class.dex中的class均是通过PathClassLoader来加载的，可以验证下

![11](/screenshot/理解ClassLoader加载机制/11.png)

打印结果为：`dalvik.system.PathClassLoader`

### 2.DexClassLoader

首先看下官方的描述：

> A class loader that loads classes from .jar and .apk filescontaining a classes.dex entry
>
> This can be used to execute code notinstalled as part of an application

很明显，对比PathClassLoader只能加载已经安装应用的dex或apk文件，DexClassLoader则没有此限制，可以从SD卡上加载包含class.dex的.jar和.apk，这也是插件化和热修复的基础，在不需要安装应用的情况下，完成需要使用的dex的加载。

DexClassLoader的源码里面只有一个构造方法

![12](/screenshot/理解ClassLoader加载机制/12.png)

参数说明：

1. dexPath：包含class.dex的apk，jar文件路径，多个路径用文件分隔符（默认是":"）分隔
2. optimizedDirectory：用来缓存优化的dex文件的路径，即从apk或jar文件中提取出来的dex文件。该路径不可为空，且应该是应用私有的，有读写权限的路径

## 使用DexClassLoader实现热修复

### 1.创建Android项目DexClassLoaderHotFix

项目结构如下：

![13](/screenshot/理解ClassLoader加载机制/13.png)

ISay.java是一个接口，内部只定义了一个方法saySomething

```java
public interface ISay{
    String saySomething();
}
```

SayException.java实现了ISay接口，但是在saySomething方法中，打印"something wrong here"来模拟一个线上bug

![14](/screenshot/理解ClassLoader加载机制/14.png)

最后在MainActivity.java中，当点击Button时，将saySomething返回的内容通过Toast显示

![15](/screenshot/理解ClassLoader加载机制/15.png)

### 2.创建HotFix patch包

创建Java项目，并分别创建两个文件：ISay.java和SayHotFix.java

![16](/screenshot/理解ClassLoader加载机制/16.png)

![17](/screenshot/理解ClassLoader加载机制/17.png)

ISay接口的包名和类名，必须和Android项目中保持一致！！！

SayHotFix实现ISay接口，并在saySomething中返回了新结果，用来模拟bug修复后的结果。

将ISay.java和SayHotFix.java打包成say_something.jar，然后通过dx工具将生成的say_something.jar包中的class文件优化为dex文件

> dx --dex --output=say_something_hotfix.jar say_something.jar

上述say_something_hotfix.jar就是我们最终需要用作horfix的jar包

**将HotFix patch包拷贝到SD卡主目录，并使用DexClassLoader加载SD卡中的ISay接口**

首先将HotFix patch保存到本地目录下。一般在真实项目中，我们可以通过想后台发送请求的方式，将最新的HotFix patch下载到本地中。这里为了演示就使用adb push将say_something_hotfix.jar包push到SD卡主目录下。

接下来修改MainActivity中的逻辑，使用DexClassLoader加载HotFix patch中的SayHotFix类，如下：

![18](/screenshot/理解ClassLoader加载机制/18.png)

> 注意：这里需要SD卡访问权限的

最后运行就会发现Toast的内容改变了

## 总结

- ClassLoader就是用来加载class文件的，不管是jar中还是dex中的class
- Java中的ClassLoader 通过双亲委派来加载各自指定路径下的class文件
- 可以自定义ClassLoader ，一般覆盖findClass()方法，不建议重写loadClass方法
- Android中常用2中ClassLoader ，分别为：PathClassLoader 和DexClassLoader 













