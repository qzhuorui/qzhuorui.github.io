---
layout: post
title: 'CountDownLatch使用&总结'
date: 2021-02-06
author: qzhuorui
color: rgb(98,170,255)
tags: 并发
---


# CountDownLatch使用&总结

> 计数器锁。使一个线程在等待其他线程执行完成后，再继续执行

## 使用场景一：
有一个异步的请求，在onSuccess()和onFailure()中有不同的逻辑，且会对异步执行完后的操作有影响
当然可以考虑使用handler在不同情况下分发处理，但是增加代码量，并且分发不易于直观阅读，所以使用CountDownLatch

```
//伪代码如下
mCountDownLatch = new CountDownLatch(1);

requestDeal(params , new Callback(){
	@Override
	public void onSuccess(){
		mCountDownLatch.countDown();
	}
	
	@Override
	public void onFailure(){
		mCountDownLatch.countDown();
	}
});

try{
	mCountDownLatch.await();
}catch(Exception e){
}

```

以上情况时：在执行异步请求操作的同时，程序会 阻塞 在 await处，等待onSuccess和onFailure处理后，计数器为0，此时await才会放行，程序继续往下走。
适合使用的情况是：接下来的程序逻辑需要依赖onSuc和onFail的处理（比如说决定是否await之后继续执行后面的逻辑）

引用知乎的一篇文章：`https://zhuanlan.zhihu.com/p/148231820`