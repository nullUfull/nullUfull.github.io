# 

# 面试经历


## 简历
1. 热爱移动产品研发，
2. 熟练掌握java、kotlin，数据Android SDK、熟悉Android的UI/网络/数据库框架
3. 具备Android开发经验，能独立开发Android App
4. 对软件产品有强烈的责任心，具备良好的沟通能力和优秀的团队协作能力



1. 卡顿监控
2. 连续掉帧  流畅率 指标是怎么样的

## 项目

### 话题列表页
1. MVI Flow搭建Redux核心框架  上报也是使用action
2. 演变 MVI雏形，viewmodel暴露dispatch接口-> 讨论架构，引入redux -> 存在ViewAction和StateAction -> 存在usecase层，抽出viewmodel中可复用的逻辑 -> 新增ReportAction用于上报 -> 引入middleware，usecase中的逻辑迁移到此 -> 去除ViewAction
3. Repository对外暴露数据的接口使用Flow承载
4. 包括为了兼容旧的业务逻辑，以前是存在很多EventBus的方式接受请求回报，统一封装在repository层，对外屏蔽EventBus，这种应该退出历史舞台
5. View可以内部闭环的逻辑就没必要通过Viewmodel去处理了
6. 历史债：使用jce定义的数据结构，非data class，无法做数据diff
7. 为什么UiState下还要区分HasData、NoData
8. 如果列表中的Item的功能足够复杂，所以会拆分为多个区域去处理对应的UI显示以及以及业务逻辑，与此同时，对应的区域存在对应的数据去描述，而且需要处理局部刷新的问题：对比数据变化->UI变化的区域，使用payload去进行刷新，如果用户的一个操作会导致多个UI区域刷新，则需要payload多个数据到recyclerView的item，所以getChangePayload中的逻辑则是这样的：创建一个用于list用于容纳payloads，如果某个区域数据变化，则add该区域的payload数据，对应的区域处理对应的payload
9. EventBus收归到Repository，EventBusRepository，对外提供flow
10. copy不会导致内存爆掉吗？因为state都是用的copy的方式去描述页面状态的变化，答案当然是不会，state的变化并不是深拷贝，而是拷贝所变化的部分，也就是差异化拷贝，可以理解state是一个树状的一个结构，一般来说state树发生变化也只是叶子节点的数据发生变化，每一次的变化不会涉及到整个state的对象的深拷贝，所以内存的压力可以不用考虑
11. 

### 

## 复习

### Android

#### 常见的框架

##### Glide

##### OKHttp

#### 虚拟机
[dalvik vs art](https://www.jianshu.com/p/1361d1e3a344)

#### Binder

- 为什么AMS和zygote进程通信不用Binder，而是socket？因为fork操作不能在多线程的环境下
[为什么SystemServer进程与Zygote进程通讯采用Socket而不是Binder](https://codeantenna.com/a/jj2VwT8qlS)

##### mmap
mmap是一种内存映射文件的方法，它将一个文件映射到进程的地址空间中，实现文件磁盘地址和进程虚拟地址空间中一段虚拟地址的一一对映的关系。实现这样
的映射关系后，进程就可以采用指针的方式读写操作这一段内存，而系统会自动会写脏页面到对应的文件磁盘上，即完成了对文件的操作而不必要再调用read、write等系统
调用函数。相反，内核空间对这段内存区域的修改也直接反映用户空间，从而实现不同进程间的数据共享

![pc](https://upload-images.jianshu.io/upload_images/13774375-211e6032044be1ba?imageMogr2/auto-orient/strip|imageView2/2/w/682/format/webp)


#### 模块化
1. 模块之间的通信？

单工程->模块化->组件化

模块化：按业务模块来划分
组件化：各个业务会依赖到的可重用的逻辑，比如说日志组件、网络组件、ui组件等

而模块之间的通信呢？

我们这边是用的Router，各个模块会依赖Router基础库，向外部暴露统一的接口，注册到Router中，因此各个模块都可以通过Router来获取到其他模块对外暴露的接口，通过Router的中转，依赖接口隔离各个模块的具体的实现逻辑，所以各模块之间的依赖关系是很清晰的。

api 对外暴露的类？



### Java基础

#### 动态代理
[Android插件化原理解析——Hook机制之动态代理](https://weishu.me/2016/01/28/understand-plugin-framework-proxy-hook/)

#### 并发

##### 生产者消费者
- 为什么condition.signal要在lock.unlock之前，这之间的是怎么运作的？
[ReentrantLock中Condition的wait方法、signal方法简单场景回顾](https://codeantenna.com/a/6YTX3qDfFG)

##### ConcurrentHashmap
[深入解析ConcurrentHashMap：感受并发编程智慧](https://segmentfault.com/a/1190000038416595)

### Android
#### ViewModel
1. 生命周期是怎么样的

2.0之前空的Fragment

2.0以后，AndroidX的支持，LifecycleOwner支持


## 面试题目

- 聊一下 Binder 的实现，如果让你 hook binder 拿到 binder 里面传输的数据你会怎么做
[Android知识体系之Binder Hook](https://www.zybuluo.com/TryLoveCatch/note/2287028)
- MVC & MVP & MVVM，这么多设计模式你都用过，你觉得哪个更好，他们之前主要的差别是什么
[Android架构学习之路三-MVX](https://ljd1996.github.io/2022/01/11/Android%E6%9E%B6%E6%9E%84%E5%AD%A6%E4%B9%A0%E4%B9%8B%E8%B7%AF%E4%B8%89-MVX/)
- 聊一下 coroutines 的底层实现

开放性问题
- 你对自己有什么规划
- 最近在关注什么新的技术，
- 有没有做过一下性能优化的例子，展开讲讲
- 如果加入隐私合规，你觉得会给隐私合规团队带来什么
