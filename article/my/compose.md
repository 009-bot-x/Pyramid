> 公众号回复 `Compose` 获取安装包
>
> 项目地址： [Wanandroid-Compose](https://github.com/lulululbj/Wanandroid-Compose)

经过前段时间的 Android Dev Summit ，相信你已经大概了解了 **Jetpack Compose** 。如果你还没有听说过，可以阅读这篇文章 [Jetpack Compose 最新进展](https://juejin.im/post/5db7933ae51d4529f07977e4) 。总而言之，Compose 是一个 **颠覆性** 的 **声明式 UI 框架** ，它的口号就是 **消灭 xml 文件 ！**

尽管 Jetpack Compose 还只是预览版，API 可能发生变化，缺乏足够的控件支持，甚至不是那么稳定，但这阻止不了我这颗好奇的心。我在第一时间就上手撸了一款 Compose 版本 Wanandroid 应用，功能也比较简单，仅仅包括首页，广告和最新项目，类似于 Android 原生页面的 `Viewpager + TabLayout` 。下面的 gif 展示了应用的基本页面：


![](https://user-gold-cdn.xitu.io/2019/11/17/16e79f0bbf88f340?w=480&h=800&f=gif&s=450425)

可以看出来页面并不是那么流畅，View 的复用应该是个问题，甚至我也没发现应该怎么做下拉刷新。那么，Compose 给我们带来了什么呢？在解答这个问题之前，我想先来说说 Android 应用架构问题。

## 荒芜年代  —— MVC

在我刚入行的时候，可以说是 Android 开发的黄金时代，也可以说是开发者的荒芜时代。一方面，毫不夸张的说，基本会写 xml 都能谋得一份工作。另一方面，对于开发者来说，远远没有现在的规范的开发架构。没记错的话，我当年的主力开发框架是 `xUtils 2` ，一个类库大包干，从 布局和 ID 绑定，网络请求，到图片展示，ORM 操作，一应俱全。当时的 布局和 ID 绑定还是运行时反射，而不是编译期注解。很长一段时间以来，Android 连一个官方的网络库都没有。

在架构方面，很多人都是一个 Activity 撸到死，我真的见过上千行zouguolai的 MainActivity 。并且觉得这就是 MVC 架构，实体类 Entity 就是 `Model` 层，`Activity/Fragment` 就是 `Controller` 层，布局文件就是 `View` 层。

但这当真是 `MVC` 吗？其实并不是。不管是 `MVC`， `MVP`，还是 `MVVM`，都应该遵循一个最起码的原则，**表现层和业务层分离** ，也就是 Android 官网给出的 [应用架构指南](https://developer.android.google.cn/jetpack/docs/guide) 中强调的 **分离关注点** 。`Activity/Fragment` 既要承担视图层的任务，展示和更新 UI，又要处理业务层逻辑，获取数据等。这并不符合架构设计的基本原则。

正确的 MVC 模式中，Model 层不仅包含实体类 Entity，更重要的作用是处理业务逻辑。View 层负责处理视图逻辑。而 Controller 就是 Model 和 View 之间的桥梁。桥怎么建，其实并没有标准，根据你自己的需求就可以了。


![](https://user-gold-cdn.xitu.io/2019/11/17/16e79f0f2eb84f96?w=601&h=365&f=png&s=10634)

引用一张阮一峰老师的图，大致是这么个意思，但是也不一定就完全都是单向依赖。

从荒芜时代走过来，MVC 总算有点分层的味道在里面了，分离了视图层和业务层。但是 View 层和 Model 层的依赖关系，造成代码耦合，终将导致 Activity 日益臃肿。那么有没有办法将 View 层和 Model 层彻底分离，做到视图层和模型层完全分离呢？ **MVP** 就应运而生了。

## 青铜年代 —— MVP

依旧是阮一峰老师的图片：


![](https://user-gold-cdn.xitu.io/2019/11/17/16e79f12cb0772b6?w=537&h=323&f=png&s=9849)

相较于 MVC ，MVP 用 `Presenter 层` 代替了 `Controller 层` ，且 `View 层` 和 `Model 层` 完全分离，依靠 Presenter 进行通信 。

想象一个获取用户信息的场景。`IView` 接口中定义了一系列视图层接口 ，View 层（Activity）实现 `IView` 接口中相应视图逻辑。 View 层通过持有的 Presenter 处理业务逻辑，即请求用户信息。一般情况下，Presenter 也不直接处理业务逻辑，而是通过 Model 层，例如数据仓库 Repository， 来获取数据，避免 Presenter 重蹈覆辙，日渐臃肿。同时，Presenter 层也是持有 VIew 的，获取用户信息之后再转发给 View 。

总结一下，MVP 中 View 和 Model 完全解耦，通过 Presenter 通信。View 和 Presenter 共同处理视图层逻辑，Model 层负责业务逻辑。

在 Github 上 Android 官方的架构示例 [architecture-samples](https://github.com/android/architecture-samples) 中 MVP 作为主分支坚挺了很久。我最初也是根据这个官方示例改造了自己的 MVP 架构，并且使用了很长时间。但是 MVP 作为一款面向接口编程的架构，随着业务的复杂程度不断加大，有种遍地都是接口的既视感，实在显得有点繁琐。

另外一点，Presenter 的职责边界不够清晰，它除了承担调用 Model 层获取业务逻辑之外，还要控制 View 层处理 UI。用下面一段代码表示一下：

```java
class LoginPresenter(private val mView: LoginContract.View) : LoginContract.Presenter {

    ......

    override fun login(userName: String, passWord: String) {
        CoroutineScope(Dispatchers.Main).launch {
            val result = WanRetrofitClient.service.login(userName, passWord).await()
            with(result) {
                if (errorCode == -1)  mView.loginError(errorMsg) else mView.login(data)
            }
        }
    }
}
```

一旦 View 层发生任何变化，Presenter 层也要做出相应改动。虽然 View 和 Model 之间解耦了，但是 View 和 Presenter 却耦合了。理想情况下，Presenter 层应该仅负责数据的获取，View 层自动观察数据的变化。于是，**MVVM** 来了。

## 黄金时代 —— MVVM

Google 官图镇楼 。


![](https://user-gold-cdn.xitu.io/2019/11/17/16e79f3a3c7e10db?w=960&h=720&f=png&s=29741)

MVP 风光早已不在， Android 官方的架构示例 [architecture-samples](https://github.com/android/architecture-samples) 的主分支已经切换到 MVVM 。在 Android 的 MVVM 架构中，**ViewModel** 是重中之重，它一方面通过数据仓库 Repository 获取数据，另一方面根据获取的数据更新 View 层的 Activity/Fragment。等等，这句话怎么听着这么耳熟，Presenter 不也是干了这些事吗？的确，它们干的事情都差不多，但是实现上完全不一样。

以我的开源项目 [Wanandroid](https://github.com/lulululbj/wanandroid) 中的 `LoginViewModel` 为例：

```kotlin
class LoginViewModel(val repository: LoginRepository) : BaseViewModel() {

    private val _uiState = MutableLiveData<LoginUiModel>()
    val uiState: LiveData<LoginUiModel>
        get() = _uiState


    fun loginDataChanged(userName: String, passWord: String) {
        emitUiState(enableLoginButton = isInputValid(userName, passWord))
    }

    // ViewModel 只处理视图逻辑，数据仓库 Repository 负责业务逻辑
    fun login(userName: String, passWord: String) {
        viewModelScope.launch(Dispatchers.Default) {
            if (userName.isBlank() || passWord.isBlank()) return@launch

            withContext(Dispatchers.Main) { showLoading() }

            val result = repository.login(userName, passWord)

            withContext(Dispatchers.Main) {
                if (result is Result.Success) {
                    emitUiState(showSuccess = result.data,enableLoginButton = true)
                } else if (result is Result.Error) {
                    emitUiState(showError = result.exception.message,enableLoginButton = true)
                }
            }
        }
    }

    private fun showLoading() {
        emitUiState(true)
    }

    private fun emitUiState(
            showProgress: Boolean = false,
            showError: String? = null,
            showSuccess: User? = null,
            enableLoginButton: Boolean = false,
            needLogin: Boolean = false
    ) {
        val uiModel = LoginUiModel(showProgress, showError, showSuccess, enableLoginButton,needLogin)
        _uiState.value = uiModel
    }

    data class LoginUiModel(
            val showProgress: Boolean,
            val showError: String?,
            val showSuccess: User?,
            val enableLoginButton: Boolean,
            val needLogin:Boolean
    )
}
```

可以看到，ViewModel 中是没有 View 的引用的，View 通过可观察的 LIveData 来观察数据变化，基于观察者模式做到和 ViewModel 完全解耦。

**数据驱动视图** ，这是 **Jetpack MVVM** 推崇的一个重要原则。其基本数据流如下所示 ：

* 数据层 Repository 负责从不同数据源获取和整合数据，基本负责所有的业务逻辑
* ViewModel 持有 Repository，获取数据并驱动 View 层更新
* View 持有 ViewModel，观察 LiveData 携带的数据，数据驱动 UI

曾经和一些开发者讨论过这样一个问题，** 不使用 DataBinding 还算是 MVVM 吗 ？**  我认为 MVVM 的核心从来不在于 DataBinding 。DataBinding 只是可以帮助我们将 **数据驱动视图** 做到极致，顺便还可以双向绑定。

要说到对 Jetpack MVVM 中最不满意的一块，那非 DataBinding 莫属了。在我狭隘的认为 DataBinding 就是一个在 xml 里面写逻辑代码的反人类的库时，我是坚决反对在任何项目中引入它的。固执己见的时候就容易走进误区，在阅读 KunminX 的 [重学安卓：从 被反对 到 真香 的 Jetpack DataBinding！](https://xiaozhuanlan.com/topic/9816742350) 之后，正如这篇文章名字一样，真香。

香的确是香，一切能让我早下班的都是好东西。在我的某次提交日志上，我写下了 **消灭 Adapter** 几个字，那时我刚用 DataBinding 消灭了大部分 RecyclerView 的 Adapter 。可是在提交之后，我的良心惴惴不安，我追究还是在 xml 文件里写逻辑代码了，难道这真的不反人类吗？

## 未来可期 —— Jetpack Compose

现在你应该可以理解我对 Jetpack Compose 的执念了。抛去其他特性，在我看来，它完美的解决了 **数据驱动视图** 的问题，我再也不需要使用 DataBinding 了。

简单代码展示一下 Compose 的用法。下面的代码描绘的是首页 Tab 下的文章列表。

```kotlin
@Composable
fun MainTab(articleUiModel: ArticleViewModel.ArticleUiModel?) {

    VerticalScroller {
        FlexColumn {
            inflexible {
                HeightSpacer(height = 16.dp)
            }
            flexible(1f) {
                articleUiModel?.showSuccess?.datas?.forEach {
                    ArticleItem(article = it)
                }

                articleUiModel?.showError?.let { toast(App.CONTEXT, it) }
wenjian
                articleUiModel?.showLoading?.let { Progress() }
            }
        }
    }
}
```

这种写法叫做 **声明式编程** ，会用 Flutter 的同学应该很熟悉。方法参数 `ArticleUiModel` 就是数据实体类，直接根据数据 ArticleUiModel 构建 UI 。说的大白话一点，就是给你长方形的长和宽了，让你画个长方形出来。最后加上 `@Compose` 注解，就是一个可用的 UI 组件了。仔细看代码，里面还用了两个 UI 组件 ，`ArticleItem` 和 `Progress` ，代码就不贴出来了。分别是文章列表的 item 项目 和加载进度条。

那么，数据如何更新呢？最简单的方式是使用 `@Model` 注解。

```kotlin
@Model
data class ArticleUiModel(){
  ......
}
```

对，就是这么简单。`@Model` 注解会自动把你的数据类变成可观察对象，只要 ArticleUIModel 发生变化，UI 就会自动更新。

但是我在实际开发中结合 LiveData 使用时，好像表现的不是那么正常。后来在 Medium 上无意中看到了解决方案，针对 LiveData 做了特殊处理 ：

```kotlinwenjian
// general purpose observe effect. this will likely be provided by LiveData. effect API for
// compose will also simplify soon.
fun <T> observe(data: LiveData<T>) = effectOf<T?> {
    val result = +state<T?> { data.value }
    val observer = +memo { Observer<T> { result.value = it } }

    +onCommit(data) {
        data.observeForever(observer)
        onDispose { data.removeObserver(observer) }
    }

    result.value
}wenjian
```

在 Activity/Fragment 中观测 LiveData 即可：
```kotlin
class MainActivity : BaseVMActivity<ArticleViewModel>() {

    override fun initView() {
        setContent {
           +observe(mViewModel.uiState)
            WanandroidApp(mViewModel)
        }
    }

    override fun initData() {
       mViewModel.getHomeArticleList()
    }
}
```

这样 View 层就可以自动观察 LiveData 所包含的值了。

没有 xml，没有 DataBinding，一切看起来称心如意多了。但就是 UI 体验有那么一点糟心，你可以在公众号后台回复 **Compose** 安装体验一下。由于还是早期的预览版，这也是可以理解的。我相信，等到发布 Release  版本的时候，一定足以完全代替原声的 View 体系。

本文并没有详细介绍 Jetpack Compose 的详细使用过程和其他特性，更多信息我推荐下面两篇文章：

* [Jetpack Compose 最新进展](https://juejin.im/post/5db7933ae51d4529f07977e4)
* [Jetpack Compose 新的尝试](https://juejin.im/post/5dbfc8f7f265da4d4434aca2)

## 最后       

正如 Android 官网 Jetpack 介绍页所说，Jetpack 可以帮助开发者更轻松的编写优质应用。的确，随着应用架构的规范，我们只需要把精力放在需要的代码上，加速开发，消除样板代码，减少崩溃和内存泄露，构建高质量的强大应用。我想不出来有任何理由不使用 Jetpack 来构建你的应用。而 Compose 必将称为 Jetpack 中极其重要的一块拼图。

**Jetpack Compse ，未来可期 ！**

***

添加我的微信，加入技术交流群。


![](https://user-gold-cdn.xitu.io/2019/11/17/16e79e67e6fb7c03?w=447&h=422&f=png&s=98388)

公众号后台回复 “compose”， 获取最新安装包。

![](https://user-gold-cdn.xitu.io/2019/11/13/16e656421741bfbc?w=2800&h=800&f=jpeg&s=178470)
