## 前言

你还记得是哪一年的 Google IO 正式宣布 `Kotlin` 成为 Android 一级开发语言吗？是 `Google IO 2017` 。如今两年时间过去了，站在一名 Android 开发者的角度来看，Kotlin 的生态环境越来越好了，相关的开源项目和学习资料也日渐丰富，身边愿意去使用或者试用 Kotlin 的朋友也变多了。常年混迹掘金的我也能明显感觉到 Kotlin 标签下的文章慢慢变多了（其实仍然少的可怜）。今年的 Google IO 也放出了 `Kotlin First` 的口号，许多新的 API 和功能特性将优先提供 Kotlin 支持。所以，时至今日，实在找不到安卓开发者不学 Kotlin 的理由了。

今天想聊聊的是 `Kotlin Coroutine`。虽然在 Kotlin 发布之初就有了协程，但是直到 2018 年的 **KotlinConf** 大会上，JetBrain 发布了 Kotlin1.3RC，这才带来了稳定版的协程。即使稳定版的协程已经发布了一年之余，但是好像并没有足够多的用户，至少在我看来是这样。在我学习协程的各个阶段中，遇到问题都鲜有地方可以求助，抛到技术群基本就石沉大海了。基本只能靠一些英文文档来解决问题。

关于协程的文章我看过很多，总结一下，无非下面几类。

第一类是 Medium 上热门文章的翻译，其实我也翻译过：

> [在 Android 上使用协程（一）：Getting The Background](https://juejin.im/post/5cea3ee0f265da1bca51b841)
>
> [在 Android 上使用协程（二）：Getting started](https://juejin.im/post/5cee800051882544171c5a2c)
>
> [在 Android 上使用协程（三） ：Real Work](https://juejin.im/post/5cf513d3e51d4577407b1ceb)

说实话，这三篇文章的确加深了我对协程的理解。

第二类就是官方文档的翻译了，我看过至少不下于五个翻译版本，还是觉得看 [官网文档](https://kotlinlang.org/docs/reference/coroutines/coroutines-guide.html) 比较好，如果英文看着实在吃力，可以对照着 Kotlin 中文站的翻译来阅读。

在看完官方文档的很长一段时间，我几乎只知道 `GlobalScope`。的确，官方文档上基本从头到尾都是在用 GlobalScope 写示例代码。所以一部分开发者，也包括我自己，在写自己的代码时也就直接 GlobalScope 了。一次偶然的机会才发现其实这样的问题是很大的。在 Android 中，一般是不建议直接使用 GlobalScope 的。那么，在 Android 中应该如何正确使用协程呢？再细分一点，如何直接在 Activity 中使用呢？如何配合 ViewModel 、LiveData 、LifeCycle 等使用呢？我会通过简单的示例代码来阐述 Android 上的协程使用，你也可以跟着动手敲一敲。

## 协程在 Android 上的使用

### GlobalScope
在一般的应用场景下，我们都希望可以异步进行耗时任务，比如网络请求，数据处理等等。当我们离开当前页面的时候，也希望可以取消正在进行的异步任务。这两点，也正是使用协程中所需要注意的。既然不建议直接使用 `GlobalScope`，我们就先试验一下使用它会是什么效果。

```kotlin
private fun launchFromGlobalScope() {
    GlobalScope.launch(Dispatchers.Main) {
        val deferred = async(Dispatchers.IO) {
            // network request
            delay(3000)
            "Get it"
        }
        globalScope.text = deferred.await()
        Toast.makeText(applicationContext, "GlobalScope", Toast.LENGTH_SHORT).show()
    }
}
```

在 `launchFromGlobalScope()` 方法中，我直接通过 `GlobalScope.launch()` 启动一个协程，`delay(3000)` 模拟网络请求，三秒后，会弹出一个 Toast 提示。使用上是没有任何问题的，可以正常的弹出 Toast 。但是当你执行这个方法之后，立即按返回键返回上一页面，仍然会弹出 Toast 。如果是实际开发中通过网络请求更新页面的话，当用户已经不在这个页面了，就根本没有必要再去请求了，只会浪费资源。GlobalScope 显然并不符合这一特性。[Kotlin 文档](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-global-scope/index.html) 中其实也详细说明了，如下所示:

> Global scope is used to launch top-level coroutines which are operating on the whole application lifetime and are not cancelled prematurely. Another use of the global scope is operators running in Dispatchers.Unconfined, which don’t have any job associated with them.
>
> Application code usually should use an application-defined CoroutineScope. Using async or launch on the instance of GlobalScope is highly discouraged.

大致意思是，Global scope 通常用于启动顶级协程，这些协程在整个应用程序生命周期内运行，不会被过早地被取消。程序代码通常应该使用自定义的协程作用域。直接使用 GlobalScope 的 async 或者 launch 方法是强烈不建议的。

GlobalScope 创建的协程没有父协程，GlobalScope 通常也不与任何生命周期组件绑定。除非手动管理，否则很难满足我们实际开发中的需求。所以，GlobalScope 能不用就尽量不用。

### MainScope

官方文档中提到要使用自定义的协程作用域，当然，Kotlin 已经给我们提供了合适的协程作用域 `MainScope` 。看一下 MainScope 的定义：

```kotlin
public fun MainScope(): CoroutineScope = ContextScope(SupervisorJob() + Dispatchers.Main)
```

记着这个定义，在后面 ViewModel 的协程使用中也会借鉴这种写法。

给我们的 Activity 实现自己的协程作用域：

```kotlin
class BasicCorotineActivity : AppCompatActivity(), CoroutineScope by MainScope() {}
```

通过扩展函数 `launch()` 可以直接在主线程中启动协程，示例代码如下：

```kotlin
private fun launchFromMainScope() {
    launch {
        val deferred = async(Dispatchers.IO) {
            // network request
            delay(3000)
            "Get it"
        }
        mainScope.text = deferred.await()
        Toast.makeText(applicationContext, "MainScope", Toast.LENGTH_SHORT).show()
    }
}
```

最后别忘了在 `onDestroy()` 中取消协程，通过扩展函数 `cancel()` 来实现：

```kotlin
override fun onDestroy() {
    super.onDestroy()
    cancel()
}
```

现在来测试一下 `launchFromMainScope()` 方法吧！你会发现这完全符合你的需求。实际开发中可以把 MainScope 整合到 BaseActivity 中，就不需要重复书写模板代码了。

### ViewModelScope

如果你使用了 MVVM 架构，根本就不会在 Activity 上书写任何逻辑代码，更别说启动协程了。这个时候大部分工作就要交给 ViewModel 了。那么如何在 ViewModel 中定义协程作用域呢？还记得上面 `MainScope()` 的定义吗？没错，搬过来直接使用就可以了。

```kotlin
class ViewModelOne : ViewModel() {

    private val viewModelJob = SupervisorJob()
    private val uiScope = CoroutineScope(Dispatchers.Main + viewModelJob)

    val mMessage: MutableLiveData<String> = MutableLiveData()

    fun getMessage(message: String) {
        uiScope.launch {
            val deferred = async(Dispatchers.IO) {
                delay(2000)
                "post $message"
            }
            mMessage.value = deferred.await()
        }
    }

    override fun onCleared() {
        super.onCleared()
        viewModelJob.cancel()
    }
}
```

这里的 `uiScope` 其实就等同于 `MainScope`。调用 `getMessage()` 方法和之前的 `launchFromMainScope()` 效果也是一样的，记得在 ViewModel 的 `onCleared()` 回调里取消协程。

你可以定义一个 `BaseViewModel` 来处理这些逻辑，避免重复书写模板代码。然而 Kotlin 就是要让你做同样的事，写更少的代码，于是 `viewmodel-ktx` 来了。看到 ktx ，你就应该明白它是来简化你的代码的。引入如下依赖：

```
implementation "androidx.lifecycle:lifecycle-viewmodel-ktx:2.2.0-alpha03"
```

然后，什么都不需要做，直接使用协程作用域 `viewModelScope` 就可以了。`viewModelScope` 是 ViewModel 的一个扩展属性，定义如下：

```kotlin
val ViewModel.viewModelScope: CoroutineScope
        get() {
            val scope: CoroutineScope? = this.getTag(JOB_KEY)
            if (scope != null) {
                return scope
            }
            return setTagIfAbsent(JOB_KEY,
                CloseableCoroutineScope(SupervisorJob() + Dispatchers.Main))
        }
```

看下代码你就应该明白了，还是熟悉的那一套。当 `ViewModel.onCleared()` 被调用的时候，`viewModelScope` 会自动取消作用域内的所有协程。使用示例如下：

```kotlin
fun getMessageByViewModel() {
    viewModelScope.launch {
        val deferred = async(Dispatchers.IO) { getMessage("ViewModel Ktx") }
        mMessage.value = deferred.await()
    }
}
```

写到这里，`viewModelScope` 是能满足需求的最简写法了。实际上，写完全篇，`viewModelScope` 仍然是我认为的最好的选择。

### LiveData

Kotlin 同样为 LiveData 赋予了直接使用协程的能力。添加如下依赖：

```
implementation "androidx.lifecycle:lifecycle-livedata-ktx:2.2.0-alpha03"
```

直接在 `liveData {}` 代码块中调用需要异步执行的挂起函数，并调用 `emit()` 函数发送处理结果。示例代码如下所示：

```kotlin
val mResult: LiveData<String> = liveData {
    val string = getMessage("LiveData Ktx")
    emit(string)
}
```

你可能会好奇这里好像并没有任何的显示调用，那么，liveData 代码块是在什么执行的呢？当 LiveData 进入 `active` 状态时，`liveData{ }` 会自动执行。当 LiveData 进入 `inactive` 状态时，经过一个可配置的 timeout 之后会自动取消。如果它在完成之前就取消了，当 LiveData 再次 `active` 的时候会重新运行。如果上一次运行成功结束了，就不会再重新运行。也就是说只有自动取消的 `liveData{ }` 可以重新运行。其他原因（比如 `CancelationException`）导致的取消也不会重新运行。

所以 `livedata-ktx` 的使用是有一定限制的。对于需要用户主动刷新的场景，就无法满足了。**在一次完整的生命周期内，一旦成功执行完成一次，就没有办法再触发了。** 这句话不知道对不对，我个人是这么理解的。因此，还是 `viewmodel-ktx` 的适用性更广，可控性也更好。


### LifecycleScope

```
implementation "androidx.lifecycle:lifecycle-runtime-ktx:2.2.0-alpha03"
```

`lifecycle-runtime-ktx` 给每个 `LifeCycle` 对象通过扩展属性定义了协程作用域 `lifecycleScope` 。你可以通过 `lifecycle.coroutineScope` 或者 `lifecycleOwner.lifecycleScope` 进行访问。示例代码如下：

```kotlin
fun getMessageByLifeCycle(lifecycleOwner: LifecycleOwner) {
    lifecycleOwner.lifecycleScope.launch {
        val deferred = async(Dispatchers.IO) { getMessage("LifeCycle Ktx") }
        mMessage.value = deferred.await()
    }
}
```

当 LifeCycle 回调 `onDestroy()` 时，协程作用域 `lifecycleScope` 会自动取消。在 `Activity/Fragment` 等生命周期组件中我们可以很方便的使用，但是在 MVVM 中又不会过多的在 View 层进行逻辑处理，viewModelScope 基本就可以满足 ViewModel 中的需求了，`lifecycleScope` 也显得有点那么食之无味。但是他有一个特殊的用法：

```kotlin
suspend fun <T> Lifecycle.whenCreated()
suspend fun <T> Lifecycle.whenStarted()
suspend fun <T> Lifecycle.whenResumed()
suspend fun <T> LifecycleOwner.whenCreated()
suspend fun <T> LifecycleOwner.whenStarted()
suspend fun <T> LifecycleOwner.whenResumed()
```

可以指定至少在特定的生命周期之后再执行挂起函数，可以进一步减轻 View 层的负担。

## 总结

以上简单的介绍了在 Android 中合理使用协程的一些方案，示例代码已上传至 [Github](https://github.com/lulululbj/CoroutineDemo)。关于 `MVVM + 协程` 的实战项目，可以看看我的开源项目 [wanandroid](https://github.com/lulululbj/wanandroid)，同时也期待你宝贵的意见。


> 文章首发微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解。
>
> 更多最新原创文章，扫码关注我吧！

![](https://user-gold-cdn.xitu.io/2019/4/27/16a5f352eab602c4?w=2800&h=800&f=jpeg&s=178470)
