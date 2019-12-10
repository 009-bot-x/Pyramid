往期目录：

> [走进 JDK 之 Integer](https://juejin.im/post/5c76ad1ae51d4572c95835d0)
>
> [走进 JDK 之 Long](https://juejin.im/post/5c7d45ef6fb9a049e232bde8)
>
> [走进 JDK 之 Float](https://juejin.im/post/5c7fe55a6fb9a049f23d8814)
>
> [走进 JDK 之 Byte](https://juejin.im/post/5c8c9e836fb9a049c43e927e)

今天来说说 `Boolean` 。`Boolean` 类源码也很简单，在阅读源码的过程中思考这么一个问题，`Boolean` 类型在内存中是如何表示的？或者说，`JVM` 是如何看待 `Boolean` 的？

## 类声明

```java
public final class Boolean implements java.io.Serializable,Comparable<Boolean>
```

`Boolean` 也是不可变类，事实上所有的基本类型包装类、`String`、`BigDecimal`、`BigInteger` 也都是不可变类。

## 字段

```java
private final boolean value;
public static final Boolean TRUE = new Boolean(true); // true
public static final Boolean FALSE = new Boolean(false); // false
public static final Class<Boolean> TYPE = (Class<Boolean>) Class.getPrimitiveClass("boolean");
private static final long serialVersionUID = -3665804199014368530L;
```

`Boolean` 类型只有两个值，`true` 和 `false`。

## 构造函数

```java
public Boolean(boolean value) {
    this.value = value;
}

public Boolean(String s) {
    this(parseBoolean(s));
}
```

还是熟悉的味道，第一个构造函数直接传入 `boolean`。第二个构造函数调用 `parseBoolean()` 方法，将 `String` 转换为布尔值。

## 方法

### parseBoolean()

```java
public static boolean parseBoolean(String s) {
    return ((s != null) && s.equalsIgnoreCase("true"));
}
```

直接和字符串 `true` 进行比较。

### valueOf()

```java
public static Boolean valueOf(boolean b) {
    return (b ? TRUE : FALSE);
}

public static Boolean valueOf(String s) {
    return parseBoolean(s) ? TRUE : FALSE;
}
```

### toString()

```java
public static String toString(boolean b) {
    return b ? "true" : "false";
}
```

### hashcode()

```java
public static int hashCode(boolean value) {
    return value ? 1231 : 1237;
}
```

源代码都很简单，没有什么好说的。回过头看看文章开头的问题：

> JVM 是怎么处理 Boolean 的 ？

源码中貌似也看不出什么端倪，我们得从 Java 虚拟机的角度出发了。先看下面这个例子：

```java
public class BooleanTest {
    public void test() {
        boolean flag = true;
        if (flag)
            System.out.println("This is true.");
    }
}
```

`test` 方法显然会打印字符串 `This is true`。站在人脑的思维很容易理解，下面我们站在 `JVM` 的思维来看一下该如何理解。

首先虚拟机肯定是不认识这些源代码的，它认识的只有字节码，也就是 `class` 文件。关于 `Class` 文件的具体格式，可以看看我之前的一篇文章，[Class 文件格式详解](https://juejin.im/post/5c0932cee51d45090a1da07e)。我在这就直接使用 `javap` 命令来查看字节码了。

```
javac BooleanTest.java
javap -v BooleanTest.class
```

略去常量池等部分内容，我把 `test()` 方法的字节码内容拿过来：

```
public void test();
    descriptor: ()V     // 方法描述符
    flags: ACC_PUBLIC   // 访问标志
    Code:
      stack=2, locals=2, args_size=1
         0: iconst_1
         1: istore_1
         2: iload_1
         3: ifeq          14
         6: getstatic     #2    // Field java/lang/System.out:Ljava/io/PrintStream;
         9: ldc           #3    // String This is true.
        11: invokevirtual #4    // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        14: return
```

下面逐行分析 `Code` 部分：

```
stack=2, locals=2, args_size=1
```

`stack` 表示操作数栈深度的最大值，这里是 2 。`locals` 表示局部变量表所需的存储空间，这里需要两个 `slot`。还记得什么是 `slot` 吗，`slot` 是虚拟机为局部变量分配内存所使用的最小单位。`args_size` 是参数个数，这里 `test()` 方法并没有参数，但是每个方法都有一个参数是指向当前引用自身的。

```
0: iconst_1     // 将一个 int 常量 1 加载到操作数栈
1: istore_1     // 将数值 1 从操作数栈存储到局部变量表
2: iload_1      // 将局部变量 1 加载到操作数栈
```

这三行字节码其实就是 `boolean flag = true;` 。`JVM` 并没有为 `boolean` 专门做处理，而是直接当做 `int` 处理。`true` 就是 `1`, `false` 就是 `0` 。

```
3: ifeq          14
...
14: return
```

`ifeq` 是控制转移指令，这里的含义是如果操作数栈上的值是 `0`, 就跳转到 `14` 处，`14` 处指令为 `return`，则结束方法执行。这里已经将 `1` 加载到操作数栈，所以会继续往下执行，省略号中的字节码内容就是打印语言 `System.out.println("This is true.");`，不作过多分析。

根据 Java 虚拟机规范，`JVM` 并没有任何供 `boolean` 值专用的字节码指令，Java 源代码中使用到的布尔值，在编译之后都使用 `int` 值来代替。`JVM` 也支持 `boolean` 类型数组，其一般经过编译会被当作 `byte` 数组进行处理。所以，在字节码中，你是看不到 `boolean` 的。

还记得上篇文章 [走进 JDK 之 Byte](https://juejin.im/post/5c8c9e836fb9a049c43e927e#heading-7) 中提出的一个问题，`作为方法内部局部变量的 byte 在内存中占几个字节 ？` 结论是：

> 基本类型作为方法局部变量是存储在栈帧上的，除了 long 和 double 占两个 Slot，其他都占用一个 Slot

在 `JVM` 的眼里，并没有这么多的数据类型，对于 `boolean` 、`byte` 、`short` 和 `char`，在编译期都会变成 `int` 类型，`JVM` 也仅仅只对 `int` 提供了最完整的操作码，其他类型数据的操作，都是使用相应的 `int` 类型的操作码进行操作。那么 `JVM` 为什么没有给每种数据类型都配置完整的操作码呢？这还得从操作码的长度说起。

Java 虚拟机操作码的长度为一个字节，所以字节码指令集的操作码总数不可能超过 `256` 条。这么做是为了尽可能获得短小精干的字节码，字节码指令流都是单字节对齐的，数据量小，传输效率高。当然，这么做的代价就是你不可能设计出一套面向所有数据类型都完整的操作码。如果每一种数据结构都要得到 Java 虚拟机的字节码指令的支持的话，那么指令的数量将远远超过 `256` 种。所以，这也给指令集的设计带来了麻烦。最终权衡的结果就是，只对有限的类型提供完整的指令。大部分的指令都没有支持 `byte` 、 `char` 和 `short`，`boolean` 则更惨，没有任何指令支持 boolean 类型。对于这些不支持的指令类型，一律使用 `int` 的相关指令代替。

## 总结

`Boolean` 其实是被当做 `int` 值处理的，`true` 表示 `1`，`false` 表示 `0`

`JVM` 为 `int` 提供了完善的操作码，`boolean` 、`byte` 、`char` 、`short` 在编译期或运行期都会被转换为 `int`，使用 `int` 类型的字节码指令进行处理

> 文章同步更新于微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解，欢迎关注！

![]![](https://user-gold-cdn.xitu.io/2019/3/30/169cf046d9579e78?w=258&h=258&f=jpeg&s=27711)
