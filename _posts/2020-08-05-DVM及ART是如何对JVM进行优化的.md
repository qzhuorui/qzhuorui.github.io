---
layout: post
title: 'DVM及ART是如何对JVM进行优化的？'
date: 2020-08-05
author: qzhuorui
color: rgb(255,210,32)
tags: JVM/ART
---



# DVM及ART是如何对JVM进行优化的？

## 一、什么是Dalvik

Dalvik是Google公司自己设计用于Android平台的Java虚拟机，Android工程师编写的Java或者Kotlin代码最终都是在这台虚拟机中被执行的。在**Android5.0之前叫作DVM，5.0之后改为ART** （Android RunTime）

在整个Android操作系统体系中，ART位于以下图中红框位置。![CgqCHl6qeUKAa86MAAY44MY5alU343](/assets/profile.jpg/CgqCHl6qeUKAa86MAAY44MY5alU343.png)

> 其实称DVM/ART为Android版的Java虚拟机，这种说法并不是很准确。虚拟机必须符合Java虚拟机规范，也就是要通过JCM（Java Compliance Kit）的测试并获得授权，但是DVM/ART并没有。

DVM大多数实现与传统的JVM相同，但是因为Android最初是被设计用于手机端的，对内存空间要求较高，并且起初Dalvik目标是只运行在ARM架构的CPU上。针对这几种情况， **Android DVM有了自己独有的优化措施** 。

## 二、Dex文件

传统Class文件是由一个Java源码文件生成的.Class文件，而Android是**把所有Class文件进行合并优化** ，然后**生成一个最终的class.dex文件** 。dex文件去除了class文件中的冗余信息（比如重复字符常量），并且结构更加紧凑，因此在dex解析阶段，可以减少I/O操作，提高了类的查找速度。

实际上， **dex文件在App安装过程中还会被进一步优化**为odex（optimized dex）。

这一优化也伴随这一些副作用，最经典的就是Android 65535问题。出现这个问题的根本原因是在DVM源码中的MemberldsSection.java类中，如下：

```java
@Override
protected void orderItems(){
    int idx = 0;
    if(items().size > DexFormat.MAX_MEMBER_IDX+1){
        throw new DexException(Main.TO_MANY_ID_ERROR_MESSAGE)；
    }
    for(Obejct i : items()){
        ((MemberIdItem) i).setIndex(idx);
        idx++;
    }
}
```

如果items个数超过DexFormat.MAX_MEMBER_IDX则会报错，**DexFormat.MAX_MEMBER_IDX的值为65535** ，items代表dex文件中的方法个数，属性个数，以及类的个数。也就是说理论上不止方法数，我们在Java中声明的变量，或创建的类个数，如果也超过65535个，同样会编译失败，**Android提供了MultiDex来解决这个问题** 。很多说65535错误原因是因为解析dex文件到数据结构DexFile时，使用了short来存储方法数，其实这个说法是错误的。

## 三、架构基于寄存器&基于栈堆结构

我们了解到JVM的指令集是基于栈结构来执行的；而Android却是基于寄存器的，不过这里不是直接操作硬件的寄存器，而是在内存中模拟一组寄存器。Android字节码和Java字节码完全不同，Android的字节码（smali）更多的是二地址指令和三地址指令

基于寄存器的指令会明显比基于栈的指令少，虽然增加了指令长度，但却缩减了指令的数量，执行也更为快速。

## 四、内存管理与回收

DVM与JVM另一个比较显著的不同就是内存结构的区别，主要体现在对“堆”内存的管理。

Dalvik虚拟机中的堆被划分为了2部分：Active Heap，Zygote Heap。

![CgqCHl6qe9WAY-x4AAHlcF3z4X8795](G:\GitRepository\qzhuorui.github.io\screenshot\CgqCHl6qe9WAY-x4AAHlcF3z4X8795.png)



### 为什么要分Zygote和Active两部分

Android系统的第一个Dalvik虚拟机是由Zygote进程创建的，而应用程序进程是由Zygote进程fork出来的。

Zygote进程是在系统启动时产生的，它会完成虚拟机的初始化，库的加载，预置类库的加载和初始化等操作，而在系统需要一个新的虚拟机实例时Zygote通过复制自身，最快速的提供一个进程；另外对于一些只读的系统库，所有虚拟机实例都和Zygote共享一块内存区域，大大节省了内存开销，如下图：

![CgqCHl6qe-aATBEFAAEFkKPCQb4077](G:\GitRepository\qzhuorui.github.io\screenshot\CgqCHl6qe-aATBEFAAEFkKPCQb4077.png)

当启动一个应用时，Android操作系统需要为应用程序创建新的进程，而这一步操作是通过一种写时拷贝技术（COW）直接复制Zygote进程而来。这意味着在开始的时候，程序进程和Zygote进程共享了同一个用来分配对象的堆。然而， **当Zygote进程或者应用程序进程对该堆进行写操作时，内核就会执行真正的拷贝操作，使得Zygote进程和应用进程分别拥有自己的一份拷贝** 。拷贝是一件费时费力的事情。 **因此，为了尽量避免拷贝，Dalvik虚拟机将自己的堆划为两部分** 。

事实上，Dalvik虚拟机的堆最初只有一个，也就是Zygote进程在启动过程中创建Dalvik虚拟机时，只有一个堆。但是当Zygote进程在fork第一个应用进程之前，会将已经使用的那部分堆内存划分为一部分，把还没有使用的堆内存划分为另外一部分。前者就是Zygote堆，后者就成为Active堆。以后无论是Zygote进程，还是应用程序进程，当他们需要分配对象时，都在Active堆上进行。这样就**可以使得Zygote堆尽可能少的被执行写操作** ，因此就可以**减少执行写时拷贝的操作时间**。

### Dalvik虚拟机堆

在Dalvik虚拟机中，堆实际上就是一块匿名共享内存。Dalvik虚拟机并不是直接管理这块匿名共享内存，而是将它封装成一个mspace，交给C库来管理。为什么这么做？是因为内存碎片问题其实是一个通用问题，并不只是Dalvik虚拟机在Java堆为对象分配内存时会遇到，C库的malloc函数在分配内存时也会遇到。

Android系统使用的C库bionic使用了Doug Lea写的dlmalloc内存分配器，也就是说，我们调用函数malloc时，使用的是dlmalloc内存分配器来分配内存。这是一个成熟的内存分配器，可以很好的解决内存碎片问题。