> 原文作者： [Roman Elizarov](https://medium.com/@elizarov)
>
> 原文地址： [Null is your friend, not a mistake](https://medium.com/@elizarov/null-is-your-friend-not-a-mistake-b63ff1751dd5)
>
> 译者：秉心说

![](https://user-gold-cdn.xitu.io/2019/9/18/16d42d32986fb56c?w=4000&h=1041&f=jpeg&s=715713)

[Kotlin Island from Wikimedia by Pavlikhin, CC BY-SA 4.0](https://commons.wikimedia.org/wiki/File:Kotlin_Island_west_side.jpg)

我使用 Java 语言编程已经很久很久了，掌握了通过 Java 编写和维护大型软件（百万行代码）应该注意些什么，并亲眼目睹了全行业都在竭力避免空指针异常 `NullPointerException（NPE）`，它困扰着大大小小的 Java 类库。在 2009 年其发明者 `Tony Hoare` 承认空引用是他造成的一个 [“十亿美元的错误”](https://www.infoq.com/presentations/Null-References-The-Billion-Dollar-Mistake-Tony-Hoare/)之前，人人已经意识到它的危险性。

在 1996 年 Java 1.0 发布时，这个问题还不是如此明显。让我们看一个典型的 Java API 的例子：[`File.list()`](https://docs.oracle.com/javase/8/docs/api/java/io/File.html#list--) 方法。它被用来列举文件夹中的内容，如下所示：

```java
for(String name : new File("directory").list()) {
    System.out.println(name);
}
```

仅当文件夹存在时上面的代码才会正常运行，否则将抛出 `NPE`，因为 `list()` 返回了 `null`。但是谁会写这样的代码呢？不仅仅 `list()` 方法的文档中清楚的说明了文件夹不存在时将返回 `null`，而且现代的 IDE 也会对特定的代码给你提出警告。

但是，当使用 Java 编程时，开发者经常犯这类错误。到目前为止，有大量研究表明它们是如何发生的。结果表明，大多数情况下，我们的 API 函数不应该返回 null，其他开发者也并不希望返回 null。在一些特殊情况下，比如缺省值，Java 中的惯例是返回一些 “空对象”（空集合，未填充的对象等等），或者抛出异常，比返回 null 更糟糕。这就是为什么要设计 [`Files.newDirectoryStream`](https://docs.oracle.com/javase/8/docs/api/java/nio/file/Files.html#newDirectoryStream-java.nio.file.Path-), 高级版本的 `File.list`，任何情况都不会出现 null。

所以当 null 在一些特殊情况下作为函数返回值时，如性能优化，未初始化的引用字段等等，它常常会让你措手不及，没有做好准备去处理它。不仅仅是必须处理空值的情况很少，而且在 Java 中用来处理空值的代码是很啰嗦的：

```java
String[] list = new File("directory").list();
if (list != null) {
    for (String name : list) {
        System.out.println(name);
    }
}
```

毫无疑问，除非真的需要（你的客户在生产环境发现了 NPE），不然你真的不想写这样的代码。

对 null 的恐惧导致了一些极端情况。有一些 Java 编码风格完全禁止 null，将可恶的工作交给开发者。不知道你有没有见过这样的 Java 类库，所有的域对象都要实现一个特殊的 `Null` 接口，并且提供手动编码生成的 “空对象” 实例。如果没有见过说明你还是幸运的。但是我敢打赌你已经看到了只为了避免空值而污染 Java 代码的 [`Optional <T>`](https://docs.oracle.com/javase/8/docs/api/java/util/Optional.html) 包装器（译者注：Java 8 新特性）。

有些集合框架的 API 出于谨慎禁止 null 元素，并且一些 Java 核心团队成员认为 Java 集合框架对 null 的支持是一个错误。这让人非常难过。

**事实上，null 这个概念不是一个错误，但是 Java 的类型系统认为 null 是任何类型的成员。** 让我们看看，在 Java 中 `“abc”` 是一个合法的 `String`，`null` 也是一个合法的 `String`。你可以在前者上使用 string 的所有方法，例如 `substring`。但是在后者上使用则会发生运行时错误。它是类型安全的吗？并不完全是。一个类型的特定值在进行某些操作时发生运行时异常（例如除 0）是正常的，但是当对一个值进行该类型的所有操作都发生了异常，这首先表明的是这个值并不属于这个类型。所有的那些 NPE 都表明了 Java 的类型系统存在明显的缺陷。

更多的类型安全的编程语言，例如 Kotlin，通过合理的将 null 的概念合并到类型系统中来修复这个缺陷。添加检查和警告也有一定作用，但这并不够。显然，一个健全的类型系统必须允许 `String` 类型的所有变量都支持它的操作。所以在 Kotlin 中，将 `null` 赋给 `String` 类型的变量就不仅仅只是一个警告了，而是类型错误，就像把数值 `42` 赋给 `String` 类型变量一样。

类型系统中合理的 null 支持是 API 设计的一个转折点，没有任何理由再去害怕 null 了。一些函数返回可空类型 `String?`，另一些函数返回不可空类型 `String`，就和一些函数返回 `String`，另一些返回 `Int` 一样。它们都是不同的类型，有着不同但是安全的操作集。

用类型安全的 null 来表示 “缺失的值” 是更好，更高效，更简洁的。看一下 Kotlin 标准库中的 [`String.toIntOrNull()`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.text/to-int-or-null.html) 函数，用于将 string 转为数字，不能转换的话返回 null。使用起来很愉快，编写一个命令行程序，接受 integer 参数并处理参数的缺失就很简单：

```kotlin
fun main(args: Array<String>) {
    val id = args.getOrNull(0)?.toIntOrNull()
        ?: error("id expected")
    // ...
}
```

在 API 设计中使用 null 吧，它是你和 Kotlin 的好朋友。没有理由去害怕它，也没有理由使用 `null object` 模式或者包装器来处理它，更不用说异常了。在你的 API 中合理使用 null 会给你带来更可读，更安全的代码，并且远离样板代码。

## 深入阅读

如果你喜欢这个主题，并且想了解更多关于语言设计的细节，那么可以考虑阅读这篇文章—— [Dealing with absence of value](https://medium.com/@elizarov/dealing-with-absence-of-value-307b80534903) 。
> 文章首发微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解。
>
> 更多最新原创文章，扫码关注我吧！

![](https://user-gold-cdn.xitu.io/2019/4/27/16a5f352eab602c4?w=2800&h=800&f=jpeg&s=178470)
