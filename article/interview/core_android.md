逛 Github 的时候看到一份超强面试题，来自 [MindOrks](https://mindorks.com/) 开源的 [android-interview-questions](https://github.com/MindorksOpenSource/android-interview-questions)。虽说是一份安卓面试题，但其中包含了 `数据结构与算法` 、`Java 核心基础` 、`Android 核心基础` 、`设计模式` 等各方面内容。大致浏览了一下，质量还不错，比百度出来的各种所谓 `BAT 面试题` 强一些。也看到国内一些开发者有做翻译，比如 `stormzhang` 发起的 [android-interview-questions-cn](https://github.com/stormzhang/android-interview-questions-cn)。可能由于种种原因，未能完成全部工作，且题目也已经更新了很多。

出于复习的目的吧，正好也在写 `走进 JDK` 系列的文章（可以看看我的专栏），就打算把这些题目都捋一遍，尽可能带来高质量的答案，估计会花费不短时间。今天这篇文章先把所有题目整理出来，后面会陆续配上答案。

**`拉到文末直接获取所有题目 pdf 文件 。`**

## Core Java

### 面向对象
#### 1. 什么是 OOP ？
#### 2. 抽象类和接口的区别 ？
#### 3. Iterator 和 Enumeration 的区别 ？
#### 4. 你同意 组合优先于继承 吗 ？
#### 5. 方法重载和方法重写的区别 ？
#### 6. 你知道哪些访问修饰符 ？ 它们分别的作用 ？
#### 7. 一个接口可以实现另一个接口吗 ？
#### 8. 什么是多态 ？什么是继承 ？
#### 9. Java 中类和接口的多继承
#### 10. 什么是设计模式？

### 集合和泛型

#### 11. Arrays vs ArrayLists
#### 12. HashSet vs TreeSet
#### 13. HashMap vs HashSet
#### 14. Stack vs Queue
#### 15. 解释 java 中的泛型
#### 16. String 类是如何实现的？它为什么被设计成不可变类 ？

### 对象和基本类型

#### 16. String 类是如何实现的？它为什么被设计成不可变类 ？
#### 17. 为什么说 String 不可变 ？
#### 18. 什么是 String.intern() ？ 何时使用？ 为什么使用 ？
#### 19. 列举 8 种基本类型
#### 20. int 和 Integer 区别
#### 21. 什么是自动装箱拆箱 ？
#### 22. Java 中的类型转换
#### 23. Java 值传递还是引用传递 ？
#### 24. 对象实例化和初始化之间的区别 ？
#### 25. 局部变量、实例变量以及类变量之间的区别？

### Java 内存模型和垃圾收集器

#### 26. 什么是垃圾收集器 ？ 它是如何工作的 ？
#### 27. 什么是 java 内存模型？ 它遵循了什么原则？它的堆栈是如何组织的 ？
#### 28. 什么是 内存泄漏，java 如何处理它 ？
#### 29. 什么是 强引用，软引用，弱引用，虚引用 ？

### 并发

#### 30. 关键字 synchronized 的作用 ？
#### 31. ThreadPoolExecutor 作用 ？
#### 32. 关键字 volatile 的作用 ？
#### 33. The clasess in the atomic package expose a common set of methods: get, set,, lazyset, compareAndSet, and weakCompareAndSet. Please describe them.

### 异常

#### 34. try{} catch{} finally{} 是如何工作的　?
#### 35. Checked Exception 和 Un-Checked Exception 区别 ？

### 其他

#### 36. 什么是序列化？如何实现 ？
#### 37. 关键字 transient 的作用 ？
#### 38. 什么是匿名内部类 ？
#### 39. 对象的 == 和 .equals 区别 ？
#### 40. hashCode() 和 equals() 用处 ？
#### 41. 构造函数中为什么不能调用抽象方法 ？
#### 42. 你什么时候会使用 final 关键字 ？
#### 43. final, finally 和 finalize 的区别 ？
#### 44. Java 中 static 关键字的含义 ？
#### 45. 静态方法可以重写吗 ?
#### 46. 静态代码块如何运行 ？
#### 47. 什么是反射 ？
#### 48. 什么是依赖注入 ？列举几个库 ？你使用过吗 ？
#### 49. StringBuilder 如何避免不可变类 String 的分配问题？
#### 50. StringBuffer 和 StringBuilder 区别 ？
#### 51. Enumeration and an Iterator 区别 ？
#### 52. fail-fast and fail-safe 区别 ？
#### 53. 什么是 NIO ？

## Core Android

### Base

#### 54. Android 应用组件
#### 55. Android 应用架构
#### 56. 什么是 Context？
#### 57. 什么是 AndroidManifest.xml？
#### 58. 什么是 Application ？

### Activity

#### 59. 什么是 Activity ？
#### 60. 说明一下 Activity 和 Fragment 的生命周期
#### 61. 什么是 Activity 的启动模式 ？

### Fragments

#### 62. 什么是 Fragment ?
#### 63. Activity 和 Fragment 关系和区别 ？
#### 64. 为什么建议使用默认构造函数来创建 Fragment ？
#### 65. Fragment 之间如何通信 ？
#### 66. 什么是 Retained Fragment ?

### View 和 ViewGroup

#### 67. 在 Android 中，什么是 View ？
#### 68. View.GONE 和 View.INVISIBLE 的区别 ？
#### 69. 如何创建自定义 View ？
#### 70. 什么是 ViewGroups 以及和 View 的区别 ？
#### 71. 什么是 canvas ？
#### 72. 什么是 SurfaceView ？
#### 73. 相对布局和线性布局对比
#### 74. 谈谈 Constraint Layout
#### 75. 你知道 View 树吗 ？如何优化它的深度 ？

### 展示内容集合

#### 76. ListView 和 RecyclerView 区别 ？
#### 77. 什么是 ViewHolder ？为什么使用它 ？
#### 78. 什么是 SnapHelper ？

### Dialog 和 Toast

#### 79. 什么是 Dialog ？
#### 80. 什么是 Toast ？
#### 81. Dialog 和 Dialog Fragment 区别 ？

### Intent 和 广播

#### 82. 什么是 Intent ？
#### 83. 什么是 显示 Intent ？
#### 84. 什么是 隐式 Intent ？
#### 85. 什么是 BroadcastReceiver ？
#### 86. 什么是 LocalBroadcastReceiver ？
#### 87. IntentFilter 的作用 ？
#### 88. 什么是 sticky intent ？
#### 89. 说说广播和 Intent 是如何在你的应用中传递消息的 ?
#### 90. 什么是 PendingIntent ？
#### 91. 广播的不同类型 ？

### Services
#### 92. 什么是 Service ？
#### 93. Service 和 IntentService
#### 94. 什么是 JobSchedule ？

### Inter-process Communication
#### 95. 两个不同的 app 如何通信 ？
#### 96. 一个 app 可以多进程运行吗 ？如何实现 ？
#### 97. 什么是 AIDL ？ 列举实现步骤。
#### 98. 你可以使用后台进程干什么 ？
#### 99. 什么是 ContentProvider ？一般用来干什么 ？

### Long-running Operations
#### 100. 如何进行耗时任务 ？
#### 101. 为什么要避免在主线程运行非ui代码 ？
#### 102. 什么是 ANR ？如何预防 ？
#### 103. 什么是 AsyncTask ？
#### 104. AsyncTask 有哪些问题 ？
#### 105. 你会在什么时候使用 AsyncTask 代替线程 ？
#### 106. 什么是 Loader ？
#### 107. AsyncTask 和 Activity 的生命周期有什么联系 ？会导致什么问题 ？如何避免 ?
#### 108. 解释 Looper, Handler 和 HandlerThread 的作用

### 多媒体
#### 109. 如何处理 Bitmap 占据大量内存 ？
#### 110. 一个标准的 Bitmap 和一个 .9 图的区别 ？
#### 111. 谈谈 Bitmap pool
#### 112. Android 中如何播放声音 ？

### Data Saving
#### 113. 如何持久化数据 ？
#### 114. 什么是 ORM ？它如何工作 ？
#### 115. 屏幕旋转时如何保存 Activity 状态 ？
#### 116. 你的应用中保存数据的不同方式 ？

### Look and feel
#### 117. 什么是 Spannable ？

### 内存优化
#### 118. 什么是 onTrimMemory() 方法 ？
#### 119. OutOfMemory 是如何发生的 ？
#### 120. 在 Android 中你是如何找到内存泄漏的 ？

### 电量优化
#### 121. 在 Android 中如何降低电量消耗 ？
#### 122. 什么是 Doze ？应用支持如何 ?
#### 123. 什么是过度绘制 ？

### Supporting Different Screen Sizes
#### 124. 你是如何进行屏幕适配的 ?

### Permissions
#### 125. 权限中有哪些不同的保护级别 ?

### Native Programming
#### 126. 什么是 NDK ，它的作用是什么 ？
#### 127. 什么是 renderscript ？

### Android System Internal
#### 128. 什么是 Dalvik Virtual Machine ？
#### 129. JVM, DVM 和 ART 区别
#### 130. Dalvik 和 ART 区别
#### 131. 什么是 Dex？
#### 132. 你可以手动调用垃圾回收吗 ？

### Debugging and Programming Tools
#### 133. 什么是 ADB ？
#### 134. 什么是 DDMS ？你可以用它干什么 ？
#### 135. 什么是 StrictMode ？
#### 136. 什么是 lint ？ 它的作用是什么 ？

### Others
#### 137. 为什么使用 Bundle 传递数据 ？ 为什么不使用 Map ？
#### 138. 你是如何解决应用中的 crash 的 ？
#### 139. 说说 Android 通知 体系
#### 140. Serializable 和 Parcelable 区别 ？ Android 中使用哪个更好 ？
#### 141. 开发过 widgets 吗 ？
#### 142. 什么是 AAPT ？
#### 143. 定时刷新页面的最好方法是什么 ？
#### 144. FlatBuffers 和 JSON
#### 145. HashMap, ArrayMap 和 SparseArray
#### 146. 什么是注解 ？
#### 147. android 中如何处理 multi-touch ？
#### 148. 如何实现 XML 命名空间 ？
#### 149. 什么是 support library ？以及为什么引入 ？
#### 150. 什么是 Android Data Binding ？
#### 151. 什么是 Android Architecture Components ？
#### 152. 如何使用 RxJava 操作符实现查找 ?

## 架构

#### 153. 描述一下你最近开发的 App 使用的架构
#### 154. 说说 MVP
#### 155. 什么是 Presenter ？
#### 156. 什么是 Model ？
#### 157. 说说 MVC
#### 158. 说说 MVI
#### 159. 说说 Repository pattern
#### 160. 什么是 Controller ？
#### 161. Tell something about clean code

## Android 测试驱动开发
#### 162. 什么是 Espresso ？
#### 163. 什么是 Robolectric ？
#### 164. 使用 Robolectric 的缺点是什么 ？
#### 165. 什么是 UI-Automator ？
#### 166. 说说单元测试
#### 167. 说说自动化测试
#### 168. 你进行过单元测试或者自动化测试吗 ？
#### 169. 为什么使用 Mockito ？

## 其他
#### 170. 什么是 Android Jetpack ？
#### 171. 说说 REST APIs 如何工作的
#### 172. 说说其他的 Web Api 架构
#### 173. 说说数据库，Sqlite
#### 174. 关于项目管理工具，trello, basecamp, kanban, jira, asana
#### 175. 关于构建系统， gradle, maven, ant, buck
#### 176. 应用多 Apk 文件
#### 177. 反编译 Apk
#### 178. ProGuard 被用来做什么 ？
#### 179. 什么是混淆 ？ 它的作用是什么 ？ minification 呢 ？
#### 180. 你如何构建 release 安装包 ？
#### 181. 你如何控制对于特定用户的版本更新 ？
#### 182. 我们可以找出已经卸载我们的应用的用户吗 ？
#### 183. Apk 文件大小优化
#### 184. 你尝试过 Kotlin 吗 ？
#### 185. 在开发过程中如何持续监测各种指标 ？
#### 186. 什么是 Chrome Custom Tabs ？ 如何在你的 app 中展示网页内容 ？

数据结构这块的题目不是很详细，就没有加上来。其他的根据实际情况作了部分删减，共计 186 题。

> 微信搜索 `秉心说` ，或者扫码下列二维码关注公众号，回复 `面试题` 即可获取所有题目 `pdf` 文档，后续所有答案也会通过公众号通知，欢迎大家关注。

> 文章首发于微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，JDK 、AOSP 源码解析，LeetCode 题解，欢迎扫码关注 ！


![](https://user-gold-cdn.xitu.io/2019/3/30/169cf046d9579e78?w=258&h=258&f=jpeg&s=27711)
