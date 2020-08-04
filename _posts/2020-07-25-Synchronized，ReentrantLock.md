---
layout: post
title: 'qzhuorui'
date: 2020-07-25
author: qzhuorui
color: rgb(255,210,32)
tags: 并发
---



# Synchronized，ReentrantLock

## 一、Synchronized

Synchronized可以用来修饰三个层面：

1. 修饰 实例方法
2. 修饰 静态类方法
3. 修饰 代码块

### 1.修饰 实例方法

```
public class A{
    private int sum = 0;
    public synchronized void calculate(){
        sum++;
    }
}
```

这种情况下的**锁对象是当前实例对象** ，因此**只有同一个实例对象调用此方法才会产生互斥效果** ，不同实例对象之间不会有互斥效果。

### 2.修饰 静态类方法

如果synchronized修饰的是静态类方法，则**锁对象是当前类的Class对象** 。因此即使在**不同线程中调用不同实例对象，也会有互斥效果** 。

### 3.修饰 代码块

除了直接修饰方法外，synchronized还可以作用于代码块。

```
public class A{
    private Object lock = new Obejct();
    
    public void printLog(){
        synchronized(lock){
            
        }
    }
}
```

synchronized作用于代码块时，锁对象就是跟在后面括号中的对象。任何Object对象都可以当作锁对象。

### 实现细节

synchronized既可以作用于方法，也可以作用于某一代码块。但在实现上是有区别的。比如如下代码，使用synchronized作用于代码块：

```
public class Foo{
    private int number;
    
    public void test(){
        int i=0;
        synchronized(this){
            number = i +1;
        }
    }
}
```

1.使用Javap查看上述test方法的字节码，可以看出，编译而成的字节码中会包含1个moitorenter和2个mointerexit。这是因为虚拟机需要保证当异常发生时也能释放锁。因此2个mointerexit一个是代码正常执行结束后释放锁，一个是在代码异常时释放锁。

2.当使用synchronized修饰的方法被编译为字节码后，在方法flags属性中会被标记为ACC_SYNCHRONIZED标志。当虚拟机访问一个被标记为ACC_SYNCHRONIZED的方法时，会自动在方法的开始和结束（或异常）位置添加moitorenter和mointerexit指令。

moitorenter和mointerexit，可以理解为一把具体的锁。在这个锁中保存着两个比较重要的属性：计数器和指针。计数器代表当前线程一共访问了几次这把锁；指针指向持有这把锁的线程。

## 二、ReentrantLock

### 1.ReentrantLock基本使用

ReentrantLock的使用和sync有点不同，它的**加锁和解锁都需要手动** 完成。

```
ReentrantLock lock = new ReentrantLock();

public void printLog(){
	try{
        lock.lock();
        //...
	}catch(E e){
	}finally{
        lock.unlock();
	}
}
```

将unlock()操作放在finally代码块中。这是因为ReentrantLock和sync不同，当异常发生时sync会自动释放锁，但是ReentrantLock并不会。所以放在finally中确保任何时候锁都可以被正常释放掉。



### 2.公平锁实现

ReentrantLock有一个带参数的构造器

```
public ReentrantLock(boolean fair){
    sync = fair ? new FairSync() : new NofairSync();
}
```

默认情况下，sync和ReentrantLock都是**非公平锁** 。但是ReentrantLock可以通过传入true来创建一个公平锁。

公平锁就是通过同步队列来实现 多个线程按照 申请锁的顺序 获取锁。

按照申请锁的顺序 获取锁！！！同步队列！！！

### 3.读写锁（ReentrantReadWriteLock）

在常见的开发中，我们经常会定义一个线程共享的 用作缓存的 数据结构，比如一个较大的Map。缓存中保存了全部的城市Id和城市name对应关系。

这个大Map绝大部分时间提供读服务（根据id查找name）。而写操作占有时间很少，通常是在服务启动时初始化，然后可以每隔一定时间再刷新缓存的数据。

但是写操作开始到结束之间，不能再有其他读操作进来，并且写操作完成之后的更新数据需要对后续的读操作可见。

---

在没有读写锁支持时，如果想要完成上述工作就需要使用Java的等待通知机制：当写操作开始时，所有晚于写操作的读操作均会进入等待状态，只有写操作完成并通知之后，所有等待的读操作才能继续执行。这样的目的是为了使读操作可以取到正确的数据，不会出现脏读

但是如果使用concurrent包中的读写锁（ReentrantReadWriteLock）实现上述功能，就只需要在读操作时获取读锁，写操作时获取写锁。当写锁被获取到时，后续的读写锁都会被阻塞，写锁释放后，所有操作继续执行，这种编程方式相对于使用等待通知机制的实现方式而言，变得简单明了。

```
//创建读写锁对象
ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();

//通过读写锁对象分别获取 读锁，写锁
ReentrantReadWriteLock.ReadLock readLock = rwLock.readLock();
ReentrantReadWriteLock.WriteLock writeLock = rwLock.writeLock();

//使用读/写锁，同步缓存的，读/写操作
//读操作
readLock.lock();
try{
    //从缓存中读取数据
}finally{
    readLock.unlock();
}
//写操作
writeLock.lock();
try{
    //向缓存中写入数据
}finally{
    writeLock.unlock();
}
```

通过代码日志可以看到，当写入操作在执行时，读取数据的操作会被阻塞。当写入数据操作执行成功后，读取操作继续执行，并且读取的数据也是最新写入后的实时数据。

## 三、总结

主要介绍了Java中两个实现同步的方式sync和ReentrantLock。其中sync使用更简单，加锁和释放锁都是由虚拟机自动完成，而ReentrantLock需要开发者手动去完成。但是ReentrantLock使用场景更多，公平锁和读写锁都可以在复杂情况下发挥作用。

