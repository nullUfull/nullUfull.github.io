# repeatOnLifecycle API设计故事


> 翻译自[repeatOnLifecycle API design story](https://medium.com/androiddevelopers/repeatonlifecycle-api-design-story-8670d1a7d333)，使用Google or DeepL翻译以及人工校对，后来发现Google在各大平台已经发布了[中文版本](https://juejin.cn/post/7001371050202103838)，不过感受到了机翻的意思表达不那么明确，官方的中文版还OK的，正好也能对比一下，提高下英文水准。

在这篇博文中，你将了解`Lifecycle.repeatOnLifecycle` API背后的设计决策，以及为什么我们删除了2.4.0版`lifecycle-runtime-ktx`库的第一个alpha版本中添加的一些辅助函数。

一路走来，你会看到某些协程API在某些情况下的使用是多么危险，命名是多么困难，以及为什么我们决定在库中只保留低级别的`suspend` API。

另外，你会意识到所有的API决定都需要在复杂性、可读性和API的易错性方面做出一些权衡。

特别感谢Adam Powell、Wojtek Kaliciński、Ian Lake和Yigit Boyar提供的反馈和讨论这些API的形态。

> 注意：如果您正在寻找`repeatOnLifecycle`指南，请查看博文：[从AndroidUI收集流的更安全的方法](https://medium.com/androiddevelopers/a-safer-way-to-collect-flows-from-android-uis-23080b1f8bda)。

## repeatOnLifecycle
`Lifecycle.repeatOnLifecycle API`的诞生主要是为了让Android中UI层的Flow收集更加安全。它的可重启行为考虑到了UI的生命周期，使得它成为完美的默认API，只在屏幕上的UI可见时处理逻辑。

> 注意：`LifecycleOwner.repeatOnLifecycle`也是可用的。它将功能委托给它的`Lifecycle`。有了这个，任何已经属于`LifecycleOwner`范围的代码都可以省略显式接收器。

`repeatOnLifecycle`是一个挂起函数。`repeatOnLifecycle`会暂停调用的协程，然后在给定的生命周期达到目标状态或更高时，在一个新的协程中运行一个你作为参数传递的给定的代码块。如果生命周期的状态低于目标值，为该块启动的协程就会被取消。最后，`repeatOnLifecycle`函数本身不会恢复调用的协程，直到生命周期处于`DESTROYED`。

让我们看看这个API的作用。如果你读过我之前的博文：[从AndroidUI收集流的更安全的方法](https://medium.com/androiddevelopers/a-safer-way-to-collect-flows-from-android-uis-23080b1f8bda)，这些对你来说应该都不是惊喜。

```
class LocationActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // Create a new coroutine from the lifecycleScope
        // since repeatOnLifecycle is a suspend function
        lifecycleScope.launch {
            // Suspend the coroutine until the lifecycle is DESTROYED.
            // repeatOnLifecycle launches the block in a new coroutine every time the 
            // lifecycle is in the STARTED state (or above) and cancels it when it's STOPPED.
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                // Safely collect from locations when the lifecycle is STARTED
                // and stop collecting when the lifecycle is STOPPED
                someLocationProvider.locations.collect {
                    // New location! Update the map
                }
            }
            // Note: at this point, the lifecycle is DESTROYED!
        }
    }
}
```

> 注意：如果你对`repeatOnLifecycle`是如何实现的感兴趣，这里有一个[源代码链接](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:lifecycle/lifecycle-runtime-ktx/src/main/java/androidx/lifecycle/RepeatOnLifecycle.kt;l=63)。

## 为什么它是一个挂起函数
由于可以保留调用上下文，所以挂起函数是执行重启行为的最佳选择。它在调用协程时遵从`Job`树。由于`repeatOnLifecycle`的内部实现是使用`suspendCancellableCoroutine`，它可以和取消操作共同运作：取消发起调用的协程的同时也会取消`repeatOnLifecycle`和它重启执行的`block`。

另外，我们还可以在`repeatOnLifecycle`之上添加更多的API，比如`Flow.flowWithLifecycle`流操作符。更重要的是，如果你的项目需要它还允许你在这个API之上创建辅助函数。这就是我们试图用`LifecycleOwner.addRepeatingJob`API做的事情，我们在`lifecycle-runtime-ktx:2.4.0-alpha01`中添加了这个API，事实上，在`alpha02`中删除了它。

## 移除addRepeatingJob API
`LifecycleOwner.addRepeatingJob`API在库的第一个`alpha`版本中添加了这个功能，现在已经从库中移除，它是这样实现的。

```
/* Copyright 2022 Google LLC.	
   SPDX-License-Identifier: Apache-2.0 */
public fun LifecycleOwner.addRepeatingJob(
    state: Lifecycle.State,
    coroutineContext: CoroutineContext = EmptyCoroutineContext,
    block: suspend CoroutineScope.() -> Unit
): Job = lifecycleScope.launch(coroutineContext) {
    repeatOnLifecycle(state, block)
}
```

给定一个`LifecycleOwner`，你可以运行一个`suspend`块，每当它的生命周期进入和离开目标状态时就会重新启动。这个API使用`LifecycleOwner`的`lifecycleScope`来触发一个新的协程并在其中调用`repeatOnLifecycle`。

使用`addRepeatingJob`API，上面的代码会是这样的。

```
class LocationActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        lifecycleOwner.addRepeatingJob(Lifecycle.State.STARTED) {
            someLocationProvider.locations.collect {
                // New location! Update the map
            }
        }
    }
}
```

乍一看，你可能认为这种代码更干净，需要的代码更少。然而，如果你不仔细注意的话，有一些隐藏的问题会让你自食其果：

- 尽管`addRepeatingJob`接受一个挂起代码块，但`addRepeatingJob`不是一个挂起函数。因此，你不应该在协程中调用它！！
- 更少的代码？你只节省了一行代码，代价是拥有一个更容易出错的API。

第一点可能看起来很明显，但它总是让开发者上当。而且讽刺的是，实际上它是基于协程的一个最核心的概念：[结构化并发](https://elizarov.medium.com/structured-concurrency-722d765aa952)。

`addRepeatingJob`不是一个挂起函数，因此，默认情况下不支持结构化并发（需要你注意的是你可以通过使用`coroutineContext`参数来手动使其支持）。由于`block`参数是一个挂起的lambda，你把这个API与协程联系起来，你可以很容易地写出这样的危险代码：

```
class LocationActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val job = lifecycleScope.launch {

            doSomeSuspendInitWork()

            // DANGEROUS! This API doesn't preserve the calling Context!
            // It won't get cancelled when the parent coroutine is cancelled!
            addRepeatingJob(Lifecycle.State.STARTED) {
                someLocationProvider.locations.collect {
                    // New location! Update the map
                }
            }
        }

        // If something goes wrong, cancel the coroutine launched above
        try {
            /* ... */
        } catch(t: Throwable) {
            job.cancel()
        }
    }
}
```

`addRepeatingJob`做的是协程的事情，没有什么能阻止我在协程中调用它，对吗？

由于`addRepeatingJob`使用隐含在实现细节中的`lifecycleScope`创建新的协程来运行重复块，新的协程不遵从结构化的并发，也不保留调用协程的上下文。因此，当你调用`job.cancel()`时，将不会被取消。*这可能会导致你的应用程序中出现非常不易察觉的bug，很难调试*。

## repeatOnLifecycle才是大赢家

`addRepeatingJob`里面使用的隐式`CoroutineScope`是使API在某些情况下使用不安全的原因。这是一个隐藏的问题，需要特别注意才能写出正确的代码。这一点是避免在库中的 `repeatOnLifecycle`之上增加包装API的反复论证。

`suspend repeatOnLifecycle`API的主要好处是，它默认与结构化并发合作，而`addRepeatingJob`没有。它还能帮助你思考你希望重复的任务在哪个范围内执行。这个API是自解释的并且符合开发者的预期：

- 和其他的挂起函数一样，它会将当前协程的执行中断，直到特定事件发生。比如这里是当生命周期被销毁时继续执行。
- 没有意外惊吓! 它可以和其他的协程代码一起使用，并且会按照您的预期工作。
- `repeatOnLifecycle`周围的代码可读性高，对新手来说是有意义的："首先，我启动一个跟随用户界面的生命周期的新协程。然后，我需要调用`repeatOnLifecycle`使得每当用户界面达到这个生命周期状态时启动执行这个`block`"。

## Flow.flowWithLifecycle

`Flow.flowWithLifecycle`操作符（[实现在此](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:lifecycle/lifecycle-runtime-ktx/src/main/java/androidx/lifecycle/FlowExt.kt;l=87)）是建立在`repeatOnLifecycle`之上的，只有当生命周期至少处于`minActiveState`时，才会发射上游流发送的项目。

```
class LocationActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        lifecycleScope.launch {
            someLocationProvider.locations
                .flowWithLifecycle(lifecycle, STARTED)
                .collect {
                    // New location! Update the map
                }
        }
    }
}
```

尽管这个API也有一些需要注意的问题，但我们还是决定保留它，因为它作为一个Flow操作符很有用。例如，它可以[很容易地在Jetpack Compose中使用](https://manuelvivo.dev/coroutines-addrepeatingjob#safe-flow-collection-in-jetpack-compose)。尽管你可以通过使用`produceState`和`repeatOnLifecycle`API在Compose中实现同样的功能，但我们还是把这个API留在库中，作为更多响应式方法的替代。

正如KDoc中记录的那样，问题在于你添加`flowWithLifecycle`操作符的顺序很重要。在`flowWithLifecycle`操作符之前应用的操作符会在生命周期低于`minActiveState`时被取消。然而，在这之后应用的操作符不会被取消，即使没有项目被发送。

对于最好奇的人来说，这个API名称以`Flow.flowOn(CoroutineContext)`操作符为先例，因为`Flow.flowWithLifecycle`改变了用于收集上游流的`CoroutineContext`，而使下游不受影响。

## 我们应该增加一个额外的API吗？

鉴于我们已经有了`Lifecycle.repeatOnLifecycle`、`LifecycleOwner.repeatOnLifecycle`和`Flow.flowWithLifecycle`APIs。我们应该添加其他的API吗？

新的API所带来的混乱与它们要解决的问题一样多。有多种方法来支持不同的用例，而最短的路径在很大程度上取决于周围的代码。可能对你的项目有用的东西，可能对其他人没有用。

这就是为什么我们不想为所有可能的情况提供API，可用的API越多，对开发者来说就越混乱，不知道什么时候该用什么。因此，我们做出决定，只保留最底层的API。有时，少即是多。

## 命名很重要（也很困难）

这不仅关系到我们支持哪些用例，而且还关系到如何命名它们! 名称应该符合开发者的预期，并遵循Kotlin协程的约定。比如说：

- 如果API使用隐式`CoroutineScope`（例如`addRepeatingJob`中隐式使用的`lifecycleScope`）启动一个新的协程，这必须反映在命名中，以避免错误的预期。在这种情况下，`launch`应该以某种方式包含在名称中。
- `collect`是一个挂起函数。如果它不是一个挂起函数，就不要在API名称前加上`collect`。

> 注意：`Compose`的`collectAsState`API是一个特殊情况，我们对它的名字没有意见。它不能与挂起函数混淆，因为在`Compose`中没有`@Composable suspend fun`这样的东西。

甚至`LifecycleOwner.addRepeatingJob`的API也是一个很难命名的问题。因为它用`lifecycleScope`创建了新的协程，所以它的前缀应该是`launch`。然而，我们想消除协程在后台使用的事实，而且由于它添加了一个新的生命周期观察者，这个名字与其他`LifecycleOwner`APIs的名字更一致。

这个名字也多少受到了现有`LifecycleCoroutineScope.launchWhenX`挂起API的影响。由于`launchWhenStarted`和`repeatOnLifecycle(STARTED)`提供了完全不同的功能（`launchWhenStarted`暂停了协程的执行，而`repeatOnLifecycle`取消并重新启动一个新的协程），如果新API的名称相似（例如，用`launchWhenever`来表示重启API），开发者可能会感到困惑，甚至在不知不觉中交替使用它们。

## 一行代码收集数据流
LiveData的observe函数具有生命周期感知，并且只在生命周期至少开始时处理排放。如果你要从LiveData迁移到Kotlin flow，你可能会认为有一个单行的替换是个好主意！你可以删除模板代码，而且迁移很直接。

因此，您可以像[Ian Lake](https://twitter.com/ianhlake)在他第一次开始使用`repeatOnLifecycle`API 时所做的那样。他创建了一个名为`collectIn`的便捷包装器，如下所示（为了遵循上面讨论的命名约定，我将其重命名为`launchAndCollectIn`）：

```

/* Copyright 2022 Google LLC.	
   SPDX-License-Identifier: Apache-2.0 */
inline fun <T> Flow<T>.launchAndCollectIn(
    owner: LifecycleOwner,
    minActiveState: Lifecycle.State = Lifecycle.State.STARTED,
    crossinline action: suspend CoroutineScope.(T) -> Unit
) = owner.lifecycleScope.launch {
        owner.repeatOnLifecycle(minActiveState) {
            collect {
                action(it)
            }
        }
    }
```

这样你就可以像这样从用户界面调用它：

```
class LocationActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        someLocationProvider.locations.launchAndCollectIn(this, STARTED) {
            // New location! Update the map
        }
    }
}
```

这个包装器，虽然在这个例子中看起来很好很直接，但它存在着我们前面提到的关于`LifecycleOwner.addRepeatingJob`的同样问题。它不尊重调用的上下文，在其他循环程序中使用会很危险。此外，原来的名字真的很有误导性：`collectIn`并不是一个挂起的函数 如前所述，开发者希望`collect`函数能够挂起。也许，这个封装函数的更好的名字可以是`Flow.launchAndCollectIn`，以防止不良使用。

## iosched的封装函数

`repeatOnLifecycle`必须与Fragments中的`viewLifecycleOwner`一起使用。在开源的 Google I/O 应用程序[iosched](https://github.com/google/iosched)项目中，团队决定创建一个封装函数，以避免在 Fragments 中滥用 API，其 API 名称非常明确：[Fragment.launchAndRepeatWithViewLifecycle](https://github.com/google/iosched/blob/main/mobile/src/main/java/com/google/samples/apps/iosched/util/UiUtils.kt#L60)。

> 注意：实现与`addRepeatingJob`API非常相似。当使用`alpha01`版本的库编写此代码时，在`alpha02`中添加的`repeatOnLifecycle`API lint 检查没有到位。


## 你需要封装函数吗？

如果你需要在`repeatOnLifecycle`API之上创建包装器，以适应你的应用程序中最常见的用例，请问你是否真的需要它，以及为什么需要它。如果你确信并想继续下去，我建议你选择一个非常明确的API名称，以明确定义包装器的行为，避免误用。同时，要非常清楚地记录它，让新来的人能够充分理解使用它的意义。

我希望这篇博文能让您了解团队在决定如何处理`repeatOnLifecycle`API以及我们可以在其之上添加的潜在帮助方法时的考虑因素。

再次感谢 Adam Powell、Wojtek Kaliciński、Ian Lake 和 Yigit Boyar 提供反馈并讨论这些 API 的形态。
