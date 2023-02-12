# 不错的技术文章


# kotlin
[Android-Redux-Intro](https://jayrambhia.com/blog/android-redux-intro)
## 协程
[Coroutines : First things first](https://mp.weixin.qq.com/s/HCjf1MBOMLvpy_SPiSr67w)

[如何优雅的取消协程 ？](https://mp.weixin.qq.com/s/h8Qg_5fLkpcNDzP3IZDyqg)

[Kotlin Coroutines(协程) 完全解析（三），封装异步回调、协程间关系及协程的取消](https://johnnyshieh.me/posts/kotlin-coroutine-integration-and-cancel/)

[[译] 如何优雅的处理协程的异常？](https://juejin.im/post/5ebeaef5f265da7bcb65ff80)

[Kotlin 协程五 —— 在Android 中使用 Kotlin 协程](https://www.cnblogs.com/joy99/p/15805969.html)

[Kotlin Coroutines dispatchers](https://kt.academy/article/cc-dispatchers)

[Kotlin 协程](https://www.hiczp.com/kotlin/kotlin-xie-cheng.html) 
谈了谈CPS变换以及状态机

[硬核万字解读——Kotlin协程原理解析](https://mp.weix
in.qq.com/s/N9BiufCWTRuoh6J-QERlWQ) 
贴了比较多的源码，说得也很不错，搭配类图很直观，很硬核

[Design of Kotlin Coroutines](https://proandroiddev.com/design-of-kotlin-coroutines-879bd35e0f34) 
也是源码解析，和上面的有点类似

[Which of coroutines (goroutines and kotlin coroutines) are faster](https://stackoverflow.com/questions/46864623/which-of-coroutines-goroutines-and-kotlin-coroutines-are-faster)


[为什么无栈协程不能被非协程函数嵌套调用？](https://www.zhihu.com/question/458843623)

[官方Coroutines doc](https://github.com/Kotlin/KEEP/blob/master/proposals/coroutines.md)

### 疑问

- [ ] CPS变换
- [ ] 既然kotlin协程从实现方式来说一种无栈协程，continuation和callback的区别到底在哪里？
- [ ] 

「无栈协程」大多数都是类似 Kotlin 这种实现，将一个函数体拆成多个 subroutine ，执行异步操作时，通过 continuation 保存上下文状态（包括代码行号、局部变量），并从当前函数退出；当异步操作完成时，会通过 continuation 重新进入该函数，并恢复上下文。

## flow
[实战 | 使用 Kotlin Flow 构建数据流 "管道"](https://juejin.cn/post/7078225994871472158)

[设计repeatOnLifecycle API 背后的故事](https://juejin.cn/post/7001371050202103838)

[repeatOnLifecycle API design story](https://medium.com/androiddevelopers/repeatonlifecycle-api-design-story-8670d1a7d333)

[A safer way to collect flows from Android UIs](https://medium.com/androiddevelopers/a-safer-way-to-collect-flows-from-android-uis-23080b1f8bda)

[Flow 操作符 shareIn 和 stateIn 使用须知](https://juejin.cn/post/6998066384290709518)

[Deep dive into Coroutine Flow 2](https://myungpyo.medium.com/deep-dive-into-coroutine-flow-2-d43ba0d1f45d)

[Migrating from LiveData to Kotlin’s Flow](https://medium.com/androiddevelopers/migrating-from-livedata-to-kotlins-flow-379292f419fb)

## 内存泄露
[ViewLifecycleLazy and other ways to avoid View memory leaks in Android Fragments](https://bladecoder.medium.com/viewlifecyclelazy-and-other-ways-to-avoid-view-memory-leaks-in-android-fragments-4aa982e6e579)

# android
## binder
[Binder | 内存拷贝的本质和变迁](https://juejin.cn/post/6844904113046568973)

[认真分析mmap：是什么 为什么 怎么用 ](https://www.cnblogs.com/huxiao-tee/p/4660352.html)

# java

## jvm
[你知道 JVM 的方法区是干什么用的吗？](https://zhuanlan.zhihu.com/p/166190558)
