## 前言

第一次接触 `Kotlin` 还是 2017 年，当时 Kotlin 还没扶正，也不是 Android 的官方开发语言。至于我是怎么被安利的，没记错的话，应该是 [开源实验室](https://kymjs.com/code/2017/02/03/01/) 的 Kotlin 教程。当时身边几乎没有人在学 Kotlin，网上相关的资料也很少，我还翻译了一部分官网文档，写了一本 [GitBook](https://lulululbj.gitbooks.io/kotlin-document/) 。 当然现在有更好的 [Kotlin 语言中文站](https://www.kotlincn.net/) 了，对于英文基础不是很好的同学，这是一个不错的入门资料。

两年半时间过来了，Kotlin 摇身一变，稳坐 Android 官方开发语言。尽管可能是为了避免与 Oracle 的诉讼，但我相信这绝对不是主要原因。被定义为 `Better Java` 的 Kotlin，的确做到了更好，也逐渐为更多人所使用。

> [《全新 LeakCanary 2 ! 完全基于 Kotlin 重构升级 ！》](https://juejin.im/post/5d1225546fb9a07ecd3d6b71)
>
> [Retrofit 2.6.0 ! 更快捷的协程体验 ！](https://juejin.im/post/5d0793616fb9a07eac05d407)

越来越多的知名三方库都已经开始提供对 Kotlin 的支持。Android 官方的新特性也将优先支持 Kotlin。除了 Android 领域，`Spring 5` 正式发布时，也将 Kotlin 作为其主打的新特性之一。肉眼可见的，在 JVM 语言领域，Kotlin 必将有一番作为。

我在之前的一篇文章 [真香！Kotlin+MVVM+LiveData+协程 打造 Wanandroid！](https://juejin.im/post/5cb473e66fb9a068af37a6ce) 中开源了 Kotlin MVVM 版本的 [Wanandroid](https://github.com/lulululbj/wanandroid) 应用，到现在一共有 `138` 个 star 和 `20` 个 fork。数据并不是多亮眼，但算是我开源生涯中的一大步。这个项目我也一直在更新，欢迎大家持续关注。

目前的工作中，除非一些不可抗拒因素，我已经将 Kotlin 作为第一选择。在进行多个项目的开发工作之后，我发现我经常在各个项目之间拷贝代码，base 类，扩展函数，工具类等等。甚至有的时候会翻回以前的 Java 代码，或者 Blankj 的 [AndroidUtilCode](https://github.com/Blankj/AndroidUtilCode)，拿过来直接使用。但是很多时候直接生搬硬套 Java 工具类，并不是那么的优雅，也没有很好的运用 Kotlin 语言特性。我迫切需要一个通用的 Kotlin 工具类库。

基于此，[**AndroidUtilCodeKTX**](https://github.com/lulululbj/AndroidUtilCodeKTX) 诞生了。如果你用过 Blankj 的 `AndroidUtilCode`，它们的性质是一样的，但绝不是其简单的 Kotlin 翻译版本，而是更多的糅合了 Kotlin 特性，一切从 “简” ，代码越少越好。`Talk is easy，show me the code ！` 话不多说，下面就代码展示 [**AndroidUtilCodeKTX**](https://github.com/lulululbj/AndroidUtilCodeKTX) 中一些工具类的使用。

## AndroidUtilCodeKTX

### 权限请求

```kotlin
request(Manifest.permission.READ_CALENDAR, Manifest.permission.RECORD_AUDIO) {
    onGranted { toast("onGranted") }
    onDenied { toast("onDenied") }
    onShowRationale { showRationale(it) }
    onNeverAskAgain { goToAppInfoPage() }
}
```

借助扩展函数和 DSL ，可以轻松的在 Activity 中优雅的请求权限以及处理回调。这不是一个全新的轮子，主要代码来自 [PermissionsKt](https://github.com/sembozdemir/PermissionsKt) 。但是它有一个致命的缺点，需要开发者手动覆写 `onRequestPermissionsResult()` 来处理权限请求结果，这显得并不是那么简洁和优雅。我这里借鉴了 [RxPermissions](https://github.com/tbruyelle/RxPermissions/) 的处理方式，通过在当前 Activity 依附一个 Fragment 进行权限请求以及回调处理，这样对用户来说是无感的，且避免了额外的代码。后续会单独写一篇文章，分析其中 Kotlin DSL 的应用。

### SharedPreferences

```kotlin
putSpValue("int", 1)
putSpValue("float", 1f)
putSpValue("boolean", true)
putSpValue("string", "ktx")
putSpValue("serialize", Person("Man", 3))

getSpValue("int", 0)
getSpValue("float", 0f)
getSpValue(key = "boolean", default = false)
getSpValue("string", "null")
getSpValue("serialize", Person("default", 0))
```

基本的存储和读取操作都只需要一个函数即可完成，依赖泛型无需主动声明值的类型。默认存储文件名称为包名，如果你想自定义文件名称，声明 `name` 参数即可：

```kotlin
putSpValue("another","from another sp file",name = "another")
getSpValue("another","null",name = "another")
```

### Activity 相关

主要是优化了 `startActivity` 的使用，争取做到任何情况下都可以一句代码启动 Activity。

普通跳转：

```kotlin
startKtxActivity<AnotherActivity>()
```

带 flag 跳转：

```kotlin
startKtxActivity<AnotherActivity>(Intent.FLAG_ACTIVITY_NEW_TASK)
```

startActivityForResult ：

```kotlin
startKtxActivityForResult<AnotherActivity>(requestCode = 1024)
```

带值跳转：

```kotlin
startKtxActivity<AnotherActivity>(value = "string" to "single value")
```

带多个值跳转：

```kotlin
startKtxActivity<AnotherActivity>(
                values = arrayListOf(
                    "int" to 1,
                    "boolean" to true,
                    "string" to "multi value"
                )
            )
```

带 Bundle 跳转：

```kotlin
startKtxActivity<AnotherActivity>(
                extra = Bundle().apply {
                    putInt("int", 2)
                    putBoolean("boolean", true)
                    putString("string", "from bundle")
                }
            )
```

基本涵盖了所有的 Activity 跳转情况。

### Aes 加密相关

```kotlin
ByteArray.aesEncrypt(key: ByteArray, iv: ByteArray, cipherAlgotirhm: String = AES_CFB_NOPADDING): ByteArray
ByteArray.aesDecrypt(key: ByteArray, iv: ByteArray, cipherAlgotirhm: String = AES_CFB_NOPADDING): ByteArray
File.aesEncrypt(key: ByteArray, iv: ByteArray, destFilePath: String): File?
File.aesDecrypt(key: ByteArray, iv: ByteArray, destFilePath: String): File?
```

封装了 Aes 加密操作，提供了快捷的数据和文件加解密方法，无需关注内部细节。默认使用 `AES/CFB/NoPadding` 模式，你可以通过 `cipherAlgotirhm` 参数修改模式。

```kotlin
plainText.toByteArray().aesEncrypt(key, iv, "AES/CBC/PKCS5Padding"
plainText.toByteArray().aesEncrypt(key, iv, "AES/ECB/PKCS5Padding"
plainText.toByteArray().aesEncrypt(key, iv, "AES/CTR/PKCS5Padding"
```

### Hash 相关

给 `String` 和 `ByteArray` 提供了扩展方法，可以快捷的进行哈希。

```kotlin
ByteArray.hash(algorithm: Hash): String
String.hash(algorithm: Hash, charset: Charset = Charset.forName("utf-8")): String
ByteArray.md5Bytes(): ByteArray
ByteArray.md5(): String
String.md5(charset: Charset = Charset.forName("utf-8")): String
ByteArray.sha1Bytes(): ByteArray
ByteArray.sha1(): String
String.sha1(charset: Charset = Charset.forName("utf-8")): String
ByteArray.sha224Bytes(): ByteArray
ByteArray.sha224(): String
String.sha224(charset: Charset = Charset.forName("utf-8")): String
ByteArray.sha256Bytes(): ByteArray
ByteArray.sha256(): String
String.sha256(charset: Charset = Charset.forName("utf-8")): String
ByteArray.sha384Bytes(): ByteArray
ByteArray.sha384(): String
String.sha384(charset: Charset = Charset.forName("utf-8")): String
ByteArray.sha512Bytes(): ByteArray
ByteArray.sha512(): String
String.sha512(charset: Charset = Charset.forName("utf-8")): String
File.hash(algorithm: Hash = Hash.SHA1): String
```

支持 `MD5`, `sha1`, `sha224`, `sha256`, `sha384`, `sha512`。`MD5` 已经不再安全，密码学上来说已经不再推荐使用。

```kotlin
val origin = "hello"
val md5 = origin.hash(Hash.MD5)
val sha1 = origin.hash(Hash.SHA1)
```

除了字符串取哈希，还提供了对文件取哈希值操作。

```kotlin
val file = File("xxx")
val md5 = file.hash(Hash.MD5)
val sha1 = file.hash(Hash.SHA1)
```

### Intent 相关

跳转到应用信息界面：

```kotlin
Context.goToAppInfoPage(packageName: String = this.packageName)
```

跳转到日期和时间页面：

```kotlin
Context.goToDateAndTimePage()
```

跳转到语言设置页面：

```kotlin
Context.goToLanguagePage()
```

跳转到无障碍服务设置页面：

```kotlin
Context.goToAccessibilitySetting()
```

浏览器打开指定网页：

```kotlin
Context.openBrowser(url: String)
```

安装 apk ：

```kotlin
// need android.permission.REQUEST_INSTALL_PACKAGES after N
Context.installApk(apkFile: File)
```

在应用商店中打开应用:

```kotlin
Context.openInAppStore(packageName: String = this.packageName)
```

启动 App ：

```kotlin
Context.openApp(packageName: String)
```

卸载 App ：（好像并没有什么用）

```kotlin
Context.uninstallApp(packageName: String)
```

### Log 相关

```kotlin
fun String.logv(tag: String = TAG) = log(LEVEL.V, tag, this)
fun String.logd(tag: String = TAG) = log(LEVEL.D, tag, this)
fun String.logi(tag: String = TAG) = log(LEVEL.I, tag, this)
fun String.logw(tag: String = TAG) = log(LEVEL.W, tag, this)
fun String.loge(tag: String = TAG) = log(LEVEL.E, tag, this)

private fun log(level: LEVEL, tag: String, message: String) {
    when (level) {
        LEVEL.V -> Log.v(tag, message)
        LEVEL.D -> Log.d(tag, message)
        LEVEL.I -> Log.i(tag, message)
        LEVEL.W -> Log.w(tag, message)
        LEVEL.E -> Log.e(tag, message)
    }
}
```

`tag` 默认为 `ktx`，你也可以在参数中自己指定。

```kotlin
"abc".logv()
"def".loge(tag = "xxx")
```

### SystemService 相关

原来我们是这样获取系统服务的：

```kotlin
val powerManager = getSystemService(Context.POWER_SERVICE) as PowerManager
```

其实也挺简洁的，但是我们忘记了 `Context.WINDOW_SERVICE` 咋办？通过扩展属性可以让它更优雅。

```kotlin
val Context.powerManager get() = getSystemService<PowerManager>()

inline fun <reified T> Context.getSystemService(): T? =
    ContextCompat.getSystemService(this, T::class.java)
```

对于用户来说，直接使用 `powerManager` 即可，无需任何额外工作。

已作为扩展属性的系统服务如下：

```kotlin
val Context.windowManager
val Context.clipboardManager
val Context.layoutInflater
val Context.activityManager
val Context.powerManager
val Context.alarmManager
val Context.notificationManager
val Context.keyguardManager
val Context.locationManager
val Context.searchManager
val Context.storageManager
val Context.vibrator
val Context.connectivityManager
val Context.wifiManager
val Context.audioManager
val Context.mediaRouter
val Context.telephonyManager
val Context.sensorManager
val Context.subscriptionManager
val Context.carrierConfigManager
val Context.inputMethodManager
val Context.uiModeManager
val Context.downloadManager
val Context.batteryManager
val Context.jobScheduler
```

### App 相关

```kotlin
Context.versionName: String
Context.versionCode: Long
Context.getAppInfo(apkPath: String): AppInfo
Context.getAppInfos(apkFolderPath: String): List<AppInfo>
Context.getAppSignature(packageName: String = this.packageName): ByteArray?
```

这一块内容暂时还比较少，主要是整理了我在项目中遇到的一些常见需求。获取版本号，版本名称，根据 apk 文件获取应用信息，获取应用签名等等。App 相关的其实会有很多工具类，后续会慢慢补充。

### View 相关

```kotlin
View.visible()
View.invisible()
View.gone()
var View.isVisible: Boolean
var View.isInvisible: Boolean
var View.isGone: Boolean
View.setPadding(@Px size: Int)
View.postDelayed(delayInMillis: Long, crossinline action: () -> Unit): Runnable
View.toBitmap(config: Bitmap.Config = Bitmap.Config.ARGB_8888): Bitmap
```

也都是一些比较常用的函数。

### Common

```kotlin
Context.dp2px(dp: Float): Int
Context.px2dp(px: Float): Int
View.dp2px(dp: Float): Int
View.px2dp(px: Float): Int
Context.screenWidth
Context.screenHeight
Context.copyToClipboard(label: String, text: String)
```

### 其他

```kotlin
RecyclerView.itemPadding(top: Int, bottom: Int, left: Int = 0, right: Int = 0)
ByteArray.toHexString(): String
......
```

## 使用

```
implementation 'luyao.util.ktx:AndroidUtilKTX:0.0.5'
```

## 总结

以上都是我在项目中提炼出来的，也正因如此，现在的 [**AndroidUtilCodeKTX**](https://github.com/lulululbj/AndroidUtilCodeKTX) 还只是个孩子，做不到那么的尽善尽美，面面俱到。我会持续更新完善这个库，也希望大家可以多多提 [issue](https://github.com/lulululbj/AndroidUtilCodeKTX/issues/new) 和 [pr](https://github.com/lulululbj/AndroidUtilCodeKTX/pulls)，提供您宝贵的意见！

在这里也十分感谢 [Jie Smart](https://github.com/JieSmart) 昨天的邮件反馈，提供了一些修改意见，也让我更有动力去继续维护。另外，最新版本的 AndroidUtilCodeKTX 我都会第一时间用在我的开源项目 [wanandroid]([Wanandroid](https://github.com/lulululbj/wanandroid)) 上，欢迎大家 star ，fork ！


> 文章首发微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解。
>
> 更多相关知识，扫码关注我吧！

![](https://user-gold-cdn.xitu.io/2019/4/27/16a5f352eab602c4?w=2800&h=800&f=jpeg&s=178470)
