项目地址：[Box](https://github.com/lulululbj/Box)

为什么要做 `Box` ? 其实源于开发过程中一些很小的需求，也许并不常见，但是每次碰到都要费一些功夫。所以想着写一个小应用，集成一些常用功能，给开发带来一些便利。下面这些小需求，你也遇到过吗？

## 如何查看当前 Activity ？

看到别人优秀的 UI 界面想借鉴一下如何实现？看到别人的应用实现了自己不知道如何实现的功能？反编译了别人的 apk 却不知道去哪找代码？经常做逆向的同学应该经常碰到这些问题。首先，你肯定得找到当前 `Activity` 的名称，才能顺路找到相应的代码。那么，如何查看当前 `Activity` 的名称呢？常见的做法是通过 `adb` 命令：

```
adb shell dumpsys activity activities | grep mFocusedActivity
```

执行结果如下：


![](https://user-gold-cdn.xitu.io/2019/3/14/1697c520bf1092c3?w=732&h=78&f=png&s=19211)

无奈记性不好，经常忘记命令，每次都得去搜一下，顺便推荐一个 [adb 命令集合](https://github.com/mzlogin/awesome-adb)。再来看一下 `Box` 中这一功能是什么效果：


![](https://user-gold-cdn.xitu.io/2019/3/14/1697c59b4e914c4e?w=1080&h=1920&f=png&s=110134)

通过悬浮窗实时显示当前 `Activity` ，简单便捷。实现原理也很简单，通过 **无障碍服务** 监听窗口变化并获取当前 `Activity` 名称。悬浮窗没有自己造轮子了，使用了开源项目 [FloatWindow](https://github.com/yhaolpz/FloatWindow)。有段时间没更新了，顺便也改了几个 bug。

## 如何获取已安装应用的 Apk 文件 ？

某天突然看上了手机里的某个 App，想拖到 `jadx` 里面看一看，如何快速的获取到安装包文件呢？我们都知道对于已安装的应用，系统都备份了安装包，存储在 `/data/app/[packageName]` 目录下，一般文件名为 `base.apk`。如果是具有 root 权限的手机，我们可以直接拿到文件。对于非 root 手机，还是有读权限的，可以通过文件 API 直接读取。`Box` 中界面如下所示：


![](https://user-gold-cdn.xitu.io/2019/3/14/1697c6a76be49fc5?w=1080&h=1920&f=png&s=178188)

安装包文件会复制到手机根目录 `/Box/apk/应用名` 文件夹下。

## 如何快速查看 AndroidManifest.xml 文件 ？

`AndroidManifest.xml` 包含了应用的基本信息，如何快速的查看应用的清单文件？之前有一个开源工具，`AXmlPrinter.jar`，可以直接解析安装包中的二进制清单文件。本想直接把 jar 包拿过来用，可是用的不是那么随心应手。加上之前亲手解析过 `AndroidManifest.xml` 文件，详见 [Android逆向（一） —— AndroidManifest.xml 二进制解析](https://juejin.im/post/5c2253f6f265da616d54377b)。索性就用了自己的代码，顺便也修复了一些解析过程中遇到的 bug。具体效果如下：


![](https://user-gold-cdn.xitu.io/2019/3/14/1697c777479a57e9?w=1080&h=1920&f=png&s=142024)

![](https://user-gold-cdn.xitu.io/2019/3/14/1697c77a70de71a0?w=1080&h=1920&f=png&s=405700)

## 其他

主要功能都在上面了，另外还加了一块 `本机信息`,包括内容如下：

* 品牌、版本号、型号、主板、制造商等
* 屏幕、RAM、ROM、SDK 版本、Android 版本、ABIS 等
* IMEI、MEID、SN、MAC 地址等


![](https://user-gold-cdn.xitu.io/2019/3/14/1697c7a8f66058f6?w=1080&h=1920&f=png&s=144573)

![](https://user-gold-cdn.xitu.io/2019/3/14/1697c7abb5c8c407?w=1080&h=1920&f=png&s=159211)

`Box` 使用 `kotlin` 开发，简单的使用了 `协程` 来进行异步任务，如复制 Apk 并更新进度等。界面设计很一般，实在是很为难我这么一个理科生。图片基本上都是 `Android Studio` 里面自动生成的，`logo` 暂时还没有，有合适的可以 `push` 一个。

第一个版本功能比较简单，后面再遇到开发中的痛点需求，还会加进来，持续更新。有 bug 或者好的想法，欢迎 [issue](https://github.com/lulululbj/Box/issues) 、 [pr](https://github.com/lulululbj/Box/pulls) 砸过来！

项目源码： [Box](https://github.com/lulululbj/Box)

> 文章首发微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解，欢迎关注！

![](https://user-gold-cdn.xitu.io/2019/3/30/169cf046d9579e78?w=258&h=258&f=jpeg&s=27711)
