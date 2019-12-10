> 原文作者： [Jose Alcérreca](https://medium.com/@JoseAlcerreca)
>
> 原文地址： [ViewModels and LiveData: Patterns + AntiPatterns](https://medium.com/androiddevelopers/viewmodels-and-livedata-patterns-antipatterns-21efaef74a54)
>
> 译者：秉心说

![Typical interaction of entities in an app built with Architecture Components](https://user-gold-cdn.xitu.io/2019/10/14/16dc9148b2c57daf?w=803&h=230&f=png&s=18933)

## View 和 ViewModel

### 分配责任

理想情况下，ViewModel 应该对 Android 世界一无所知。这提升了可测试性，内存泄漏安全性，并且便于模块化。
通常的做法是保证你的 ViewModel 中没有导入任何 `android.*`，`android.arch.*` (译者注：现在应该再加一个 `androidx.lifecycle`)除外。
这对 Presenter(MVP) 来说也一样。

> ❌   不要让 ViewModel  和 Presenter 接触到 Android 框架中的类

条件语句，循环和通用逻辑应该放在应用的 ViewModel 或者其它层来执行，而不是在 Activity 和 Fragment 中。
View 通常是不进行单元测试的，除非你使用了 [Robolectric](http://robolectric.org/)，所以其中的代码越少越好。
View 只需要知道如何展示数据以及向 ViewModel/Presenter 发送用户事件。这叫做 [Passive View](https://martinfowler.com/eaaDev/PassiveScreen.html) 模式。

> ✅  让 Activity/Fragment 中的逻辑尽量精简

### ViewModel 中的 View 引用

[ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel.html) 和 Activity/Fragment
具有不同的作用域。当 Viewmodel 进入 alive 状态且在运行时，activity 可能位于 [生命周期状态](https://developer.android.com/guide/components/activities/activity-lifecycle.html) 的任何状态。
Activitie 和 Fragment 可以在 ViewModel 无感知的情况下被销毁和重新创建。

![ViewModels persist configuration changes](https://user-gold-cdn.xitu.io/2019/10/14/16dc93660f91bef2?w=1280&h=960&f=png&s=46798)

向 ViewModel 传递 View(Activity/Fragment) 的引用是一个很大的冒险。假设 ViewModel 请求网络，稍后返回数据。
若此时 View 的引用已经被销毁，或者已经成为一个不可见的 Activity。这将导致内存泄漏，甚至 crash。

> ❌  避免在 ViewModel 中持有 View 的引用

在 ViewModel 和 View 中通信的建议方式是观察者模式，使用 LiveData 或者其他类库中的可观察对象。

### 观察者模式

![](https://user-gold-cdn.xitu.io/2019/10/14/16dc9478fef9f71d?w=412&h=103&f=png&s=7087)

在 Android 中设计表示层的一种非常方便的方法是让 View 观察和订阅 ViewModel（中的变化）。
由于 ViewModel 并不知道 Android 的任何东西，所以它也不知道 Android 是如何频繁的杀死 View 的。
这有如下好处：

1. ViewModel 在配置变化时保持不变，所以当设备旋转时不需要再重新请求资源（数据库或者网络）。
2. 当耗时任务执行结束，ViewModel 中的可观察数据更新了。这个数据是否被观察并不重要，尝试更新一个
	不存在的 View 并不会导致空指针异常。
3. ViewModel 不持有 View 的引用，降低了内存泄漏的风险。

```java
private void subscribeToModel() {
  // Observe product data
  viewModel.getObservableProduct().observe(this, new Observer<Product>() {
      @Override
      public void onChanged(@Nullable Product product) {
        mTitle.setText(product.title);
      }
  });
}
```

> ✅  让 UI 观察数据的变化，而不是把数据推送给 UI

## 胖 ViewModel

无论是什么让你选择分层，这总是一个好主意。如果你的 ViewModel 拥有大量的代码，承担了过多的责任，那么：

* 移除一部分逻辑到和 ViewModel 具有同样作用域的地方。这部分将和应用的其他部分进行通信并更新
             ViewModel 持有的 LiveData。
* 采用 [Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html),添加一个 domain 层。这是一个可测试，易维护的架构。[Architecture Blueprints](https://github.com/android/architecture-samples) 中有 Clean Architecture 的示例。

> ✅  分发责任，如果需要的话，添加 domain 层

### 使用数据仓库

如 [应用架构指南](https://developer.android.com/jetpack/docs/guide) 中所说，大部分 App 有多个数据源:

1. 远程：网络或者云端
2. 本地：数据库或者文件
3. 内存缓存

在你的应用中拥有一个数据层是一个好主意，它和你的视图层完全隔离。保持缓存和数据库与网络同步的算法并不简单。建议使用单独的 Repository 类作为处理这种复杂性的单一入口点.

如果你有多个不同的数据模型，考虑使用多个 Repository 仓库。

> ✅ 添加数据仓库作为你的数据的单一入口点。

## 处理数据状态

考虑下面这个场景：你正在观察 ViewModel 暴露出来的一个 LiveData，它包含了需要显示的列表项。那么 View 如何区分数据已经加载，网络错误和空集合？
* 你可以通过 ViewModel 暴露出一个 `LiveData<MyDataState>`，`MyDataState` 可以包含数据正在加载，已经加载完成，发生错误等信息。

* 你可以将数据包装在具有状态和其他元数据(如错误消息)的类中。查看示例中的 [Resource](https://developer.android.com/jetpack/docs/guide#addendum) 类。

> ✅ 使用包装类或者另一个 LiveData 来暴露数据的状态信息


## 保存 activity 状态

当 activity 被销毁或者进程被杀导致 activity 不可见时，重新创建屏幕所需要的信息被称为 activity 状态。屏幕旋转就是最明显的例子，如果状态保存在 ViewModel 中，它就是安全的。

但是，你可能需要在 ViewModel 也不存在的情况下恢复状态，例如当操作系统由于资源紧张杀掉你的进程时。

为了有效的保存和恢复 UI 状态，使用 `onSaveInstanceState()` 和 ViewModel 组合。

详见：[ViewModels: Persistence, onSaveInstanceState(), Restoring UI
State and Loaders](https://medium.com/google-developers/viewmodels-persistence-onsaveinstancestate-restoring-ui-state-and-loaders-fc7cc4a6c090) 。

## Event

Event 指只发生一次的事件。ViewModel 暴露出的是数据，那么 Event 呢？例如，导航事件或者展示 Snackbar 消息，都是应该只被执行一次的动作。

LiveData 保存和恢复数据，和 Event 的概念并不完全符合。看看具有下面字段的一个 ViewModel：

```java
LiveData<String> snackbarMessage = new MutableLiveData<>();
```

Activity 开始观察它，当 ViewModel 结束一个操作时需要更新它的值：

```java
snackbarMessage.setValue("Item saved!");
```

Activity 接收到了值并且显示了 SnackBar。显然就应该是这样的。

但是，如果用户旋转了手机，新的 Activity 被创建并且开始观察。当对 LiveData 的观察开始时，新的 Activity 会立即接收到旧的值，导致消息再次被显示。

与其使用架构组件的库或者扩展来解决这个问题，不如把它当做设计问题来看。**我们建议你把事件当做状态的一部分。**

> 把事件设计成状态的一部分。更多细节请阅读 [LiveData with SnackBar,Navigation and other events (the SingleLiveEvent case)](https://medium.com/google-developers/livedata-with-snackbar-navigation-and-other-events-the-singleliveevent-case-ac2622673150)

## ViewModel 的泄露

得益于方便的连接 UI 层和应用的其他层，响应式编程在 Android 中工作的很高效。LiveData 是这个模式的关键组件，你的 Activity 和 Fragment 都会观察 LiveData 实例。

LiveData 如何与其他组件通信取决于你，要注意内存泄露和边界情况。如下图所示，视图层（Presentation Layer）使用观察者模式，数据层（Data Layer）使用回调。

![Observer pattern in the UI and callbacks in the data layer](https://user-gold-cdn.xitu.io/2019/10/17/16dda3d4a9734d85?w=796&h=237&f=png&s=20874)

当用户退出应用时，View 不可见了，所以 ViewModel 不需要再被观察。如果数据仓库 Repository 是单例模式并且和应用同作用域，**那么直到应用进程被杀死，数据仓库 Repository 才会被销毁。** 只有当系统资源不足或者用户手动杀掉应用这才会发生。如果数据仓库 Repository 持有 ViewModel 的回调的引用，那么 ViewModel 将会发生内存泄露。

![The activity is nished but the ViewModel is still around](https://user-gold-cdn.xitu.io/2019/10/17/16dda4831386c5d5?w=809&h=229&f=png&s=19872)

如果 ViewModel 很轻量，或者保证操作很快就会结束，这种泄露也不是什么大问题。但是，事实并不总是这样。理想情况下，只要没有被 View 观察了，ViewModel 就应该被释放。

![](https://user-gold-cdn.xitu.io/2019/10/17/16dda4b8473e6423?w=798&h=230&f=png&s=21680)

你可以选择下面几种方式来达成目的：

* 通过 **ViewModel.onCLeared()** 通知数据仓库释放 ViewModel 的回调
* 在数据仓库 Repository 中使用 **弱引用** ，或者 **Event Bu**（两者都容易被误用，甚至被认为是有害的）。
* 通过在 View 和 ViewModel 中使用 LiveData 的方式，在数据仓库和 ViewModel 之间进程通信

> ✅  考虑边界情况，内存泄露和耗时任务会如何影响架构中的实例。
>
> ❌   不要在 ViewModel 中进行保存状态或者数据相关的核心逻辑。 ViewModel 中的每一次调用都可能是最后一次操作。

## 数据仓库中的 LiveData

为了避免 ViewModel 泄露和回调地狱，数据仓库应该被这样观察：


![](https://user-gold-cdn.xitu.io/2019/10/17/16dda58e78d0699c?w=890&h=234&f=png&s=24311)

当 ViewModel 被清除，或者 View 的生命周期结束，订阅也会被清除：


![](https://user-gold-cdn.xitu.io/2019/10/17/16dda59e14ddb6fd?w=798&h=230&f=png&s=21680)

如果你尝试这种方式的话会遇到一个问题：如果不访问 LifeCycleOwner 对象的话，如果通过 ViewModel 订阅数据仓库？使用 [Transformations](https://developer.android.com/topic/libraries/architecture/livedata#transform_livedata) 可以很方便的解决这个问题。`Transformations.switchMap` 可以让你根据一个 LiveData 实例的变化创建新的 LiveData。它还允许你通过调用链传递观察者的生命周期信息：

```java
LiveData<Repo> repo = Transformations.switchMap(repoIdLiveData, repoId -> {
        if (repoId.isEmpty()) {
            return AbsentLiveData.create();
        }
        return repository.loadRepo(repoId);
    }
);
```

在这个例子中，当触发更新时，这个函数被调用并且结果被分发到下游。如果一个 Activity 观察了 `repo`，那么同样的 LifecycleOwner 将被应用在 `repository.loadRepo(repoId)` 的调用上。

> 无论什么时候你在 [ViewModel](https://developer.android.com/reference/android/arch/lifecycle/ViewModel.html) 内部需要一个 [LifeCycle](https://developer.android.com/reference/android/arch/lifecycle/Lifecycle.html) 对象时，[Transformation](https://developer.android.com/topic/libraries/architecture/livedata#transform_livedata) 都是一个好方案。

## 继承 LiveData

在 ViewModel 中使用 LiveData 最常用的就是 `MutableLiveData`，并且将其作为 `LiveData` 暴露给外部，以保证对观察者不可变。

如果你需要更多功能，继承 LiveData 会让你知道活跃的观察者。这对你监听位置或者传感器服务很有用。

```java
public class MyLiveData extends LiveData<MyData> {

    public MyLiveData(Context context) {
        // Initialize service
    }

    @Override
    protected void onActive() {
        // Start listening
    }

    @Override
    protected void onInactive() {
        // Stop listening
    }
}
```

### 什么时候不要继承 LiveData

你也可以通过 `onActive()` 来开启服务加载数据。但是除非你有一个很好的理由来说明你不需要等待 LiveData 被观察。下面这些通用的设计模式：

* 给 `ViewModel` 添加 `start()` 方法，并尽快调用它。[见 [Blueprints example](https://github.com/android/architecture-samples/blob/dev-todo-mvvm-live/todoapp/app/src/main/java/com/example/android/architecture/blueprints/todoapp/addedittask/AddEditTaskFragment.java#L64)]
* 设置一个触发加载的属性 [见 [GithubBrowerExample](https://github.com/googlesamples/android-architecture-components/blob/master/GithubBrowserSample/app/src/main/java/com/android/example/github/ui/repo/RepoFragment.kt)]

> 你并不需要经常继承 LiveData 。让 Activity 和 Fragment 告诉 ViewModel 什么时候开始加载数据。


## 分割线

翻译就到这里了，其实这篇文章已经在我的收藏夹里躺了很久了。
最近 Google 重写了 [Plaid](https://github.com/android/plaid) 应用，用上了一系列最新技术栈， [AAC](https://developer.android.com/topic/libraries/architecture/)，MVVM, Kotlin，协程 等等。这也是我很喜欢的一套技术栈，之前基于此开源了 [Wanandroid](https://github.com/lulululbj/wanandroid) 应用 ，详见 [真香！Kotlin+MVVM+LiveData+协程 打造 Wanandroid！](https://juejin.im/post/5cb473e66fb9a068af37a6ce) 。

当时基于对 MVVM 的浅薄理解写了一套自认为是 MVVM 的 MVVM 架构，在阅读一些关于架构的文章，以及 Plaid 源码之后，发现了自己的 MVVM 的一些认知误区。后续会对 [Wanandroid](https://github.com/lulululbj/wanandroid) 应用进行合理改造，并结合上面译文中提到的知识点作一定的说明。欢迎 Star ！

> 文章首发微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解。
>
> 更多最新原创文章，扫码关注我吧！

![](https://user-gold-cdn.xitu.io/2019/4/27/16a5f352eab602c4?w=2800&h=800&f=jpeg&s=178470)
