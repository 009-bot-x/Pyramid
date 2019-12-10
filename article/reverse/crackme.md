往期目录：

> [Class 文件格式详解](https://juejin.im/post/5c0932cee51d45090a1da07e)
>
> [Smali 语法解析——Hello World](https://juejin.im/post/5c093fd751882535422e4f05)
>
> [Smali —— 数学运算，条件判断，循环](https://juejin.im/post/5c0d1a7e6fb9a04a0604b0ec)
>
> [Smali 语法解析 —— 类](https://juejin.im/post/5c0fc82c5188250d2722a8b1)
>
> [Android逆向笔记 —— AndroidManifest.xml 文件格式解析](https://juejin.im/post/5c2253f6f265da616d54377b)
>
> [Android逆向笔记 —— DEX 文件格式解析](https://juejin.im/post/5cdd5e0ce51d453a4a357ea1)

无意中在看雪看到一个简单的 [CrackMe](https://bbs.pediy.com/thread-250705.htm) 应用，正好就着这个例子总结一下逆向过程中基本的常用工具的使用，和一些简单的常用套路。感兴趣的同学可以照着尝试操作一下，过程还是很简单的。APK 我已上传至 Github，[下载地址](https://github.com/lulululbj/android-reverse/blob/master/android/crackme/crackme.apk)。

首先安装一下这个应用，界面如下所示：


![](https://user-gold-cdn.xitu.io/2019/5/22/16adf76834a897dc?w=1080&h=1920&f=png&s=119150)

要求就是通过注册。爆破的方法很多，大致可以归为三类，第一种是直接修改 smali 代码绕过注册，第二种是捋清注册流程，得到正确的注册码。第三种是 hook 。下面就来说说这几种爆破过程。

## 直接修改 smali 进行爆破

要获取 smali 代码，首先得反编译这个 Apk，通过 [ApkTool](https://ibotpeaches.github.io/Apktool/) 就可以完成。`ApkTool` 的使用过程就不在这里赘述了，执行如下命令：

```bash
apktool d creackme.apk
I: Using Apktool 2.3.4-dirty on crackme.apk
I: Loading resource table...
I: Decoding AndroidManifest.xml with resources...
I: Loading resource table from file: /home/luyao/.local/share/apktool/framework/1.apk
I: Regular manifest package...
I: Decoding file-resources...
I: Decoding values */* XMLs...
I: Baksmaling classes.dex...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...
```

会在当前目录生成 `crackme` 文件夹，文件夹目录如下：


![](https://user-gold-cdn.xitu.io/2019/5/22/16adf81c5014ecca?w=565&h=279&f=png&s=15251)

其中的 `smali` 文件夹就包含了该 Apk 的所有 smali 代码。阅读和修改 smali 代码的工具很多，我个人偏好将整个反编译得到的文件夹导入 IDEA 或者 Android Studio 进行阅读和修改，可能我是 Android 开发，用这两个工具会比较顺手，全局搜索功能也很给力。

导入 Android Studio 之后，看到了所有的 smali 代码，那么我们该从何下手呢？注册失败的时候会弹一个 Toast，“无效用户名或注册码”，这就是突破口。全局搜索这个字符串，
![](https://user-gold-cdn.xitu.io/2019/5/22/16adf88141d16d38?w=829&h=177&f=png&s=24527)

发现这个字符串定义在 `string.xml` 中的 `unsuccessd` ，在写代码的时候就是 `R.string.unsuccessd`，这是一个 int 值，编译后就直接是一个数字了。我们再来全局搜索 `unsuccessd` :
![](https://user-gold-cdn.xitu.io/2019/5/22/16adf8a555465cfb?w=826&h=217&f=png&s=41762)

在 `public.xml` 中可以看到它的 `id`,代码中直接使用的就是这个 id了。全局搜索一下 `0x7f05000b`，看一下这个 Toast 是在哪里弹出的。


![](https://user-gold-cdn.xitu.io/2019/5/22/16adf8cbd4ffff15?w=827&h=211&f=png&s=43069)

可以看到这个 id 在 `MainActivity.smali` 中的 433 行使用到了，我们定位到这个文件：

```smali
    .line 117
    if-nez v0, :cond_0  # 如果 v0 不等于 0 ，跳转到 cond_0

    .line 119
    const v0, 0x7f05000b

    .line 118
    invoke-static {p0, v0, v2}, Landroid/widget/Toast;->makeText(Landroid/content/Context;II)Landroid/widget/Toast;

    move-result-object v0

    .line 119
    invoke-virtual {v0}, Landroid/widget/Toast;->show()V
```

这段逻辑很简单。判断寄存器 v0 的值是否为 0，不为 0 的话则弹出 “无效用户名或注册码” 。所以最简单的改法，逻辑反一下，v0 为 0 的时候弹出该 Toast，把 `if-nez` 改为 `if-ez` 即可。修改之后使用 `ApkTool` 重打包，重打包命令如下：

```bash
apktool b crackme -o crackme_new.apk
```

会在当前目录生成 `crackme_new.apk` 文件，注意这个安装包是未签名的，无法直接安装，需要先签名。使用 `jarsinger` 或者 `apksigner` 都可以。签名之后安装，输入用户名：


![](https://user-gold-cdn.xitu.io/2019/5/22/16adfdd99e2806b6?w=1080&h=1920&f=png&s=121090)

这样就注册成功了。方法虽然有点 low ，但好歹爆破成功了。下面我们不修改 smali 代码，通过阅读 smali 代码理解其注册码生成逻辑，通过正规方式来注册。

## 获取注册码爆破

我们之前已经找到了具体的逻辑是在 `MainActivity.smali` 中，找到这个按钮的 `onClick()` 事件，来看一下具体逻辑：

```smali
.line 116
invoke-direct {p0, v0, v1}, Lcom/droider/crackme0201/MainActivity;->checkSN(Ljava/lang/String;Ljava/lang/String;)Z

move-result v0

.line 117
if-eqz v0, :cond_0

.line 119
const v0, 0x7f05000b

.line 118
invoke-static {p0, v0, v2}, Landroid/widget/Toast;->makeText(Landroid/content/Context;II)Landroid/widget/Toast;

move-result-object v0

.line 119
invoke-virtual {v0}, Landroid/widget/Toast;->show()V

goto :goto_0
```

这里只截取了 `onClick` 中的部分核心代码，调用 `checkSN()` 方法获得一个 Boolean 值，根据这个值来判断是否注册成功。这个 `checkSN()` 方法就是我们需要重点关注的，我对这个方法的 smali 代码逐行添加了注释，还是很容易理解的，感兴趣的同学可以看一下：

```smali
.method private checkSN(Ljava/lang/String;Ljava/lang/String;)Z
    .locals 10  # 使用 10 个寄存器
    .param p1, "userName"   # Ljava/lang/String; 参数寄存器 p1 保存的是用户名 userName
    .param p2, "sn"    # Ljava/lang/String; 参数寄存器 p2 保存的是注册码 sn

    .prologue
    const/4 v7, 0x0 # 将 0x0 存入寄存器 v7

    .line 45
    if-eqz p1, :cond_0  # 如果 p1，即 userName 等于 0，跳转到 cond_0

    :try_start_0
    invoke-virtual {p1}, Ljava/lang/String;->length()I # 调用 userName.length()

    move-result v8  # 将 userName.length() 的执行结果存入寄存器 v8

    if-nez v8, :cond_1 # 如果 v8 不等于 0，跳转到 cond_1

    .line 69
    :cond_0
    :goto_0
    return v7

    .line 47
    :cond_1
    if-eqz p2, :cond_0  # 如果 p2，即注册码 sn 等于 0，跳转到 cond_0

    invoke-virtual {p2}, Ljava/lang/String;->length()I  # 执行 sn.length()

    move-result v8  # 将 sn.length() 执行结果存入寄存器 v8

    const/16 v9, 0x10 # 将 0x10 存入寄存器 v9

    if-ne v8, v9, :cond_0   # 如果 sn.length != 0x10 ，跳转至 cond_0

    .line 49
    const-string v8, "MD5"  # 将字符串 "MD5" 存入寄存器 v8

    # 调用静态方法 MessageDigest.getInstance("MD5")
    invoke-static {v8}, Ljava/security/MessageDigest;->getInstance(Ljava/lang/String;)Ljava/security/MessageDigest;

    move-result-object v1   # 将上一步方法的返回结果赋给寄存器 v1，这里是 MessageDigest 对象

    .line 50
    .local v1, "digest":Ljava/security/MessageDigest;
    invoke-virtual {v1}, Ljava/security/MessageDigest;->reset()V # 调用 digest.reset() 方法

    .line 51
    invoke-virtual {p1}, Ljava/lang/String;->getBytes()[B   # 调用 userName.getByte() 方法

    move-result-object v8   # 上一步得到的字节数组存入 v8

    invoke-virtual {v1, v8}, Ljava/security/MessageDigest;->update([B)V # 调用 digest.update(byte[]) 方法

    .line 52
    invoke-virtual {v1}, Ljava/security/MessageDigest;->digest()[B  # 调用 digest.digest() 方法

    move-result-object v0   # 上一步的执行结果存入 v0，是一个 byte[] 对象

    .line 53
    .local v0, "bytes":[B
    const-string v8, "" # 将字符串 "" 存入 v8

    # 调用 MainActivity 中的 toHexString(byte[] b,String s) 方法
    invoke-static {v0, v8}, Lcom/droider/crackme0201/MainActivity;->toHexString([BLjava/lang/String;)Ljava/lang/String;

    move-result-object v3   # 上一步方法返回的字符串存入 v3

    .line 54
    .local v3, "hexstr":Ljava/lang/String;
    new-instance v5, Ljava/lang/StringBuilder;  # 新建 StringBuilder 对象

    invoke-direct {v5}, Ljava/lang/StringBuilder;-><init>()V    # 执行 StringBuilder 的构造函数

    .line 55
    .local v5, "sb":Ljava/lang/StringBuilder;   # 声明变量 sb 指向刚才创建的 StringBuilder 实例
    const/4 v4, 0x0 # v4 = 0x0

    .local v4, "i":I    # i = 0x0
    :goto_1 # for 循环开始
    invoke-virtual {v3}, Ljava/lang/String;->length()I  # 获取 hexstr 字符串的长度

    move-result v8  # v8 = hexstr.length()

    if-lt v4, v8, :cond_2   # 如果 v4 小于 v8，即 i < hexstr.length(), 跳转到 cond_2

    .line 58
    # 这里已经跳出 for 循环
    invoke-virtual {v5}, Ljava/lang/StringBuilder;->toString()Ljava/lang/String;

    move-result-object v6   # v6 = sb.toString()

    .line 63
    .local v6, "userSN":Ljava/lang/String;  # userSN = sb.toString()

    # userSN.equalsIgnoreCase(sn)
    invoke-virtual {v6, p2}, Ljava/lang/String;->equalsIgnoreCase(Ljava/lang/String;)Z

    move-result v8  # v8 = userSN.equalsIgnoreCase(sn)

    if-eqz v8, :cond_0 # 如果 v8 等于 0，跳转到 cond_0，即 userSN != sn

    .line 69
    const/4 v7, 0x1

    goto :goto_0    # 跳转到 goto_0，结束 checkSN() 方法并返回 v7

    .line 56
    .end local v6    # "userSN":Ljava/lang/String;
    :cond_2
    invoke-virtual {v3, v4}, Ljava/lang/String;->charAt(I)C # 执行 hexstr.charAt(i)

    move-result v8  # v8 = hexstr.charAt(i)

    # 调用 sb.append(v8)
    invoke-virtual {v5, v8}, Ljava/lang/StringBuilder;->append(C)Ljava/lang/StringBuilder;
    :try_end_0
    .catch Ljava/security/NoSuchAlgorithmException; {:try_start_0 .. :try_end_0} :catch_0

    .line 55
    add-int/lit8 v4, v4, 0x2    # v4 自增 0x2，即 i+=2

    goto :goto_1    # 跳转到 goto_1，形成 循环

    .line 65
    .end local v0    # "bytes":[B
    .end local v1    # "digest":Ljava/security/MessageDigest;
    .end local v3    # "hexstr":Ljava/lang/String;
    .end local v4    # "i":I
    .end local v5    # "sb":Ljava/lang/StringBuilder;
    :catch_0
    move-exception v2

    .line 66
    .local v2, "e":Ljava/security/NoSuchAlgorithmException;
    invoke-virtual {v2}, Ljava/security/NoSuchAlgorithmException;->printStackTrace()V

    goto :goto_0
.end method
```

大致逻辑就是对输入的用户名 UserName 作 MD5 运算得到 Hash 值，再转成十六进制字符串就是注册码了。那么，如何获取注册码呢 ？一般有三种方式，打 log，动态调试 smali，自己写注册机。下面逐个说明一下。

### 打 log 日志

其实在逆向过程中，注入 log 代码是很常见的操作。适当的打 log，可以很好的帮助我们理解代码执行流程。在这里例子中，最终会拿我们输入的注册码和正确的注册码进行比较，在比较的时候我们就可以通过打 log 把正确的注册码打印出来，这样我们就可以直接输入注册码进行注册了。

打 log 的 smali 代码是固定的，一般格式如下：

```smali
const-string vX, "TAG"
invoke-static {vX,vX}, Landroid/util/Log;->e(Ljava/lang/String;Ljava/lang/String;)I
```

`vX` 都是指寄存器。把这两行代码加到注册码的检验操作之前就可以了：

```smali
.line 63
.local v6, "userSN":Ljava/lang/String;  # userSN = sb.toString()

const-string v8, "TAG"
invoke-static {v8,v6}, Landroid/util/Log;->e(Ljava/lang/String;Ljava/lang/String;)I

# userSN.equalsIgnoreCase(sn)
invoke-virtual {v6, p2}, Ljava/lang/String;->equalsIgnoreCase(Ljava/lang/String;)Z
```

再次重新打包运行，输入用户名和注册码，就会有如下日志：


![](https://user-gold-cdn.xitu.io/2019/5/22/16adffd234c563cd?w=942&h=116&f=png&s=19180)

这样就拿到正确的注册码了。

### 动态调试 smali

动态调试 smali 来的更加直截了当。不管是你自己写程序，还是做逆向，debug 永远都是快速理清逻辑的好方法。smali 也是可以进行动态调试的，依赖于 [Smalidea](https://bitbucket.org/JesusFreke/smali/commits/) 插件，你可以在 Android Studio 的 Plugin 中进行安装，也可以下载下来本地安装。

第一步，我们要保证我们的应用处于 debug 版本，在 AndroidManifest.xml 中加上 `android:debuggable="true"` 即可，重打包再安装到手机上。

第二步，将之前反编译得到的 smali 文件夹导入 Android Studio 或者 IDEA，并配置远程调试环境。选择 Run -> Edit Configurations，点击左上角 + 号，选择 Remote，弹出配置窗口，如下图所示：


![](https://user-gold-cdn.xitu.io/2019/5/22/16ae010f984b57d7?w=1085&h=684&f=png&s=71517)

注意记住自己填写的端口号，端口号不是固定的，只要未被占用即可。配置完成后，记得在合适的地方打上断点，我这里就在 `checkSN()` 方法内打上断点。

第三步，命令行启动进程调试等待模式。首先执行：

```bash
adb shell am start -D -n com.droider.crackme0201/.MainActivity
```

应用此时会进入等待调试模式，如下图所示：


![](https://user-gold-cdn.xitu.io/2019/5/22/16ae015b6dca3222?w=1080&h=1920&f=png&s=72954)

然后建立端口转发，输入如下命令：

```bash
adb forward tcp:8700 jdwp:pid
```

用你自己的应用的 pid 替换进去。关于 pid 的获取，可以通过 `ps` 和 `grep` 组合：

```bash
adb shell ps | grep com.droider.crackme0201
u0_a364   30110 537   2166480 30204 futex_wait 0000000000 S com.droider.crackme0201
```

我这里的 pid 就是 `30010` 。

最后在 Android Studio 或 IDEA 中启动 debug 。 点击 Run -> Debug，应用就进入调试模式了。之后的操作就和我们开发中的 debug 模式一模一样了。我们可以在运行中看到寄存器中的值，运行逻辑一览无遗。运行至注册码校验处的断点，截图如下：


![](https://user-gold-cdn.xitu.io/2019/5/22/16ae01de750b074a?w=1002&h=486&f=png&s=75313)

`userName` 是用户名，`sn` 是我输入的注册码，`userSN` 是正确的注册码。

### 注册机

注册机其实就是自己重写注册码生成过程了，看懂了 smali 就可以自己写个程序来生成注册码了。这个就不多说了。

## Hook

具体的 Hook 操作由于篇幅原因就不在这里演示了。关于 Java 层的 Hook 工具很多，最普遍的就是 Xposed，直接 hook `checkSN` 方法的返回值，或者打印出正确的注册码。如果你没有 Root 设备，还有一系列基于 [VirtualApp](https://github.com/asLody/VirtualApp) 的 hook 框架，例如支持 Xposed 应用的 [VirtualXposed](https://github.com/android-hacker/VirtualXposed) 等等，当然 VirtualApp 本身也支持 hook 操作。另外，还有 [Frida](https://www.frida.re/) 等等框架，也可以进行类似的操作。

## JADX

最后再介绍一个反编译利器 [JADX](https://github.com/skylot/jadx) ，它可以直接将 Apk 反编译成 Java 代码进行查看，毕竟 smali 代码不是那么人性化。我拿到一个 Apk，基本上第一件事就是丢到 JADX 中进行查看，它同时支持命令行操作和图形化界面。我们就用 JADX 打开这个 CrackMe 应用看一下：


![](https://user-gold-cdn.xitu.io/2019/5/23/16ae500141406f83?w=1346&h=750&f=png&s=114662)

直接就可以看到对应的 Java 代码，理清逻辑之后再去阅读 smali 代码进行修改，事半功倍。支持反编译 Java 代码的工具还有很多，例如基于 Python 实现的 [Androgurad](https://androguard.readthedocs.io/en/latest/) 等等，大家也可以尝试去使用一下。

## 总结

就逆向难度来说，这个 CrackMe 还是很简单的，但本文主旨在于介绍一些逆向相关的知识，实际逆向过程中你面对的任何一个 Apk 肯定都比这复杂的多。看到这里，你应该了解到了下面这些知识点：

* 使用 ApkTool 反编译以及重打包
* smali 代码的基本阅读能力
* smali 代码中注入 log 日志
* 动态调试 smali 代码
* 常用 hook 框架
* jadx 使用

关于 smali 语法我之前也写过几篇文章，往期目录：

> [Class 文件格式详解](https://juejin.im/post/5c0932cee51d45090a1da07e)
>
> [Smali 语法解析——Hello World](https://juejin.im/post/5c093fd751882535422e4f05)
>
> [Smali —— 数学运算，条件判断，循环](https://juejin.im/post/5c0d1a7e6fb9a04a0604b0ec)
>
> [Smali 语法解析 —— 类](https://juejin.im/post/5c0fc82c5188250d2722a8b1)
>
> [Android逆向笔记 —— AndroidManifest.xml  文件格式解析](https://juejin.im/post/5c2253f6f265da616d54377b)
>
> [Android逆向笔记 —— DEX 文件格式解析](https://juejin.im/post/5cdd5e0ce51d453a4a357ea1)

下一篇来写写 Android Apk 中资源包文件 `resources.arsc` 的文件结构，同样会配套思维导图和 Java 源码解析。


> 文章首发微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解。
>
> 更多 JDK 源码解析，扫码关注我吧！

![](https://user-gold-cdn.xitu.io/2019/4/27/16a5f352eab602c4?w=2800&h=800&f=jpeg&s=178470)
