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
5. 

### 

## 复习

### 模块化
1. 模块之间的通信？

单工程->模块化->组件化

模块化：按业务模块来划分
组件化：各个业务会依赖到的可重用的逻辑，比如说日志组件、网络组件、ui组件等

而模块之间的通信呢？

我们这边是用的Router，各个模块会依赖Router基础库，向外部暴露统一的接口，注册到Router中，因此各个模块都可以通过Router来获取到其他模块对外暴露的接口，通过Router的中转，依赖接口隔离各个模块的具体的实现逻辑，所以各模块之间的依赖关系是很清晰的。

api 对外暴露的类？



### Java基础

#### 并发

##### ConcurrentHashmap
[深入解析ConcurrentHashMap：感受并发编程智慧](https://segmentfault.com/a/1190000038416595)

### Android
#### ViewModel
1. 生命周期是怎么样的

2.0之前空的Fragment

2.0以后，AndroidX的支持，LifecycleOwner支持
