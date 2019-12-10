在进入正题之前，推荐阅读一下之前的两篇文章。第一篇是我的一篇译文 —— 译文找不到了，就放一下原文吧。

> [Closer Look At Android Runtime: DVM vs ART](https://android.jlelse.eu/closer-look-at-android-runtime-dvm-vs-art-1dc5240c3924)

上面这篇文章简单比较了 Dalvik 和 Art 。其中的一些细节在我的另一篇文章 [说说方舟编译器](https://juejin.im/post/5cb07000f265da037d4f9be6) 中也有所提及，大家可以大致浏览一下。

然后再推荐一篇 [Android逆向笔记 —— DEX 文件格式解析](https://juejin.im/post/5cdd5e0ce51d453a4a357ea1#heading-12)，在最后解析 `DexCode` 部分时，详细的逐字节的解析了一段 Dalvik 字节码。大家可以挑这一段阅读一下，对 Dalvik 字节码有一个大概的认识。

下面就正式来进入 Dalvik 的世界。

## Dalvik 虚拟机

`Dalvik` 是早期 Android 版本中用于运行安卓应用的虚拟机，由 `Dan Bornstein` 编写的，名字来源于他的祖先曾经居住过名叫 Dalvík 的小渔村，村子位于冰岛。当年也有一部分业内人士认为 Dalvik 是 Google 为了避免与 Oracle 的诉讼而诞生的产物。Dalvik 是基于 `Apache License 2.0` 发布的。Google 说 Dalvik 是一个清洁室（clean room）的实现，而不是一个在标准 Java 运行环境的改进，这意味着它不继承标准版本的或开源的 Java 运行环境的版权许可限制。关于这一点，Oracle 和一些专家还在讨论中。

Dalvik 是解释执行加上 JIT，每次app运行的时候，它动态的将一部分 Dalvik 字节码
解释为机器码。随着 App 的运行，更多的字节码被编译和缓存。因为 JIT 只编译了一部分代码，它具有更小的内存占用和更少的设备物理空间占用。但是，边解释边执行，效率低下，这也是后来 Dalvik 遭到抛弃的原因。

从 Android 4.4 开始，Google 开始引入了全新的虚拟机 ART（Android Runtime）。ART 是基于 AOT 编译的，由于安装应用耗时过程，后期高版本的 Android 系统加入了加强版的 JIT 编译。Dalvik 在 Android 5.0 中正式被删除，ART 完成上位。那么现在来学习 Dalvik 还有必要吗？其实 ART 是向下兼容的，ART 和 Dalvik 是运行 Dex 字节码的兼容运行时，因此针对 Dalvik 开发的应用也能在 ART 环境中运作。不过，Dalvik 采用的一些技术并不适用于 ART。因此，Dalvik 虚拟机的部分特性以及 Dalvik 字节码指令其实和 ART 都是相通的。

### Dalvik 和 JVM

Dalvik 和 JVM 并不兼容，甚至可以说完全是两套机制。下面来说几点它们之间的区别。

1. 运行的字节码不同

我们都知道 JVM（Java 虚拟机）识别的是 Class 文件，我之前写过一篇 [Class 文件格式详解](https://juejin.im/post/5c0932cee51d45090a1da07e)，详细介绍了 Class 文件的二进制结构。JVM 运行的是 Java 字节码，而 Dalvik 运行的是 Dalvik 字节码。Dalvik 不识别单个的 Class 文件，而是将所有 Class 文件打包成 DEX 文件格式，通过解释 DEX 文件来执行字节码。

这样带来的直接好处就是 Dalvik 的可执行文件的体积更小。如果你了解 Class 文件格式的话，你会知道每个 Class 文件都有单独的字符串常量池。如果不同的 Class 文件中有相同的字符串，那么就存在重复存储的情况。同样的，如果一个类引用了其他类中的方法，相应的方法签名也会被复制到该类文件中。这样就会有很多不必要的冗余信息，既浪费内存也影响执行效率。

那么 DEX 文件是如何解决这个问题的呢？对 DEX 文件结构不了解的话，可以阅读我的另一篇文章 [Android逆向笔记 —— DEX 文件格式解析](https://juejin.im/post/5cdd5e0ce51d453a4a357ea1)。DEX 文件提供了一个统一的共享的常量池，供所有类文件使用，这样就避免了冗余信心，减小了文件体积，提高了解析效率。

2. 虚拟机架构不同

JVM 是基于栈架构的。当程序运行时，Java 虚拟机会频繁的对栈进行读写数据的操作。在这个过程中，不仅会多次进行指令分派和内存访问，而且会耗费大量的 CPU 时间，因此，对于资源有限的手机设备来说，是一笔很大的开销。每调用一个方法，就会分配一个新的栈帧并压入栈。每从一个方法返回，就弹出相应的栈帧。

Dalvik 是基于寄存器架构的，数据的访问直接在寄存器之间传递。

> 基于堆栈的机器与基于寄存器的机器谁更有优势一直是个争论不休的话题。
一般来说,基于堆栈的机器必须使用指令才能从堆栈上的加载和操作数据,因此,相对基于寄存器的机器，它们需要更多的指令才能实现相同的性能。但是基于寄存器机器上的指令必须经过编码,因此,它们的指令往往更大。

上面这段来自百度。的确，Java 虚拟机的操作码都是单字节的，其指令字总操作码个数不超过 256 条。而 Dalvik 指令则长的多的多，数量也多的多。要执行相同的操作，JVM 需要短但是更多的指令，Dalvik 需要长但是更少的指令。Dalvik 的思路是用更长但是更少的指令来减少指令分派和内存访问，以此提高运行效率。

## Dalvik 指令

### 指令格式

关于 Dalvik 指令格式，[官网](https://source.android.google.cn/devices/tech/dalvik/instruction-formats.html?hl=zh-tw) 中也有相关介绍。只是官网的介绍实在过于晦涩，看了很多遍才理解。我这里还是从实际的 Dalvik 指令来进行分析。把之前分析过的 `main()` 方法直接拿过来：

```java
public static void main(String[] args) {
    System.out.println(HELLO_WORLD);
}
```

其 DexCode 如下：

```
62 00 01 00 62 01 00 00 6E 20 03 00 10 00 0E 00
```

这里要注意的是 DEX 文件是小端表示法，低位在前，高位在后。通常低 8 位就是 op 码，也就是我们说的操作码。在上面的例子中，第一个操作码就是 `62`，我们在 Dalvik 指令集中可以找到其代表的指令。关于 Dalvik 指令集，Android 开发者网站也做了总结，[点我查看](https://source.android.google.cn/devices/tech/dalvik/dalvik-bytecode?hl=zh-tw#instructions)。另外，在 Android 4.4 之前的 AOSP 源码中的 [dalvik/libdex/DexOpcodes.h](http://androidxref.com/4.4_r1/xref/dalvik/libdex/DexOpcodes.h) 中也有定义。我这里直接在官网资源中查找 `62`，结果如下图所示：


![](https://user-gold-cdn.xitu.io/2019/6/12/16b4bbc2610c7c1e?w=969&h=234&f=png&s=67448)

操作码 `62` 表示的指令是 `sget-object`，表示获取一个静态对象。仅仅知道了操作码的含义还是不够的，我们还不知道该条指令的完整格式。注意列表最左侧的 `21c`，它表示的就是指令的格式。关于指令的格式，Android 官网也做了相关总结，[点我查看](https://source.android.google.cn/devices/tech/dalvik/instruction-formats.html?hl=zh-tw#formats)。同样的，AOSP 中也有相关定义，位于 Android 4.0 版本中的 [dalvik/docs/instruction-formats.html](http://androidxref.com/4.0.4/xref/dalvik/docs/instruction-formats.html) 文件。查一下 `21c` 的指令格式，如下图所示：


![](https://user-gold-cdn.xitu.io/2019/6/12/16b4bc297cf0d230?w=756&h=295&f=png&s=38649)

可得 `21c` 对应的指令格式为 `AA|op BBBB`。对应的上面的十六进制，做一下对比：

```
AA|op BBBB -> op vAA kind@BBBB
00|62 0001 -> 62 v00 kind@0001
```

这样一看，就很清晰了。该指令一共是两个 16 位的字，第一个 16 位的低 8 位是操作码 62，表示 `sget-object`，高 8 位表示使用的是 `v0` 寄存器。第二个 16 位是索引值 1，指向 Dex 中 field_id 部分的第一项，根据之前的解析结果，第一项表示的字段是 `Ljava/lang/System;->out;Ljava/io/PrintStream`，整合一下，这个指令的完整格式如下：

```
sget-object v0, Ljava/lang/System;->out:Ljava/io/PrintStream;
```

表示获取静态字段 `PrintStream.out`，并保存在寄存器 `v0` 中。回头再看一下 `21c`，它的每一个字符其实都是有含义的。

* `2` 表示该指令有多少个 16 位的字组成
* `1` 表示该指令最多使用多少个寄存器
* `c` 为类型码，c 代表常量池索引

关于类型码，还有很多种，如下表所示：

| 助记符 | 位数	| 含义 |
| --- | --- | --- |
| b	|8|	有符号立即数（字节）|
|c|	16、32|	常量池索引|
|f|	16	|接口常量（仅对静态链接格式有效）
|h|	16	|有符号立即数（32 位或 64 位值的高阶位，低阶位全为 0）
|i|	32	|有符号立即数（整型）或 32 位浮点数
|l|	64	|有符号立即数（长整型）或 64 位双精度浮点数
|m|	16	|方法常量（仅对静态链接格式有效）
|n|	4	|有符号立即数（半字节）
|s|	16	|有符号立即数（短整型）
|t|	8、16、32|	分支目标
|x|	0	|无额外数据

还有一种特殊情况指令的末尾会多出一个字母。如果是 `s`，表示指令采用静态链接。如果是字母 `i`，表示指令应该被内联处理。

### 寄存器命名

我们都知道 Dalvik 虚拟机是基于寄存器架构的，其使用的寄存器都是 32 位的。对于 64 位类型，使用相邻两个寄存器来表示。Dalvik 基本都是基于 ARM 架构的，ARM 架构的 CPU 本身就含有一定数量的寄存器，那么 Dalvik 虚拟机支持多少个寄存器呢？我们来看一个 `move` 指令：

```
move/16 vAAAA, vBBBB
```

`vAAAA vBBBB`，每个大写字母表示 4 位，一共就是 `2^16 -1`，也就是 `65535` 个。当然，不可能会有 65535 个真实寄存器。Dalvik 使用的是虚拟寄存器，它会将部分寄存器映射到 ARM 的寄存器上，另外一部分通过调用栈进行模拟。

Dalvik 虚拟机为每一个进程维护一个调用栈，这个栈的作用之一就是虚拟寄存器。虚拟机通过处理字节码对寄存器进行读写操作，实际上就是对栈空间进行读写。但是在实践中，一个方法需要 16 个以上的寄存器不太常见，而需要 8 个以上的寄存器却相当普遍，因此很多指令仅限于寻址前 16 个寄存器。

那么寄存器是如何命名的呢？上面的分析中提到过 `v0` 寄存器，是不是 65535 个寄存器就是 `v0 - v65535` 呢？实际上，寄存器有两种命名方式，`v 命名法` 和 `p 命名法`。在介绍它们之前，先来说一些基本概念。以下面这个 `add` 函数为例：

```java
public int add(int a,int b){
    int c = a+b;
    return c;
}
```

它使用了几个寄存器？如果不是很确定，可以查看其 smali 代码中的 `.registers` 字段。答案是 4 个。根据 Dalvik 虚拟机规定，方法参数使用最后面的寄存器。这 4 个寄存器中的最后两个就是存储参数 `a` 和 `b`。由于 `add()` 是非静态函数，所以该方法总是会传入当前对象的引用 `this`，所以实际上是 3 个参数，占用最后 3 个寄存器。而剩余的开头的寄存器就是局部变量寄存器，在 `add()` 方法中只有一个局部变量寄存器，用于存储 `a+b` 的值，就是第一个寄存器。下面就来看看 `v 命名法` 和 `p 命名法` 分别是如何给这 4 个寄存器命名的。

#### v 命名法

v 命名法其实很简单，就是上面说的 `v0 - v65535`。不管是参数寄存器，还是局部变量寄存器，一律以 `v` 开头。在 `add()` 函数中，4 个寄存器命名如下：

* `v0 :` 局部变量寄存器，存储 a+b 的值
* `v1 :` 当前引用 this
* `v2 :` 参数寄存器，存储 a 的值
* `v3 :` 参数寄存器，存储 b 的值

#### p 命名法

p 命名法针对参数寄存器进行了优化，参数寄存器的命名从 `p0` 开始，使得局部变量寄存器和参数寄存器得以很容易的进行区分。smali 语法中就是用了 p 命名法。我们来看下 `add()` 方法的 smali 代码：

```smali
.method public add(II)I
    .registers 4
    .param p1, "a"    # I
    .param p2, "b"    # I

    .prologue
    .line 6
    add-int v0, p1, p2

    .line 7
    .local v0, "c":I
    return v0
.end method
```

这样就很清晰了，4 个寄存器命名如下所示：

* `v0 :` 局部变量寄存器，存储 a+b 的值
* `p0 :` 当前引用 this
* `p1 :` 参数寄存器，存储 a 的值
* `p2 :` 参数寄存器，存储 b 的值

`p 命名法` 更加已读，一般都是使用 p 命名法。

### Dalvik 描述符

在更深入的了解 Dalvik 字节码前，先来看一下 Dalvik 是如何描述字段和方法的，这也有助于我们阅读 smali 代码。

#### 类型描述符

Dalvik 字节码中只有两种类型，基本类型和引用类型。除了对象和数组以外，其他的所有 Java 类型都是基本类型。这和 JVM 的类型描述符是基本一致的。基本类型都是使用单个字母来表示。数组类型使用 `[` 表示。除数组以外的引用类型使用 `L` 加上全限定名表示。如下表所示：

| 类型描述符 | 类型 |
| --- | --- |
| v | void，只用于返回值类型|
| Z | boolean |
| B | byte |
| S | short |
| C | char |
| I | int |
| J | long |
| F | float |
| D | double |
| L | 对象类型 |
| [ | 数组 |

基本类型都很简单，就不多说了，下面举一个引用类型的例子。例如 `String` 对象，其全限定名是 `java/lang/String;`，在 Dalvik 中就表示为 `Ljava/lang/String;`。对于数组，又可以分为基本类型数组和引用类型数组，其格式都是 `[` 加上类型描述符。`int[]` 就是 `[I`，`String[]` 就是 `[java/lang/String;`。多维数组就是多个 `[`，例如 `int[][]` 就是 `[[I`。

#### 字段

字段的表示统一用如下格式：

```
类型;->字段名称：类型描述符
```

比如一个 `com.test.Test` 类中的一个 `String` 类型的 `name` 字段，在 Dalvik 中就可表示为：

```
Lcom/test/Test;->name:Ljava/lang/String
```

#### 方法

方法的描述和字段的描述有一些类似，区别在于方法多了一个返回值的描述，其基本格式如下：

```
类型;->方法名(参数类型描述符)返回值类型描述符
```

以 `com.test.Test` 类中的 `add()` 方法为例，就是上面用到的两数相加的函数，其在 Dalvik 中描述为：

```
Lcom/test/Test;->add(II)I
```

`add(II)` 中的两个 I 表示两个 int 类型参数，后面跟的一个 I 表示返回值类型是 int。

### Dalvik 指令集

有了上面的知识储备之后，就可以具体的学习 Dalvik 指令集了。除了之前介绍过的官方文档和 AOSP 中关于 Dalvik 指令集的整理，我个人经常阅读的，还有一份国外开发者整理的 [Dalvik Opcodes](http://pallergabor.uw.hu/androidblog/dalvik_opcodes.html)，都是很好的学习资料。我也会基于此版本整理一份完整的中文版 Dalvik 操作码，可能还需要一段时间才能整理出来，到时候会开源出来。

下面简单整理一下 Dalvik 指令集。

#### 空指令

语法 | 说明
 --- | ---
 nop | 空指令，通常用于对齐

#### 数据操作指令

 语法 | 说明
 --- | ---
 move vA, vB | 将 vB(4 位) 寄存器的值赋给 vA(4 位) 寄存器
 move/from vAA, vBBBB | 将 vBBBB(16 位) 寄存器的值赋给 vAA(8 位) 寄存器
 move-object vA, vB |  将 vB(4 位) 寄存器存储的对象赋给 vA(4 位) 寄存器
 move-object/from16 vAA, vBBBB | 将 vBBBB(16 位) 寄存器存储的对象赋给 vAA(8 位) 寄存器
 move-result vAA | 将最新的 invoke-kind 的单字非对象结果移到指定的寄存器 vAA 中
 move-result-wide vAA | 将最新的 invoke-kind 的双字非对象结果移到指定的寄存器 vAA 中
 move-result-object vAA | 将最新的 invoke-kind 的对象结果移到指定的寄存器 vAA 中
 move-exception | 将刚刚捕获的异常保存到给定寄存器中

#### 返回指令

语法 | 说明
--- | ---
return-void | 返回 void
return-vAA | 返回一个 32 位非对象类型
return-wide vAA | 返回一个 64 位非对象类型
return-object vAA | 返回一个对象类型

#### 数据定义指令

语法 | 说明
--- | ---
const/4 vA, #+b | 将给定的字面值（符号扩展为 32 位）移到指定的寄存器 vA 中
const vAA, #+BBBBBBBB | 将给定的字面值移到指定的寄存器 vAA 中
const/high16 vAA, #+BBBB0000 | 将给定的字面值（右零扩展为 32 位）移到指定的寄存器 vAA 中
const-wide/16 vAA, #+BBBB | 将给定的字面值（符号扩展为 64 位）移到指定的寄存器对 vAA 中
const-wide vAA, #+BBBBBBBBBBBBBBBB | 将给定的字面值移到指定的寄存器对 vAA 中
const-string vAA, string@BBBB | 将通过给定的索引获取的字符串引用移到指定的寄存器 vAA 中
const-class vAA, type@BBBB | 将通过给定的索引获取的类引用移到指定的寄存器 vAA 中

#### 锁指令

语法 | 说明
--- | ---
monitor-enter vAA | 获取指定对象的互斥锁
monitor-exit vAA | 释放指定对象的互斥锁

#### 类型判断指令

语法 | 说明
--- | ---
check-cast vAA, type@BBBB | 如果给定寄存器 vAA 中的引用不能转型为指定的类型，则抛出 ClassCastException
instance-of vA, vB, type@CCCC | 如果指定的引用是给定类型的实例，则为给定目标寄存器赋值 1，否则赋值
new-instance vAA, type@BBBB | 根据指定的类型构造新实例，并将对该新实例的引用存储到目标寄存器 vAA 中

#### 数组操作指令

语法 | 说明
--- | ---
array-length vA, vB | 获取寄存器 vB 中数组的长度，并存入寄存器 vA
new-array vA, vB, type@CCCC | 构造指令类型(type@CCCC) 和指定大小(vB) 的数组，并赋给 寄存器 vA
filled-new-array {vC, vD, vE, vF, vG}, type@BBBB | 构造指令类型(type@BBBB) 和指定大小的数组，并填充内容

#### 异常指令

语法 | 说明
--- | ---
throw vAA | 抛出 vAA 寄存器指定的异常

#### 跳转指令

语法 | 说明
--- | ---
goto +AA | 无条件跳转至指定偏移处，偏移量为 AA
if-test vA, vB, +CCCC | 如果两个给定寄存器的值比较结果符合预期，则跳转到偏移量 CCCC 处

同 `if-test`一样，还有 `if-eq` `if-ne` `if-lt` `if-ge` `if-gt` `if-le` `if-testz` `if-eqz` `if-nez` `if-ltz` `if-gez` `if-ltz` `if-gez` `if-gtz` `if-lez`，这些指令格式都是一致的。

#### 字段操作指令

字段操作指令分为两类，分别是对于普通字段和静态字段的操作。

##### 普通字段

语法 | 说明
--- | ---
iinstanceop vA, vB, field@CCCC | 对已标识的字段执行已确定的对象实例字段运算，并将结果加载或存储到值寄存器中

针对不同类型的普通字段，有如下命令：

```
iget、iget-wide、iget-object、iget-boolean、iget-byte、iget-char、iget-short
iput、iput-wide、iput-object、iput-boolean、iput-byte、iput-char、iput-short
```

##### 静态字段

语法 | 说明
--- | ---
sstaticop vAA, field@BBBB | 对已标识的静态字段执行已确定的对象静态字段运算，并将结果加载或存储到值寄存器中

针对不同类型的静态字段，有如下命令：

```
sget、sget-wide、sget-object、sget-boolean、sget-byte、sget-char、sget-short
sput、sput-wide、sput-object、sput-boolean、sput-byte、sput-char、sput-short
```

#### 方法调用指令

方法调用指令的格式为 `invoke-kind {vC, vD, vE, vF, vG}, meth@BBBB`, 具体的有如下指令：

语法 | 说明
--- | ---
invoke-virtual | 调用正常的虚方法（该方法不是 private、static 或 final，也不是构造函数）
invoke-super | 调用父类方法
invoke-direct | 调用非 static 直接方法（也就是说，本质上不可覆盖的实例方法，即 private 实例方法或构造函数）
invoke-static | 调用 static 方法
invoke-interface | 调用实例的接口方法

#### 数据运算和转换指令

数据运算指令和数据转换指令都比较简单，且数量很多，这里就不浪费篇幅来写出来了，感兴趣的同学可以查阅资料看一下。后续我也会开源一个完整版的 Dalvik 指令集的表格。

## 总结

本文介绍了 Dalvik 虚拟机的相关知识，比较了 Dalvik 虚拟机和 JVM，后续着重介绍了 Dalvik 指令集。看懂看会 Dalvik 指令对我们做逆向是很有帮助的，毕竟想要修改程序逻辑，大部分时间就是在和 smali 代码打交道。而 smali 代码就是基于 Dalvik 指令集的。如果你阅读过 smali 代码，应该会对上面提到的 Dalvik 指令很熟悉。

最后再放一下 Android 逆向笔记系列的其他文章，按顺序阅读效果更佳！

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
>
> [Android 逆向笔记 —— ARSC 文件格式解析](https://juejin.im/post/5ce7d488f265da1b5d5785f2)
>
> [Android 逆向笔记 —— 一个简单 CrackMe 的逆向总结](https://juejin.im/post/5ce52ed951882532df70ab58)


***
> 文章首发微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解。
>
> 更多逆向相关知识，扫码关注我吧！

![](https://user-gold-cdn.xitu.io/2019/4/27/16a5f352eab602c4?w=2800&h=800&f=jpeg&s=178470)
