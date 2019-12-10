回顾一下前面的一系列文章，

> [走进 JDK 之 Integer](https://juejin.im/post/5c76ad1ae51d4572c95835d0)
>
> [走进 JDK 之 Long](https://juejin.im/post/5c7d45ef6fb9a049e232bde8)
>
> [走进 JDK 之 Float](https://juejin.im/post/5c7fe55a6fb9a049f23d8814)
>
> [走进 JDK 之 Byte](https://juejin.im/post/5c8c9e836fb9a049c43e927e)
>
> [走进 JDK 之 Boolean](https://juejin.im/post/5c9996d6f265da60d82dd645)

除了 `char` 和 `double`，基本涵盖了 Java 的所有基本类型。今天就来总结一下基本类型的相关知识。

## 基本类型概述

Java 中有 8 种基本类型，如下表所示：

| 基本类型 | 大小  | 最大值 | 最小值 | 包装类 | 虚拟机中符号 |
|  :---:   | :---: | :---:  |  :---: |:---:   | :---: |
| boolean | - | - | - | Boolean | Z |
| char | 16 bits | 65536 | 0 | Character | C |
| byte | 8 bits | 127 | -128 | Byte | B |
| short | 16 bits | 2<sup>15</sup>-1 | - 2<sup>15</sup> | Short | S |
| int | 32 bits | 2<sup>31</sup>-1 | 2<sup>31</sup> | Integer | I |
| long | 64 bits | 2<sup>63</sup>-1 | -2<sup>63</sup> | Long | J |
| float | 32 bits | 3.4028235e+38f | -3.4028235e+38f | Float | F |
| double | 64 bits | 1.7976931348623157e+308 | -1.7976931348623157e+308 | Double | D |

每种基本类型所占的存储空间大小不会随着机器硬件架构的变化而变化，具有良好的可移植性。

数值类型有 `byte` 、`short` 、`int` 、`long` 、`float` 、`double`。其中 `byte` 、`short` 、`int` 、`long` 是整数类型。`float` 和 `double` 是浮点数类型，分别表示单精度和双精度，遵循 `IEEE 754` 浮点数标准。Java 中所有数值类型都是有符号数，最高位表示符号位。

`boolean` 是布尔类型，只有两个值 `TRUE` 和 `FALSE`，在 `JVM` 中当做 `int` 处理，两个值分别为 `1` 和 `0`。

`char` 是字符类型，为 `Unicode` 编码。在某些应用场景下，可以把 `char` 当作无符号数来处理。

对于上面表格中的内容，还有几点需要说明一下：

> 溢出问题

整数类型都有自己的取值范围，如果强行超出它的范围会怎么样呢？如下面代码：

```java
public void test() {
    byte max = Byte.MAX_VALUE;
    byte over = (byte) (max + 1);
    System.out.println(over);
}
```

打印结果是 `-128`，即 `Byte.MIN_VALUE`。你可以把值域范围想象成时钟的刻度，最大值是 12 ，最小值是 0 ，已经到了最大值 12 ，你还要再加 1 ，就发生了溢出，又回到了最小值 0 。

> 栈上大小

在栈上，`byte` 、`short` 、`boolean` 、`char` 占用的空间和 `int` 是一致的，都是占一个 `slot`。`long` 和 `double` 是占用两个 `slot`。具体原因在之前的文章中具体分析过，这里再总结一下：

* 局部变量表可以看成一个 slot 数组，这样设计方便使用索引来获取数据
* 操作码是单字节的，最多只有 256 个字节码指令，不可能为每一个基本数据类型提供完整的指令支持。

当然，这仅仅只是针对栈上，对于堆上和数组中分配的基本类型，其大小还是和表中匹配的。

* 不同类型的字节码指令处理

这块内容在 [走进 JDK 之 Boolean](https://juejin.im/post/5c9996d6f265da60d82dd645) 中详细介绍过。操作码是单字节的，最多只有 256 个字节码指令，不可能为每一个基本数据类型提供完整的指令支持。大部分的指令都没有支持 `byte` 、 `char` 和 `short`，`boolean` 则更惨，没有任何指令支持 `boolean` 类型。对于这些不支持的指令类型，一律使用 `int` 的相关指令代替。

对于不确定的情况，你可以看一下 class 文件中的字节码：

```
javac Test.java
javap -v Test.class
```

## 为什么需要基本类型 ？

基本所有语言都有基本数据类型，Java 当然也是如此。标榜 `万物皆对象` 的 Java 为什么需要基本数据类型的存在呢？回顾一下你的职业生涯中写过的代码，基本数据类型出现的概率可以说相当之高，基本数据类型是直接存储在栈内存的，而 `new` 出来的对象是存储在堆内存中的。显然对象占用的内存是高于基本数据类型的。对象在内存中存储的布局可以分为 3 块区域：`对象头` 、`实例数据` 、`对齐填充`。如果把你的代码中所有基本类型全部替换为其包装类，无疑会占用更多的内存，也降低了运行效率。

## 为什么需要包装类 ？

就冲那句 `万物皆对象` ，当然也得有包装类了！开个玩笑而已 ... 基本数据类型占用内存更少，运行速度更快，但它也有办不到的事情，比如我要构造一个包含 `int` 的 `List` 集合：

```java
List list = new ArrayList<int>();
```

显然这是没法通过编译的。这里我们传进去的必须是一个对象，所以基本数据类型还必须得有一个包装类。包装类中包装了基本类型的数值，并且提供了一系列处理数值的方法，就像前面分析的过得源码中各种方法，丰富了基本数据类型的功能。

## 自动拆箱与自动装箱

把基本数据类型转换成包装类的过程叫做装箱。

把包装类转换成基本数据类型的过程叫做拆箱。

在Java SE 5.0 之前，装箱和拆箱需要手动进行。可能 Java 开发者们觉得这样实在太不人性化了，一切应该以提高生产力为主，在 Java SE 5.0 中就推出了自动装箱和拆箱。以 `Integer` 为例：

```java
Integer i = 10;  //自动装箱
int b = i;     //自动拆箱
```

我们已经解析过 `Integer` 的源码，不难想象，自动装箱和拆箱必然是调用了包装类的某些方法，`javap` 看一下字节码：

```
2: invokestatic  #2         // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
5: astore_1
6: aload_1
7: invokevirtual #3         // Method java/lang/Integer.intValue:()I
```

很明了，之前的代码经过编译器编译，已经改变了模样：

```java
Integer i = Integer.valueOf(10);  //装箱
int b = i.intValue();     //拆箱
```

还记得 `valueOf()` 方法吗？

```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

通过 `IntegerCache` 缓存了 `-128` 到 `127`。在此范围内直接返回缓存的实例，否则创建新对象。

不仅仅是 `Integer`，其他基本数据类型也都是使用类似 `valueOf()` 和 `xxxValue()` 方法来进行自动装箱和拆箱的。

现在你应该知道了自动装箱和自动拆箱实际上是编译器在里面做了手脚，和 Java 虚拟机并没有什么关系。编译器在生成类的字节码时，插入必要的方法调用。虚拟机只是执行这些字节码。

## 最后

走进 JDK 系列的基本数据类型部分就说到这里了，还有什么疑问的话，欢迎关注我的公众号 **`秉心说`** 给我留言。

下一篇是 `走进 JDK 之 String`，不出意外的话应该要过几天，`String` 的内容还是比较多的，可以先阅读阅读我之前的一篇文章 [String 为什么不可变？](https://juejin.im/post/59cef72b518825276f49fe40) 。

> 文章首发于微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解，欢迎关注！


![](https://user-gold-cdn.xitu.io/2019/3/30/169cf046d9579e78?w=258&h=258&f=jpeg&s=27711)
