# 2021_Android知识整理

## Android 基础知识

1. Acitvity 的生命周期是什么样的？如何摧毁一个 Activity?
2. Activity 的 4 大启动模式，与开发中需要注意的问题，如 onNewIntent() 的调用；a. Activity 的启动模式，区别 b. singleInstance 如果不指定栈名，是怎么分配的？
3. Intent 显示跳转与隐式跳转，如何使用？
4. Activity A 跳转 B，B 跳转 C，A 不能直接跳转到 C，A如何传递消息给 C？
5. Activity 如何保存状态的？
6. 请描诉 Activity 的启动流程，从点击图标开始；a. APP 是怎么启动的？b. 启动一个Activity的流程分析
7. Service 的生命周期是什么样的？a. Service 两种生命周期以及区别
8. 你会在什么情况下使用 Service？
9. startServer 和 bindServier 的区别？
10. Service 和 Thread 的区别？
11. IntentService 与 Service 的区别？
12. ContentProvider 如何自定义与使用场景是什么？
13. BroadcastReciver 的静态注册与动态注册的区别？
14. 广播的分类与工作原理
15. 可以再 onReceive 中开启线程么，会有什么问题？
16. 什么是有序广播？
17. Application、Activity、Service中context 的区别？能否启动一个 activity、dialog?
18. Fragment 的生命周期？
19. Fragment 的构造函数为啥不让传参？
20. Fragment add 与 replace 的区别，分别对 Fragment 的生命周期影响
21. 内存不足时候使用的字段
22. 你遇到的内存泄露的情况
23. 安卓点九图的使用
24. Fragment 与 Activity 之间的通信
25. 点击事件 5s 内不响应的话，该如何处理
26. BroadcastReceiver 和 LocalBroaccastReceiver 的区别
27. 安卓的跨进程通信
28. 怎么用 OkHttp 监控数据请求的状态
29. 消息机制中 Message 的创建方式
30. Android 进程通信的方法
31. Intent 是怎么进程通信的
32. 内存泄漏有哪几种情况
33. Activity A 启动 Activity B，两个Activity的生命周期顺序
34. 关于Android Activity之间传递数据的方式
35. 同一个进程里面activity和另外一个activity的交互方法
36. SDK如何嵌入到APP当中
37. fragment和activity的交互和生命周期
38. 安卓中有内存泄露，其中单例的情况详细说下
39. 安卓里面广播的注册方式，为啥静态注册占用内存比较大
40. bimap压缩图像



## Android View体系

1. View 绘制流程与自定义 View 注意点
2. 在 onResume 中可以测量宽高吗？
3. 事件分发机制是什么过程？
4. 冲突怎么解决？
5. View 分发反向制约的方法？
6. 自定义 Behavior，NestScroll，NestChild
7. View.inflater 过程与异步 inflater
8. inflater 为什么比自定义 View 慢？
9. onTouchListener，onTouchEvent，onClick 的执行顺序
10. 怎么拦截事件 onTouchEvent 如果返回 false，onClick 还会执行么？
11. 事件的分发机制，责任链模式的优缺点 
12. 动画的分类以及区别
13. 属性动画与普通的动画有什么区别？
14. 插值器 估值器的区别？
15. RecyclerView 与 ListView 的对比，缓存策略，优缺点
16. WebView 如何做资源缓存？
17. WebView 和 JS 交互的几种方式与拦截方法
18. 自定义 view 与 viewgroup 的区别
19. View 的绘制原理
20. View 中 onTouch，onTouchEvent 和 onClick 的执行顺序
21. View 的滑动方式
22. invalidate() 和 postInvalicate() 区别
23. View 的绘制流程是从 Activity 的哪个生命周期方法开始执行的
24. Activity，Window，View 三者的联系和区别
25. 如何实现 Activity 窗口快速变暗
26. ListView 卡顿的原因以及优化策略
27. ViewHolder 为什么要被声明成静态内部类
28. Android 中的动画有哪些? 动画占用大量内存，如何优化
29. 自定义 View 执行 invalidate()方法，为什么有时候不会回调 onDraw()
30. DecorView，ViewRootImpl，View 之间的关系，ViewGroup.add()会多添加一个 ViewrootImpl 吗？
31. 如何通过WindowManager添加Window(代码实现)？
32. 为什么Dialog不能用Application的Context？
33. WindowMangerService中token到底是什么？有什么区别？
34. RecyclerView 是什么？如何使用？如何返回不一样的 Item
35. RecyclerView 的回收复用机制
36. 如何给 ListView & RecyclerView加上拉刷新 & 下拉加载更多机制
37. 如何对 ListView & RecycleView 进行局部刷新的？
38. ScrollView 下嵌套一个 RecycleView 通常会出现什么问题？
39. 一个 ListView 或者一个 RecyclerView 在显示新闻数据的时候，出现图片错位，可能的原因有哪些 & 如何解决？
40. Requestlayout，onlayout，onDraw，DrawChild 区别与联系
41. 如何优化自定义 View
42. Android 属性动画实现原理，补间动画实现原理
43. 与 ListView 对比，RecyclerView 的优点
44. RecyclerView 的缓存机制
45. 触摸事件的传递机制
46. Window 机制中的 DecorView 的关系
47. 安卓如何渲染页面的

    

## Android Framework

1. Android 中多进程通信的方式有哪些？进程通信你用过哪些？原理是什么？
2. 描述下 Binder 机制原理？
3. Binder 线程池的工作过程是什么样？
4. Handler 怎么进行线程通信，原理是什么？
5. Handler 如果没有消息处理是阻塞的还是非阻塞的？
6. handler.post(Runnable)  runnable 是如何执行的？
7. handler 的 Callback 和 handlemessage 都存在，但 callback 返回 true handleMessage 还会执行么？
8. Handler 的 sendMessage 和 postDelay 的区别？
9. IdleHandler 是什么？怎么使用，能解决什么问题？
10. 为什么 Looper.loop 不阻塞主线程？a.Looper 无限循环为啥没有 ANR
11. Looper 如何在子线程中创建？
12. Looper、handler、线程间的关系。例如一个线程可以有几个 Looper 可以对应几个 Handler？
13. 如何更新 UI，为什么子线程不能更新 UI？
14. ThreadLocal 的原理，以及在 Looper 是如何应用的？
15. Android 有哪些存储数据的方式？
16. SharedPreference 原理，commit 与 apply 的区别是什么？使用时需要有哪些注意？
17. 如何判断一个 APP 在前台还是后台？
18. 如何做应用保活？
19. 一张图片 100x100 在内存中的大小？
20. Intent的 原理，作用，可以传递哪些类型的参数?
21. 如果需要在 Activity 间传递大量的数据怎么办？
22. 打开多个页面，如何实现一键退出?
23. LiveData 的生命周期如何监听的?



## Android 性能优化

1. App 稳定性优化
2. App 启动速度优化
3. App 内存优化
4. App 绘制优化
5. App 瘦身
6. 网络优化
7. App 电量优化
8. Android 安全优化
9. 为什么 WebView 加载会慢呢？
10. 如何优化自定义 View
11. FC(Force Close)什么时候会出现？
12. Java 多线程引发的性能问题，怎么解决？
13. TraceView 的实现原理，分析数据误差来源
14. 是否使用过 SysTrace，原理的了解？
15. mmap + native 日志优化？
16. 如何提升用户体验感



## Android 三方框架

1. Glide ：加载、缓存、LRU 算法 (如何自己设计一个大图加载框架) （LRUCache 原理）
2. EventBus
3. LeakCanary
4. ARouter
5. 插件化（不同插件化机制原理与流派，优缺点。局限性）
6. 热修复
7. RXJava （RxJava 的线程切换原理）
8. Retrofit （Retrofit 在 OkHttp 上做了哪些封装？动态代理和静态代理的区别，是怎么实现的）
9. OkHttp



