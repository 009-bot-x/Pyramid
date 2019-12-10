
>  项目地址： [Box](https://github.com/lulululbj/Box)
>
> 文末扫码获取最新安装包 。

## 前言

有将近一个月没有更新文章了，一方面在啃 AOSP ，消化起来确实比较慢。在阅读的过程中，有时候上来就会陷入源码细节，其实这是没有必要的。刚开始更多的应该从整体脉络上去理解，摸清整个流程之后再去有针对性的看某些细节，才会事半功倍。下一篇应该会带来 **Activity 启动流程分析** 。

<!-- more -->

除了啃 AOSP 之外，剩下的时间都花在了开源项目的维护和更新上。一个是 [Wanandroid](https://github.com/lulululbj/wanandroid) 应用，主要技术栈是 `Kotlin 、 MMVM 、 协程` ，开源了一段时间，一度觉得自己的 MVVM 写的还不错。在阅读相关架构文章以及 Google 重构了 [plaid](https://github.com/android/plaid) 之后，发现了自己的框架在 **分离关注点** 方面存在的一些问题。主要针对架构方面做了一些调整，目前来看还是比较符合 MVVM 的思想的。另外，也新增了网页版的新功能 “广场”。

说一说 Wanandroid 后续的更新计划，第一点，Jetpack 的深anzhuangb入使用。包括 Navigation 单 Activity 实现，Room ，Page 等类库的使用。第二点，完成一个 **Jetpack Compse** 版本，虽然 Compose 还是预览版，但我坚定看好 Compose，实在忍不住不去尝试一下，其实也已经在开发中了，完成了一些简单页面，有在学习 Compose 的朋友可以交流交流，项目地址在这里 -》 [Wanandroid-Compose](https://github.com/lulululbj/Wanandroid-Compose) 。

## Box V0.2.0

另一个开源项目就是今天要说的 [Box](https://github.com/lulululbj/Box) 了，说来惭愧，已经好几个月没有更新了。这次带来了一个 "黑科技"，对，没错，就是堪比 **小米手机八项黑科技** 的 **手机端反编译**  功能。熟悉反编译的同学应该对这个功能很熟悉，但都是在 PC 上操作的，`Apktool` ，`Jadx` 等开源工具都提供了 PC 端的命令行操作或者图形界面。其实第一次看到手机端反编译功能是在 Trinea 的 [Android 开发助手](https://www.trinea.cn/android/android-dev-tools-5-6-0/) 上，当时感觉挺惊艳的，也比较好奇是如何实现的。anzhuangb

其实很简单，Apktool 和 Jadx 都是开源的，移植到 Android 上就可以了。大致浏览了一下 Jadx 源码，就开始了移植工作。鉴于 Jadx 源码的优秀设计，整个移植过程也没有费太大功夫。结合  Android 开发助手的 UI 设计，不难看出 Trinea 也是移植了 Jadx 源码。
box_app_managerbox_app_manager
下面的 gif 简单展示了反编译功能的使用：

![](https://user-gold-cdn.xitu.io/2019/11/13/16e655dd5a97b797?w=480&h=800&f=gif&s=740477)nager

除此之外，针对之前的 **当前 Activity** 功能做了一些完善，主要替换了悬浮窗的依赖库，现在使用的是 [EasyFloat](https://github.com/princekin-f/EasyFloat)。这是一个 Kotlin 版本，且更加稳定。下面也用一个 gif  演示一下该功能：


![](https://user-gold-cdn.xitu.io/2019/11/13/16e655f39aacecce?w=480&h=800&f=gif&s=1963035)

另外，在更新 [AndroidUtilCodeKTX](https://github.com/lulululbj/AndroidUtilCodeKTX) 的文件工具类部分时，为了能总结的尽量完整，就在 Box 里面增加了 **文件管理** 功能，界面相对简陋，但功能还算完整，后续会继续完善，大家可以提提 issue 。

针对 **应用管理** 功能，新增了对本地安装包文件的支持。无需安装也能直接查看各种应用信息。关于其中一个查看 `AndroidManifest.xml` 文件的功能，建议阅读 [Android逆向笔记 —— AndroidManifest.xml 文件格式解析](https://juejin.im/post/5c2253f6f265da616d54377b) 。


![](https://user-gold-cdn.xitu.io/2019/11/13/16e655f756a47f3f?w=480&h=800&f=gif&s=1926197)

## 最后

如果你有新奇的想法和功能，欢迎前来交流。

添加我的微信，加入技术交流群。


![](https://user-gold-cdn.xitu.io/2019/11/17/16e79e67e6fb7c03?w=447&h=422&f=png&s=98388)

公众号后台回复 “Box”， 获取最新安装包。


![](https://user-gold-cdn.xitu.io/2019/11/13/16e656421741bfbc?w=2800&h=800&f=jpeg&s=178470)
