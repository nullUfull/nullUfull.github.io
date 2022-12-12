# kotlin flow


## 基础

`Flow`是一种基于流的编程模型，可以理解在`Kotlin`协程中提供的响应式编程方式就是`Flow`

![](https://developer.android.com/static/images/kotlin/flow/flow-entities.png)

`Flow`的定义很简单，表示它是一个可以被订阅收集的东西
```
public interface Flow<out T> {
    @InternalCoroutinesApi
    public suspend fun collect(collector: FlowCollector<T>)
}
```

而其中的`FlowCollector`代表流的接收者，其中定义了发送数据的功能。

```
public interface FlowCollector<in T> {

    /**
     * Collects the value emitted by the upstream.
     * This method is not thread-safe and should not be invoked concurrently.
     */
    public suspend fun emit(value: T)
}
```


### LiveData

1. 简单易用，但难以处理复杂数据
2. `postValue`丢失数据
3. 数据倒灌
4. 不防抖

### 冷流

`Flow`是一种 “冷流”(`Cold Stream`)。

冷流是一种数据源，该类数据源的生产者会在每个监听者开始消费事件的时候执行，从而在每个订阅上创建新的数据流。

这种冷流的行为我们可以从`SafeFlow`的实现中得知，在订阅的时候会执行传入的代码块：

```
private class SafeFlow<T>(private val block: suspend FlowCollector<T>.() -> Unit) : AbstractFlow<T>() {
    override suspend fun collectSafely(collector: FlowCollector<T>) {
        // 执行代码块
        collector.block()
    }
}
```

而一旦消费者停止监听（例如：`collect`所在协程被取消了），数据流将会被自动关闭，不会再发射数据。

1. 结构式并发；父子协程之间的关系；主从作用域、协同作用域
2. 协程的取消
3. `UI`层订阅`Flow`的最佳实践

#### Flow的创建
```
flow {
    delay(100) // ok
    emit(1) // ok
    withContext(Dispatcher.IO) {
        emit(2) // fail with ISE
    }
}

public fun <T> flow(
    @BuilderInference block: suspend FlowCollector<T>.() -> Unit
): Flow<T> = SafeFlow(block)
```

1. `flow`是一个高阶函数，参数是带接收者`FlowCollector`并且有`suspend`修饰符的函数类型，意味着`block`代码块中可以调用挂起函数
2. `emit`函数不是线程安全的，不应该并发调用，所以再`flow`的代码块中不允许切换线程

除此之外还封装了`asFlow`、`flowOf`等函数，都是基于`flow{}`实现的，方便调用方使用

```
public fun <T> Iterable<T>.asFlow(): Flow<T> = flow {
    forEach { value ->
        emit(value)
    }
}

@FlowPreview
public fun <T> (suspend () -> T).asFlow(): Flow<T> = flow {
    emit(invoke())
}

public fun <T> flowOf(vararg elements: T): Flow<T> = flow {
    for (element in elements) {
        emit(element)
    }
}
```

### 热流

#### SharedFlow

共享流 1->n

```
public interface SharedFlow<out T> : Flow<T> {
    /**
     * A snapshot of the replay cache.
     */
    public val replayCache: List<T>
}
```

以广播方式在其所有`Flow`器之间共享发射值的热流，以便所有收集器获得所有发射值。共享流称为热流，因为它的活动实例独立于收集器的存在而存在。

这与常规`Flow`相反，例如由`flow { ... }`函数定义的，它是冷的并且为每个收集器单独启动。

共享流程永远不会完成。在共享流上调用`Flow.collect`永远不会正常完成，由`Flow.launchIn`函数启动的协程也不会正常完成。共享流的活动收集器称为订阅者。

#### StateFlow

特殊的`ShareFlow`  `replay = 1` 对标`LiveData`，必须有一个默认值，因为对外提供了一个接口用于获取当前值

```
public interface StateFlow<out T> : SharedFlow<T> {
    /**
     * The current value of this state flow.
     */
    public val value: T
}
```

事件：事件是一次有效的，新订阅者不应该收到旧的事件，事件数据适合用`SharedFlow`(replay=0)
状态：状态是可以恢复的，新订阅者允许收到旧的状态数据，状态数据适合用`StateFlow`


比如说`ViewModel`通知`View`显示一个弹窗提示，切后台后回来会走`onStart`，会重新订阅事件对应的`Flow`，因此会再次收到事件，而显示弹窗

### 操作符

#### 终端操作符
```
collect
collectLatest // 有新值发出时，如果此时上个收集还没完成，则会取消掉上个值的收集操作并开始新收集操作
fold
reduce
lanchIn
first // 返回流发出的第一个元素然后取消流收集，如果为空会抛异常
firstOrNull // 返回流发出的第一个元素然后取消流收集。如果流为空，则返回null 
...
```

#### 中间操作符
```
// 回调
onStart
onEach
onCompletion
onEmpty
onSubscription

// 变换
transform // 对发出的值进行变换
transformLatest // 当原始流发出一个新值时，前一个transform块被取消
transformWhile // 这个变换的lambda的返回值为Boolean，如果为false则不再进行后续变换（取消订阅），为ture则继续执行

fun Flow<DownloadProgress>.completeWhenDone(): Flow<DownloadProgress> =
    transformWhile { progress ->
        emit(progress) // always emit progress
        !progress.isDone() // continue while download is not done
    }
}

map
mapNotNull
mapLatest // 当有新值发送时如果上个变换还没结束，会先取消掉

// 过滤
filter
filterInstance
filterNot
filterNotNull
takeWhile // 返回包含满足给定predicate的第一个元素的流
debounce // 防抖节流，指定时间内的值只接收最新的一个，其他的过滤掉。搜索联想场景适用
sample // 采样
distinctUntilChanged

// 组合
combine
combineTransform
merge
flattenConcat

// 功能
merge
catch
buffer
conflate
```

### UI收集流的最佳实践

生命周期 资源浪费

launch vs launchWhenX vs repeatOnLifecycle

![](https://miro.medium.com/max/720/0*pDKnvDJ9FzaCgXCd)

launch
和视图的生命周期同步

launchWhenX
在应用程序处于后台时接收更新可能会导致崩溃，这可以通过暂停视图中的收集来解决。然而，当应用程序在后台时，上游流量保持活跃，可能会浪费资源。

repeatOnLifecycle
每当生命周期处于STARED会在新的协程中启动执行代码块，并在生命周期进入STOPPED时取消协程

官方推荐

```
    // 创建新的协程
    lifecycleScope.launch {
        // 每当生命周期处于STARED会在新的协程中启动执行代码块，并在生命周期进入STOPPED时取消协程
        repeatOnLifecycle(Lifecycle.State.STARTED) {
            // 当生命周期处于STARTED时安全地从flow中获取数据，生命周期进入STOPPED时停止收集数据
            viewModel.coldFlow.collect {
                Log.i("FlowTest", "repeatOnLifecycle $it")
            }
        }
        // 当运行到此处，生命周期已进入DESTROYED状态

        repeatOnLifecycle(Lifecycle.State.STARTED) {
            viewModel.coldFlow.collect {
                Log.i("FlowTest", "repeatOnLifecycle $it")
            }
        }
    }

```

### 冷流转化为热流

使用`shareIn`or`shateIn`扩展方法可以使冷流转化为热流

```
public fun <T> Flow<T>.shareIn(
    scope: CoroutineScope, // 指定开始共享的协程作用域
    started: SharingStarted, // 控制什么时候启动和停止共享的策略
    replay: Int = 0 // 重播给新订阅者的值的数量(不能为负数，默认为零)
): SharedFlow<T>

public fun <T> Flow<T>.stateIn(
    scope: CoroutineScope,
    started: SharingStarted,
    initialValue: T // 初始值
): StateFlow<T>

```

#### 共享策略

- `SharingStarted.Eagerly` 立即开始共享并且不会停止
- `SharingStarted.Lazily` 当收个订阅者出现时开始共享并且会停止
- `SharingStarted.WhileSubscribed` 当第一个订阅者出现时开始共享，当最后一个订阅者消失时立即停止共享（默认），永远保持重播缓存（默认）

```
public fun WhileSubscribed(
    stopTimeoutMillis: Long = 0, // 配置最后一个订阅者消失和共享协程停止之间的延迟（以毫秒为单位)，默认为零（立即停止）
    replayExpirationMillis: Long = Long.MAX_VALUE // 配置共享协程停止和重放缓存重置之间的延迟（以毫秒为单位）默认为Long.MAX_VALUE（永远保留重放缓存，不重置缓冲区）
)
```

推荐使用`SharingStarted.WhileSubscribed(5000)`共享策略作为默认实现

1. 如果`Activity`配置发生改变，数据是不会丢的，5000ms足够`Activity`从配置改变到恢复
2. 可以搭配`repeatOnLifecycle`（UI层安全收集流）使用，如果应用切后台则UI层会取消对于流的`collect`（取消订阅），而这里因为配置`WhileSubscribed`策略超时后会自动停止共享，共享流会断开对上游冷流的`collect`（取消订阅），上游冷流的数据的生产也就会被`cancel`掉，避免应用处于后台时还存在数据的生产


### 实战

#### 简单示例
```
class BooksRepository{
    val userData = flow {
        emit(Book("1984"))
        emit(Book("茶馆"))
        emit(Book("一句顶一万句"))
        emit(Book("山月记"))
    }
}

class BookViewModel{
     val books: StateFlow<List<BookState>> = 
        booksRepository.userData
            .filter { it.author == "刘震云" }
            .stateIn(
                scope = viewModelScope,
                started = SharingStarted.WhileSubscribed(5_000),
                initialValue = BooksFeedUiState.Loading
            )
}

class BookListFragment : Fragment() {
    ...
    private fun initObserver() {
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.books.collect { adapter.submitList(it) }
            }
        }
    }
    ...
}

```

#### Flow实现数据拉取
```
class VideoDetailRepository {
    private val api by lazy { NetworkApi.getInstance().createApi(MovieApi::class.java) }

    fun getVideoSelectData(videoSelectRequest: VideoSelectRequest): Flow<Result<VideoSelectResponse>> {
        return api.getVideoSelect(videoSelectRequest.toStGetVideoSelectReq()).map {
            if (it.ret == 0) {
                Result.success(it.toVideoSelectResponse())
            } else {
                Result.failure(Exception(it.msg))
            }
        }.catch {
            emit(Result.failure(it))
        }
    }
}

class ViewModel{
    ...
    private val store by lazyStore(::videoSelectReducer, VideoSelectState())
    val state get() = store.state

    init {
        viewModelScope.launch(Dispatchers.IO) {
            loadTypeFlow.transform { loadType ->
                store.dispatch(UpdateLoadStateAndType(LoadState.LOADING, loadType))
                repository.getVideoSelectData(
                    createVideoSelectRequest(
                        _curVideoSelectModel,
                        loadType,
                        state.value.pageState.attachInfo
                    )
                ).let {
                    emitAll(it)
                }
            }.collect { result ->
                Logger.i(TAG, "curVideoSelectModel = $_curVideoSelectModel")
                result.onSuccess {
                    Logger.i(TAG, "load data success, size = ${it.videoInfo.size}")
                    store.dispatch(LoadDataSuccess(curVideoSelectStyle, it))
                }.onFailure { throwable ->
                    Logger.e(TAG, "getVideoSelectData", throwable)
                    store.dispatch(UpdateLoadState(LoadState.ERROR))
                }
            }
        }
    }
    ....
}

class VideoSelectFragment : Fragment(R.layout.fragment_video_select) {
    ...
    private fun initObserver() {
        viewModel.state.map { it.pageState.canLoadBackward }
            .distinctUntilChanged()
            .launchAndCollectIn(viewLifecycleOwner) {
                viewBinding.refreshLayout.setEnableLoadMore(it)
            }
        viewModel.state
            .map { it.videoItemStates }
            .distinctUntilChanged()
            .launchAndCollectIn(viewLifecycleOwner) { dataList ->
                pictureTextSelectAdapter.submitList(dataList)
            }
    }
    ...
}


inline fun <reified T> Flow<T>.launchAndCollectIn(
    owner: LifecycleOwner,
    minActiveState: Lifecycle.State = Lifecycle.State.STARTED,
    collector: FlowCollector<T>
) = owner.lifecycleScope.launch {
    owner.repeatOnLifecycle(minActiveState) {
        this@launchAndCollectIn.collect(collector)
    }
}
```

#### Flow版本Redux框架
[完整代码仓库](https://github.com/nullUfull/FlowRedux)

## 进阶

look forward to

## Q&A


## 参考
[nowinandroid](https://github.com/android/nowinandroid)

[实战 | 使用 Kotlin Flow 构建数据流 "管道"](https://juejin.cn/post/7078225994871472158)

[设计repeatOnLifecycle API 背后的故事](https://juejin.cn/post/7001371050202103838)

[repeatOnLifecycle API design story](https://medium.com/androiddevelopers/repeatonlifecycle-api-design-story-8670d1a7d333)

[A safer way to collect flows from Android UIs](https://medium.com/androiddevelopers/a-safer-way-to-collect-flows-from-android-uis-23080b1f8bda)

[Flow 操作符 shareIn 和 stateIn 使用须知](https://juejin.cn/post/6998066384290709518)

[Deep dive into Coroutine Flow 2](https://myungpyo.medium.com/deep-dive-into-coroutine-flow-2-d43ba0d1f45d)

[Migrating from LiveData to Kotlin’s Flow](https://medium.com/androiddevelopers/migrating-from-livedata-to-kotlins-flow-379292f419fb)
