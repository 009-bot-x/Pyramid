近日 [Retrofit](https://github.com/square/retrofit) 更新到了 [2.6.0](https://github.com/square/retrofit/releases/tag/parent-2.6.0) 版本，内置了对 Kotlin Coroutines 的支持，进一步简化了使用 Retrofit 和协程来进行网络请求的过程。其实纵观编程语言的发展历史，从汇编到 C/C++，从 Java，OC 到 Swift，Kotlin，甚至被纳入教材的 Python，都有一个共同的特点。随着 CPU 性能的越来越强悍，提高生产力似乎都成了现代高级编程语言的共同目标。Kotlin 就是一个好例子，做同样的事情，完成同样的功能，Java 的确需要更多的代码，Kotlin 也的确给 Android 开发提升了效率。特别是在异步任务方面，Kotlin 提供了协程，而这是 Java 所不具备的。关于 Kotlin Coroutines 的介绍，可以阅读我之前的三篇译文：

> [在 Android 上使用协程（一）：Getting The Background](https://juejin.im/post/5cea3ee0f265da1bca51b841)
>
> [在 Android 上使用协程（二）：Getting started](https://juejin.im/post/5cee800051882544171c5a2c)
>
> [在 Android 上使用协程（三） ：Real Work](https://juejin.im/post/5cf513d3e51d4577407b1ceb)

回到正题，本篇主要介绍 Retrofit 2.6.0 版本中协程的使用方式，不会过多涉及原理。我以我自己的 [wanandroid](https://github.com/lulululbj/wanandroid) 应用为例进行改造，源代码中 Retrofit 版本是 `2.4.0` 。这个 wanandroid 是基于 `Kotlin + 协程 + LiveData + MVVM ` 实现的，具体架构可见我的文章 [真香！Kotlin+MVVM+LiveData+协程 打造 Wanandroid！](https://juejin.im/post/5cb473e66fb9a068af37a6ce) ，个人觉得代码还是比较清晰的，很适合作为 Kotlin 的 入门项目。

## 老版本 Retrofit 的使用

在介绍如何使用 `Retrofit 2.6.0 ` 之前，我们先来看一下老版本的 Retrofit 是如何基于 Kotlin Coroutines 工作的，以登录接口为例。

首先在 [WanService](https://github.com/lulululbj/wanandroid/blob/mvvm-kotlin/app/src/main/java/luyao/wanandroid/model/api/WanService.kt) 接口中作如下定义：

```kotlin
@POST("/user/login")
fun login(@Field("username") userName: String,
          @Field("password") passWord: String): Deferred<WanResponse<User>>
```

注意这里使用的返回值是 `Deferred<T>` 对象，这就意味着使用的时候要通过 `await` 来获取返回值。那么如何让 Retrofit 直接返回 `Deferred<T>` 呢？使用的也是 JakeWharton 的开源库：

```
implementation 'com.jakewharton.retrofit:retrofit2-kotlin-coroutines-adapter:0.9.2'
```

在构建 Client 的时候添加上这个适配器：

```kotlin
...
.addCallAdapterFactory(CoroutineCallAdapterFactory.invoke())
...
```

然后给 [LoginRepository](https://github.com/lulululbj/wanandroid/blob/mvvm-kotlin/app/src/main/java/luyao/wanandroid/model/repository/LoginRepository.kt) 提供一个 suspend 方法：

```kotlin
suspend fun login(userName: String, passWord: String): WanResponse<User> {
    return apiCall { WanRetrofitClient.service.login(userName, passWord).await() }
}
```

这里使用 `await` 来获取 `Deferred<T>` 的返回值。

最后在 [LoginViewModel](https://github.com/lulululbj/wanandroid/blob/mvvm-kotlin/app/src/main/java/luyao/wanandroid/ui/login/LoginViewModel.kt) 中是这样调用的：

```kotlin
fun login(userName: String, passWord: String) {
    launch {
        val response = withContext(Dispatchers.IO) { repository.login(userName, passWord) }
        executeResponse(response, { mLoginUser.value = response.data }, { errMsg.value = response.errorMsg })
    }
}
```

`launch()` 方法做了简单的封装，感兴趣的同学可以到源码中看一下。

以上就是在 `Retrofit 2.4.0` 中使用协程的基本方式了，其实代码也很简洁。而 `Retrofit 2.6.0` 让这一切更加简单！就让我们一睹为快吧！

## Retrofit 2.6.0 中协程的使用

`Talking is cheap, show me the code !` 还是上面的登录接口，基于 `Retrofit 2.6.0` 来改造一下。

第一步，修改 Retrofit 依赖。

```
implementation 'com.squareup.retrofit2:retrofit:2.6.0'
```

第二步，修改 `WanService` 中接口的定义。

```kotlin
@POST("/user/login")
suspend fun login(@Field("username") userName: String,
                  @Field("password") passWord: String): WanResponse<User>
```

看到区别了吗？首先，不再返回 `Deferred<T>` 对象，而是直接返回我们需要的 `WanResponse` 对象。其次，使用了 `suspend` 来修改方法，标记这是挂起函数。

第三步，修改 `LoginRepository` 中方法定义。

```kotlin
suspend fun login(userName: String, passWord: String): WanResponse<User> {
    return apiCall { WanRetrofitClient.service.login(userName, passWord) }
}
```

与之前的版本相比，这里不需要调用 `await` 方法了。其实并不是不调用了，而是 Retrofit 帮助我们自动调用了。

最后别忘了去除之前添加的 `kotlin-coroutines-adapter`，因为我们不再需要人工返回 `Deferred<T>` 对象，也不再需要手动调用 `await` 了。

```kotlin
...
//.addCallAdapterFactory(CoroutineCallAdapterFactory.invoke())
...
```

至此，基于 `Retrofit 2.6.0` 版本的改造就已经完成了。我的 Wanandroid 项目已经完成全部修改，具体修改内容可见 [commit](https://github.com/lulululbj/wanandroid/commit/220344f3a7c03a7be98757d160e0df29d2f29552)。

## 总结

随着 Kotlin 成为 Android 开发的首选语言，越来越多的新特性都将在 Kotlin 上优先实现。协程作为 Kotlin 的异步利器，很值得我们学习。如果你还没有入手，那么，从我的 [Wanandroid](https://github.com/lulululbj/wanandroid) 开始吧 ！

> 文章首发微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解。
>
> 更多相关知识，扫码关注我吧！

![](https://user-gold-cdn.xitu.io/2019/4/27/16a5f352eab602c4?w=2800&h=800&f=jpeg&s=178470)
