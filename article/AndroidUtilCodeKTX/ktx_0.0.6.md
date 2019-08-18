# AndroidUtilCodeKTX ！是时候提升你的开发效率了 ！（更新啦 ！）



[AndroidUtilCodeKTX](https://github.com/lulululbj/AndroidUtilCodeKTX) （以下简称 Ktx） 正式开源已经有一个月了。到目前为止，在 Github 上收获了 `98` 个 star 和 `11` 次 fork。期间上了一次 Github Trending Kotlin 分类的榜单，也收到了一些开发者的好评以及建议。经过这一个月的龟速更新，做了一些我想添加的功能，修复了一些开发者反馈的问题。

当前最新的版本是 `0.0.6` ：

```
implementation 'luyao.util.ktx:AndroidUtilKTX:0.0.6'
```

该版本的 Change log :

```
* 增加 log 开关
* 增加 TextView.notEmpty() 扩展方法
* 增加 ShellExt , 提供执行命令行相关函数
* BaseVMFragment、BaseVMActivity 中添加默认异常处理回调
* android 版本判断
* 判断无障碍服务是否开启
* RecyclerView.itemPadding
* 发送邮件
* 文件相关
* BaseActivity/BaseFragment 添加 CoroutineScope 实现
* 自动感知生命周期的 KtxHandler
* Activity 统一管理
* 应用前后台监听
* KtxSpan, 封装常见 Span 的使用
```

下面简单介绍一下 `0.0.6 版本`的主要修改内容。

## 一系列空判断

得益于高阶函数和 lambda 表达式的使用，我们可以充分发挥自己的想象力来封装一些通用逻辑。例如，对于一个可空对象，非空时执行指定操作，等于空时执行另一操作，我们的代码中可能充斥着大量这种结构的代码：

```kotlin
if (obj == null) {
    ...
} else {
    ...
}
```

看看 Ktx 中是如何书写这种逻辑的：

```kotlin
obj.notNull({
    ...    
}, {
    ...    
})
```

额，好像比原来优雅了那么一点（容许我欺骗一下自己）。实现也很简单，如下所示：

```kotlin
fun <T> Any?.notNull(f: () -> T, t: () -> T): T {
    return if (this != null) f() else t()
}
```

支持返回值。忽略上面那个不是那么明显的优化，顺着这个思路，我们可以在稍微复杂一点的情景下进行类似的优化。比如，`TextView` 中的文字是否为空，可以定义如下扩展函数 ：

```kotlin
fun TextView.notEmpty(f: TextView.() -> Unit, t: TextView.() -> Unit) {
    if (text.toString().isNotEmpty()) f() else t()
}
```

如果你想到了更多的使用场景，欢迎来砸 [issue](https://github.com/lulululbj/AndroidUtilCodeKTX/issues)。

## 执行 shell 命令

这个没啥特殊的地方，来自我的 [Box](https://github.com/lulululbj/Box) 项目中读取 linux 内核版本的需求。虽说最后也没读取成功，但也加了这么一个顶层函数：

```kotlin
fun executeCmd(command: String): String {
    val process = Runtime.getRuntime().exec(command)

    val resultReader = BufferedReader(InputStreamReader(process.inputStream))
    val errorReader = BufferedReader(InputStreamReader(process.errorStream))

    val resultBuilder = StringBuilder()
    var resultLine: String? = resultReader.readLine()

    val errorBuilder = StringBuilder()
    var errorLine = errorReader.readLine()

    while (resultLine != null) {
        resultBuilder.append(resultLine)
        resultLine = resultReader.readLine()
    }

    while (errorLine != null) {
        errorBuilder.append(errorLine)
        errorLine = errorReader.readLine()
    }

    return "$resultBuilder\n$errorBuilder"
}
```

## BaseActivity 添加异常处理

这个来自我的 [wanandroid](https://github.com/lulululbj/wanandroid) 项目的 [issue 7](https://github.com/lulululbj/wanandroid/issues/7)，主要是针对 Kotlin Coroutine 的异常处理。原来也有异常处理，只是 `BaseViewModel` 中用来接收异常的 `mException` 是私有的，无法直接获取。这个版本做了通用的异常处理，可以在 `BaseVMActivity` 和 `BaseVMFragment` 中直接通过 `onError()` 回调得到异常。具体可以参考 [MvvmActivity](https://github.com/lulululbj/AndroidUtilCodeKTX/blob/master/app/src/main/java/luyao/util/ktx/mvvm/MvvmActivity.kt) 中的异常处理演示。

## Android 版本判断

```kotlin
fun fromM() = fromSpecificVersion(Build.VERSION_CODES.M)
fun beforeM() = beforeSpecificVersion(Build.VERSION_CODES.M)
fun fromN() = fromSpecificVersion(Build.VERSION_CODES.N)
fun beforeN() = beforeSpecificVersion(Build.VERSION_CODES.N)
fun fromO() = fromSpecificVersion(Build.VERSION_CODES.O)
fun beforeO() = beforeSpecificVersion(Build.VERSION_CODES.O)
fun fromP() = fromSpecificVersion(Build.VERSION_CODES.P)
fun beforeP() = beforeSpecificVersion(Build.VERSION_CODES.P)
fun fromSpecificVersion(version: Int): Boolean = Build.VERSION.SDK_INT >= version
fun beforeSpecificVersion(version: Int): Boolean = Build.VERSION.SDK_INT < version
```

## 判断无障碍服务是否开启

工作中遇到的一个小需求，根据无障碍服务名称判断服务是否开启。

```kotlin
fun Context.checkAccessbilityServiceEnabled(serviceName: String): Boolean {
    val settingValue =
        Settings.Secure.getString(applicationContext.contentResolver, Settings.Secure.ENABLED_ACCESSIBILITY_SERVICES)
    return settingValue.notNull({
        var result = false
        val splitter = TextUtils.SimpleStringSplitter(':')
        while (splitter.hasNext()) {
            if (splitter.next().equals(serviceName, true)) {
                result = true
                break
            }
        }
        result
    }, { false })
}
```

## RecyclerView.itemPadding

```kotlin
fun RecyclerView.itemPadding(top: Int, bottom: Int, left: Int = 0, right: Int = 0) {
    addItemDecoration(PaddingItemDecoration(top, bottom, left, right))
}
```

方便快捷添加 itemPadding ，如下所示：

```kotlin
commonRecycleView.run {
    itemPadding(5, 5, 10, 10)
    layoutManager = LinearLayoutManager(this@CommonListActivity)
    adapter = commonAdapter
}
```

单位是 `dp`，内部处理了 `dp2px` 的逻辑。

## 发送邮件

`IntentExt` 中添加了发送邮件的函数，支持最基本的 `subject` 和 `text`。

```kotlin
fun Context.sendEmail(email: String, subject: String?, text: String?) {
    Intent(Intent.ACTION_SENDTO, Uri.parse("mailto:$email")).run {
        subject?.let { putExtra(Intent.EXTRA_SUBJECT, subject) }
        text?.let { putExtra(Intent.EXTRA_TEXT, text) }
        startActivity(this)
    }
}
```

## 文件相关

文件相关的部分，本来是准备花大把功夫来整合的，可是 Kotlin 标准库关于文件操作的支持实在是太完善了。我有对照 AndroidUtilCode 中的文件工具类，标准库基本就可以满足其中的大部分功能。为了能更全面的封装文件管理相关的工具类，我在 [Box](https://github.com/lulululbj/Box) 项目中添加了简单的文件管理功能，具体可见最新 commit。


![](https://user-gold-cdn.xitu.io/2019/8/17/16c9eebb8dbedf61?w=359&h=599&f=png&s=30505)

通过这个简单的文件管理器模块，在标准库的基础上，主要添加了以下扩展属性和函数。

```kotlin
val File.canListFiles: Boolean ：是否可以获取子文件夹
val File.totalSize: Long       ：总大小，包括所有子文件夹
val File.formatSize: String    ：格式化文件总大小，包括所有子文件夹
val File.mimeType: String      ：获取文件 mimeType

/**
 * [isRecursive] 是否获取所有子文件夹
 * [filter] 文件过滤器
 * /
fun File.listFiles(isRecursive: Boolean = false, filter: ((file: File) -> Boolean)? = null): Array<out File> {}

/**
 * [append] 是否追加
 * [text] 要 write 的内容
 * [charset] 字符编码
 * /
fun File.writeText(append: Boolean = false, text: String, charset: Charset = Charsets.UTF_8)
fun File.writeBytes(append: Boolean = false, bytes: ByteArray)

/**
 *   [destFile]  目标文件/文件夹
 *   [overwrite] 是否覆盖
 *   [reserve] 是否保留源文件
 */
fun File.moveTo(destFile: File, overwrite: Boolean = true, reserve: Boolean = true)

/**
 *   [destFolder] 目标文件/文件夹
 *   [overwrite] 是否覆盖
 *   [func] 单个文件的进度回调 (from 0 to 100)
 */
fun File.moveToWithProgress(
    destFolder: File,
    overwrite: Boolean = true,
    reserve: Boolean = true,
    func: ((file: File, i: Int) -> Unit)? = null
)

fun File.rename(newName: String)
fun File.rename(newFile: File)
```

除此之外的一些常见操作大多已经包含在 Kotlin Stdlib 中，读者可以阅读一下 `/kotlin/io/Utils.kt` 文件。

## CoroutineScope

在 `BaseActivity/BaseFragment` 中添加了 `CoroutineScope` 实现，其子类中可以直接通过 `launch {}` 使用主线程的协程作用域，并在 `onDestroy()` 中会自动取消在该作用域中启动的协程。有过协程使用经验的同学应该不难理解，实现也很简单。

```kotlin
abstract class BaseActivity : AppCompatActivity(), CoroutineScope by MainScope() {

    ...
    ...
    
    override fun onDestroy() {
        super.onDestroy()
        cancel()
    }
}
```

没使用过协程的也没有关系，后续我会写一篇文章简单介绍如何正确的在 Android 上使用协程。

## 自动感知生命周期的 KtxHandler

这是一个自动感知组件生命周期的 Handler，给它注册一个 `LifecycleOwner`，它就能在组件 `onDestroy()` 时自动清理资源。这个想法来自我在网上看到的代码，恕我实在想不起来出处了，下次会提前记录出处并标注出来。

```kotlin
class KtxHandler(lifecycleOwner: LifecycleOwner, callback: Callback) : Handler(callback), LifecycleObserver {

    private val mLifecycleOwner: LifecycleOwner = lifecycleOwner

    init {
        lifecycleOwner.lifecycle.addObserver(this)
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
    private fun onDestroy() {
        removeCallbacksAndMessages(null)
        mLifecycleOwner.lifecycle.removeObserver(this)
    }
}
```

其简单使用可以参见 [LifeCycleActivity.kt](https://github.com/lulululbj/AndroidUtilCodeKTX/blob/master/app/src/main/java/luyao/util/ktx/ui/LifeCycleActivity.kt) 。

## Activity 统一管理/应用前后台监听

这个来源于 issue 区的需求。开发中的需求无非两个，关闭指定的 Activity，退出应用时关闭所有 Activity ，这两个函数定义在了单例类 `KtxManager` 中。

```kotlin
fun  finishActivity(clazz: Class<*>){
    for (activity in mActivityList)
        if (activity.javaClass == clazz)
            activity.finish()
}

fun finishAllActivity() {
    for (activity in mActivityList)
        activity.finish()
}
```

同样在 [LifeCycleActivity.kt](https://github.com/lulululbj/AndroidUtilCodeKTX/blob/master/app/src/main/java/luyao/util/ktx/ui/LifeCycleActivity.kt) 中有这两个函数的使用例子。

这块的使用还是比较巧妙的。通过 `ActivityLifecycleCallbacks` 来监听 Activity 的生命周期，而 ActivityLifecycleCallbacks 的一般使用方式就是在 `Application` 中进行注册。但是在 Ktx 中，并没有 Application 类，也无需开发者在集成时书写任何代码。它自动完成了生命周期的注册监听。可能有的同学在其他地方也见识过这个小技巧，如果不知道的话，阅读一下 [Ktx.kt](https://github.com/lulululbj/AndroidUtilCodeKTX/blob/master/ktx/src/main/java/luyao/util/ktx/Ktx.kt)，相信你就明白了。

顺带也通过 `ProcessLifecycleOwner` 监听了应用进入前台和后台，相信你也看到了对应的 Toast 提示。

## KtxSpan

最后是一个 Span 工具类，先看看效果。


![](https://user-gold-cdn.xitu.io/2019/8/17/16c9f1c83d831060?w=2048&h=1534&f=jpeg&s=296756)

对这个图很熟悉的话，说明你是 Blankji 的 [AndroidUtilCode](https://github.com/Blankj/AndroidUtilCode/) 的忠实用户。这里我没有再重新造轮子了，当然也不是完全照抄 AndroidUtilCode 。我一直是 [material-dialogs](https://github.com/afollestad/material-dialogs/) 的使用者，非常喜欢它的 API 形式，所以仿照它的 API 结构重构了 `KtxSpan`。

```kotlin
KtxSpan().with(spanTv).show {
            text(
                "SpanUtils",
                foregroundColor = Color.YELLOW,
                backgroundColor = Color.LTGRAY,
                alignment = Layout.Alignment.ALIGN_CENTER
            )
            text("前景色", foregroundColor = Color.GREEN)
            text("背景色", backgroundColor = Color.LTGRAY)
            blockLine(spanTv.dp2px(6), true)
            text("行高", lineHeight = 2 * spanTv.lineHeight, backgroundColor = Color.LTGRAY)
            text(
                "测试段落缩进，首行缩进两字，其他行不缩进，其他行不缩进",
                first = (spanTv.textSize * 2).toInt(),
                rest = 0,
                backgroundColor = Color.GREEN
            )
            text(
                "测试引用，后面的字是为了凑到两行的效果",
                quoteColor = Color.GREEN,
                quoteStripeWidth = 10,
                quoteGapWidth = 10,
                backgroundColor = Color.LTGRAY
            )
            image(
                R.drawable.small,
                verticalAlignment = ImageSpan.ALIGN_BOTTOM,
                marginLeft = dp2px(30),
                marginRight = dp2px(40)
            )
}
```

KtxSpan 只有三个方法，但已经足以覆盖大部分使用场景。

第一个方法 `text()` ，可以表示文字的大部分效果。得益于 Kotlin 的方法参数支持默认值的特性，消灭了大部分方法重载的情况。text() 方法的参数也相当之多，共有 `31` 个参数，除了 `text` 为必须的之外，其余均可以根据需求赋值。默认每个 `text()` 方法都是新的一行，你也可以传入 `isNewLine: Boolean = false` 来使得下一行不再换行。

第二个方法是 `image()` ，可以表示图片显示，支持属性有对齐方式和左右间距。

第三个方法是 `blockLine()`，表示段落间距。有两个参数，一个是段落间距的值，一个是是否为以后的段落都添加间距。

关于 KtxSpan 的具体使用，可见项目中的示例代码 :  [KtxSpanActivity](https://github.com/lulululbj/AndroidUtilCodeKTX/blob/master/app/src/main/java/luyao/util/ktx/ui/KtxSpanActivity.kt) 。

## Last

以上就是这次更新的全部内容了，**AndroidUtilCodeKTX** 还是个孩子，欢迎来撩 ！

> 文章首发微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解。
>
> AndroidUtilCodeKTX 最新更新信息，扫码关注我吧！

![](https://user-gold-cdn.xitu.io/2019/4/27/16a5f352eab602c4?w=2800&h=800&f=jpeg&s=178470)