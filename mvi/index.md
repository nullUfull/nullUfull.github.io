# mvi


# 简介
MVI 与 MVVM 比较相似，其借鉴了前端框架的思想，着重强调数据的单向流动和唯一数据源

## 优势
1. 强调数据单向流动，很容易对状态变化进行跟踪和回溯
2. 使用ViewState对State集中管理，只需要订阅一个 ViewState 便可获取页面的所有状态，相对 MVVM 减少了不少模板代码
3. View通过ViewAction与ViewModel进行通信，又通过StateAction以触发State的变化

## 问题
1. 当页面复杂，State容易膨胀


## 最佳实践

### 以页面为维度定义State，并且注意State中数据的定义，切勿数据裸在State里面
例如一个按钮的状态可能对应State中的数个字段，可进行包裹为ButtonState放置在State。

当State复杂起来

### 

## 案例

### 收藏逻辑
我要收藏一个视频，UI发送一个事件到ViewModel，ViewModel调用Repository触发我们的收藏请求。
收藏请求回来需要展示一个Toast，提示用户收藏成功或者失败，这就以为着我们的ViewModel处理请求结果后，触发State变化，也就是我们CollectButtonSetate的变化。除此之外，还要想办法通知UI显示一条Toast。

如果存储一个字段用于标识UI需要显示的Toast，UI collect到该状态时候会显示该Toast，但如果切后台再返回，UI又会显示该Toast了，因为我们的State是StateFlow承载的，

### 新手引导
用户打开一个页面，需要一次显示2个引导浮窗。

如果是在MVI的架构下，做这个逻辑；判断是否显示新手引导的逻辑得再逻辑层，即ViewModel层；

## 使用误区
![状态定义](https://raw.githubusercontent.com/nullUfull/MyPicBed/main/example_1.1.png)
如图，初次使用MVI的同学在该页面的State中定义了2个字段来描述引导浮窗的状态

其中存在两个问题：
1. 使用2个字段来描述是丑陋的
2. 使用可空类型来描述引导流程中的状态更是丑陋的

最佳实践：定义一个枚举类描述引导流程中的状态
```
enum class GuideState{
    NOTHING,
    SHOW_TITLE_GUIDE,
    SHOW_DIRECT_ITEM_GUIDE
}
```

其次，我再看看UI层的逻辑
![触发显示引导](https://raw.githubusercontent.com/nullUfull/MyPicBed/main/example_1.2.png)

其实这个逻辑也是挺奇怪的，监听Title变化的地方，存在引导显示的逻辑；这就违背了我们的原则，View只需监听相对应的状态变化即可。

![触发显示引导](https://raw.githubusercontent.com/nullUfull/MyPicBed/main/example_1.3.png)

其次，再看看这个ViewAction的具体执行逻辑，如图，通过判断是否需要显示来更新引导的状态。
所以第二个疑惑的点就是，引导的状态其实并不依赖于其他的UI上面的东西，和用户的操作无关，正如与用户的操作无关却发送一个ViewAction是不太合适的，其实只需要在列表数据加载成功后，通过接口查询当前是否展示引导 去更新引导的状态即可。


OK，我们总结下如何在MVI架构添加业务逻辑：
1. 定义一个状态来描述这个元素
2. 在UI层监听这个状态的变化做相应的逻辑
3. 在合适的时机更改这个状态

## 规范
1. ViewAction定义：用户在UI层的输入，例如：点击、下拉刷新、上拉加载更多，ClickTitle、LoadMore、Refresh
2. StateAction定义：触发状态改变的操作，例如：修改按钮状态的可见性，则可定义为 UpdateXXBtnVisibility
3. State定义：以页面为单位描述该页面的UI状态
4. Store定义：以页面为单位存储当前页面的UI状态
5. 用户的输入事件向下流动，UI状态向上流动
6. 如果要dispatch一个Action，不要一个行为派发多个Action，一个行为对应一个Action；例如：如果我们定义了两个Action，依次dispatch，以完成一个操作；倘若后面某个Action的能力改变了，是否会影响到另外一个Action呢？

## 待解决问题
### 页面初始化逻辑
目前页面数据的加载的触发由名为LoadTypeFlow的流驱动，而目前在UI层主动向ViewModel dispatch一个ViewAction以触发LoadTypeFlow的改变以触发页面数据拉取

其实这种流向是不符合我们的规范的，用户只是打开了该页面并没有输入一个事件，实际上应该是可以做成在ViewModel初始化的时候去触发数据加载的。

# 参考
[Android官方-应用架构指南](https://developer.android.com/jetpack/guide?hl=zh-cn)
[Android官方-界面事件](https://developer.android.com/jetpack/guide/ui-layer/events?continue=https%3A%2F%2Fdeveloper.android.com%2Fcourses%2Fpathways%2Fandroid-architecture%3Frec%3DCjZodHRwczovL2RldmVsb3Blci5hbmRyb2lkLmNvbS9qZXRwYWNrL2d1aWRlL2RhdGEtbGF5ZXIQARgPIAEoBDALOgMzLjc#decision-tree)
