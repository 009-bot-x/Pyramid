# 硬核讲解 Jetpack 之 LifeCycle 使用篇

大家好，我是 **LifeCycle** ，来自 Jetpack 生态链的最底端 。

我的作用是感知组件 (Activity/Fragment) 生命周期 ，并在合适的生命周期执行你分配给我的任务。我坚持贯彻 Jetpack 的 Slogan ，**Less Code ，less bug ！** 用上我，包你线下无崩溃，线上无 Bug，每天准时下班，走上人生巅峰，赢取白富美 ，自动省略 300 字 ......

好像有点跑题，拽回来。

虽然我自嘲来自 Jetpack 生态链最底端，但其实我是 Jetpack 大家族中最不可或缺的组件 。左边看看 LiveData ，右边看看 ViewModel ，它们露出一副不屑的表情，“没有你的日子里，大家不是一样过得好好的 ！”

说的好像有那么一点道理，我第一次出现还是在 Support 库年代的 **26.1.0** 版本。在这之前，大家是如何感知声明周期的呢？就在这时，一位发量依旧浓密的程序员从黑暗中甩出了他的祖传代码。

```java
public class LocationUtil {

    public void startLocation() {
        ......
    }

    public void stopLocation() {
        ......
    }

}
```

然后呢，你得这么用。

```java
public class LocationActivity extends AppCompatActivity {

    private LocationUtil locationUtil = new LocationUtil();

    @Override
   public void  onResume(){
       locationUtil.startLocation();
       ......
   }

   @Override
   public void onPause(){
       locationUtil.stopLocation();
       .....
   }
}
```

这位程序员吹嘘道，“我的 LocationUtil 久经考验，完美解决生命周期问题，绝不会内存泄漏 ！”

我不禁嗤之以鼻，你是单身久了一个人撸代码撸习惯了吧 ！没错，你一个人用是挺好的。但是对于一个现代化大型项目来说，假如有二十个页面需要使用你的 LocationUtil，这二十个页面又分配给了五个程序员来完成。你能保证所有的生命周期代码都如你所愿的被添加了吗？又或者你已经离职了，又没有留下详尽的文档，殊不知哪天就得因为莫名其妙的内存泄漏被新员工问候。

再来一个极端情况，该死的产品经理（甩锅）让你在分别在 `onCreate()` 和 `onDestroy` 中开启和关闭定位，而不是原来的 `onResume()` 和 `onPause()` 中了。这一刻，你是不是有种想掏出四十米大刀的感觉。

你这个写法，嘟噜嘟噜一大串代码，还这么脆弱 ！

他涨红了脸，只能无力的辩解，“你行你上，show me your code ！”

我早有准备，拿代码说话。首先让 LocationUtil 实现 **LifeCycleObserver**  接口。

```java
class LocationUtil( ) : LifeCycleObserver {
  @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
  fun startLocation( ){
    ......
  }

  @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
  fun stopLocation( ){
    ......
  }
}
```

然后对于任何实现了 **LifecycleOwner** 接口的生命周期组件，如果需要使用 LocationUtil 的话，只需要添加如下一行代码即可。

```java
lifecycle.addObserver(LocationUtil( ))
```

我让 LocationUtil 这种第三方组件在自己内部就可以拿到生命周期，这样原来需要在生命周期组件 Activity/Fragment 进行的生命周期逻辑处理，在组件内部就可以直接完成，进一步解耦。即使面对上面说到的变态产品经理的需求，你也只需要修改 LocationUtil 的内部实现，外部无需做任何修改。看嘛，很多时候自己觉得别人的需求很不合理，其实还是自己的问题。

四周鸦雀无声，应该都被我镇住了。这只是我的最基本用法，下面再来一些进阶的，注意，别眨眼。

## 监听应用前后台切换

一些银行 app 都有这样的功能，当切换应用到后台，会给用户一个提示，"xx 手机银行正在后台运行，请注意安全！"  。用我 LifeCycle 来实现的话，异常的简单。

```java
class KtxAppLifeObserver : LifecycleObserver {

    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    fun onForeground() {
        Ktx.app.toast("应用进入前台")
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
    fun onBackground() {
        Ktx.app.toast("应用进入后台")
    }
}
```

然后在应用启动时，如 ContentProvier，Application 中 ，调用如下代码：

```java
 ProcessLifecycleOwner.get().lifecycle.addObserver(KtxAppLifeObserver())
```

通过代码你也发现了是通过我的家族成员 `ProcessLifecycleOwner` 来实现的。它可以感知整个应用进程的生命周期，这样一来，监听应用前后台切换就轻而易举了。

## 全局管理 Activity

相信你们都做过这样的功能，特定情况下要 finish 掉打开过的所有 Activity，或者要关闭指定的 Activity 。通常的做法是在 BaseActivity 的生命周期回调中保存已经打开或关闭的 Activity 。虽然实现也还算优雅，但是如何给日益臃肿的 BaseActivity 减负呢？又到我 LifeCycle 出场了。

```java
class KtxLifeCycleCallBack : Application.ActivityLifecycleCallbacks {

    override fun onActivityCreated(activity: Activity, savedInstanceState: Bundle?) {
        KtxManager.pushActivity(activity)
        "onActivityCreated : ${activity.localClassName}".loge()
    }

    override fun onActivityStarted(activity: Activity) {
        "onActivityStarted : ${activity.localClassName}".loge()
    }

    override fun onActivityResumed(activity: Activity) {
        "onActivityResumed : ${activity.localClassName}".loge()
    }

    override fun onActivityPaused(activity: Activity) {
        "onActivityPaused : ${activity.localClassName}".loge()
    }

    override fun onActivityDestroyed(activity: Activity) {
        "onActivityDestroyed : ${activity.localClassName}".loge()shejiyuanze
        KtxManager.popActivity(activity)
    }

    override fun onActivitySaveInstanceState(activity: Activity, outState: Bundle?) {
    }

    override fun onActivityStopped(activity: Activity) {
        "onActivityStopped : ${activity.localClassName}".loge()
    }
}
```

通过我的另一个家族成员，` Application.ActivityLifecycleCallbacks` ，你可以监听到所有的 Activity 的所有生命周期。在 Application 中注册这个 Callback 就行了。

```java
application.registerActivityLifecycleCallbacks(KtxLifeCycleCallBack())
```

根据 **单一职责原则** ，一个类做一件事情，这种写法将 Activity 的管理职责集中到一个类中，降低代码耦合。而原来的写法有两个问题，第一，你必须得继承 BaseActivity 才能具备管理当前 Activity 的功能。这又涉及到合作开发，不想继承，或者忘记继承。第二，让 BaseActivity 承担了过多的职责，并不符合基本的设计原则。

## 自动处理生命周期的  Handler

来自 [程序亦非猿](https://github.com/AlanCheen/Pandora/blob/master/pandora-basic/src/main/java/me/yifeiyuan/pandora/LifecycleHandler.java) 的创意。在 onDestroy 方法里移除 Handler 的消息, 无需额外手动处理，避免内存泄露。

```java
public class LifecycleHandler extends Handler implements LifecycleObserver {

    private LifecycleOwner lifecycleOwner;

    public LifecycleHandler(final LifecycleOwner lifecycleOwner) {
        this.lifecycleOwner = lifecycleOwner;
        addObserver();
    }

    public LifecycleHandler(final Callback callback, final LifecycleOwner lifecycleOwner) {
        super(callback);
        this.lifecycleOwner = lifecycleOwner;
        addObserver();
    }

    public LifecycleHandler(final Looper looper, final LifecycleOwner lifecycleOwner) {
        super(looper);
        this.lifecycleOwner = lifecycleOwner;
        addObserver();
    }

    public LifecycleHandler(final Looper looper, final Callback callback, final LifecycleOwner lifecycleOwner) {
        super(looper, callback);
        this.lifecycleOwner = lifecycleOwner;
        addObserver();
    }

    private void addObserver() {
        notNull(lifecycleOwner);
        lifecycleOwner.getLifecycle().addObserver(this);
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
    private void onDestroy() {
        removeCallbacksAndMessages(null);
        lifecycleOwner.getLifecycle().removeObserver(this);
    }
}
```

## 更智能的事件总线

说到事件总线框架的发展史，推荐美团技术团队的一篇文章 [Android消息总线的演进之路：用LiveDataBus替代RxBus、EventBus](https://tech.meituan.com/2018/07/26/android-livedatabus.html) 。无论是 EventBus，还是 RxBus，其实都不具备生命周期感知能力，体现在代码上就是需要显示调用反注册方法。而 LiveDataBus 基于基于 LiveData 实现，LiveData 又通过 LifeCycle 获取了生命周期感知能力，无需手动反注册，无内存泄露风险。

```java
LiveEventBus
	.get("key_name", String.class)
	.observe(this, new Observer<String>() {
	    @Override
	    public void onChanged(@Nullable String s) {
	    }
	});
```
随时订阅，自动取消订阅，就是这么简单。

## 配合协程使用

协程作为 Android 异步处理方案的新贵，不仅仅是我 LifeCycle ，ViewModel，LIveData 等其他 Jetpack 组件都为其提供了特定的支持。

要使用 LifeCycle 的协程支持，需要添加 `androidx.lifecycle:lifecycle-runtime-ktx:2.2.0-alpha01`或更高版本。该支持库为每个 `LifeCycle` 对象定义了 `LifecycleScope` 。在此范围内启动的协程会在 LifeCycle 被销毁时自动取消。你可以通过 `lifecycle.coroutineScope` 或 `lifecycleOwner.lifecycleScope` 属性访问 Lifecycle 的 `CoroutineScope`。下面是一个简单的示例：

```java
class MyFragment: Fragment() {
        override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
            super.onViewCreated(view, savedInstanceState)
            viewLifecycleOwner.lifecycleScope.launch {
                val params = TextViewCompat.getTextMetricsParams(textView)
                val precomputedText = withContext(Dispatchers.Default) {
                    PrecomputedTextCompat.create(longTextContent, params)
                }
                TextViewCompat.setPrecomputedText(textView, precomputedText)
            }
        }
    }
```

更多用法可以查看官网相关介绍 [将 Kotlin 协程与架构组件一起使用](https://developer.android.google.cn/topic/libraries/architecture/coroutines#top_of_page) 。

## 最后

有了 LifeCycle ，你可以做什么？

**写更少的代码，犯更少的错 。**

生命周期的处理趋于标准化，这也正是 Jetpack 想带给我们的，让开发趋于标准化的同时可以犯更少的错误，减少崩溃和内存泄漏。浏览一下你的项目，如果还有未使用 LifeCycle 解决生命周期问题的地方，赶紧替换吧 ！

最后推荐一个我的 Jetpack MVVM 开源项目，[wanandroid](https://github.com/lulululbj/wanandroid) , 更多 bug ，等你发现。

有耐心看到这的读者，肯定要吐槽了，一点也不 **硬核** ！这也不能赖我，使用篇，怎么硬核嘛。

一切尽在 [硬核讲解 Jetpack 之 LifeCycle 源码篇]() ，敬请期待 ！

> 文章首发微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解。
>
> 更多最新原创文章，扫码关注我吧！

![](https://user-gold-cdn.xitu.io/2019/4/27/16a5f352eab602c4?w=2800&h=800&f=jpeg&s=178470)
