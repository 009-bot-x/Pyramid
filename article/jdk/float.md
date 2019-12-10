> 文中相关源码：
>
> [Float.java](https://github.com/lulululbj/jdk8u/blob/master/jdk/src/share/classes/java/lang/Float.java)
>
> [Float.c](https://github.com/lulululbj/jdk8u/blob/master/jdk/src/share/native/java/lang/Float.c)

</br>
</br>


```
0.3f - 0.2f = ?
```
相信很多人会不假思索的填上 `0.1f`，那么，打开 `IDEA`，默默的执行一下：

```
0.10000001
```

如果你对这个答案抱有疑问，那么在阅读 `Float` 源码之前，我们先来看一下 `Float` 在内存中是如何表示的。

从熟悉的十进制浮点数说起，以 `12.34` 为例，显然下面这个等式是成立的：

```java
12.34 = 1 * 10^1 + 2 * 10^0 + 3 * 10^-1 + 4 * 10^-2
```

同样的，对于二进制浮点数，也有如下等式，这里以 `10.11b`(代码块里面好像打不了下标，本文中以 `b` 结尾的均表示二进制浮点数)为例：

```java
10.11 b = 1 * 2^1 + 0 * 2^0 + 1 * 2^-1 + 1 * 2^-2
        = 2 + 1/2 + 1/4
        = 2.75
```

这样，二进制浮点数 `10.11b` 就转换成了十进制浮点数 `2.75`。

再看一个十进制小数 `1.75` 转换为二进制小数的例子：

```java
1.75 = 1 + 3/4
     = 7/4
     = 7 * 2^-2
```

`7` 的二进制表示为 `111`，`* 2^-2` 表示将小数点左移两位，得到 `1.11`。所以，`1.75 = 1.11b`。

下图列举一些常见小数的值：

| 二进制 | x^y    | 十进制 |
| ------ | -----  | ------ |
| 0.1    | 2 ^ -1 | 0.5    |
| 0.01   | 2 ^ -2 | 0.25   |
| 0.001  | 2 ^ -3 | 0.125   |
| 0.0001 | 2 ^ -4 | 0.0625   |
| 0.00001 | 2 ^ -5 | 0.03125   |


你发现问题的所在了吗？我们再回到 `0.3f - 0.2f` 的问题上。不管是整数还是浮点数，最终在内存中都是以二进制形式存在的，那么 `0.3f` 如何以二进制表示呢？显而易见，没有办法以 `x * 2^y` 的形式来准确表示 `0.3f`，也就是说，我们并不能将 `0.3f` 准确的表示为一个二进制小数，只能近似的表示它，增加二进制的长度可以提高精确度。同样，对于 `0.2f`，我们也没法准确的表示为二进制小数，所以最后的计算结果才不是 `0.1f`。

最后再看一个减法，`0.5f - 0.25f = ?`。答案是 `0.25f`，我想这时候你应该不会再答错了。因为 `0.5f` 和 `0.25f` 都可以准确的表示为二进制小数，分别是 `0.1b` 和 `0.01b`。

说到这里，其实我们还是不了解 `float` 在内存中到底是什么样的？`int` 型的 `1`, 内存中就是 `00000000000000000000000000000001`，那么 `0.75f` 呢？关于浮点数，有一个广泛使用的运算标准，叫做 [IEEE 754-1985](https://en.wikipedia.org/wiki/IEEE_754-1985)，全称 **IEEE 二进制浮点数算数标准**, 由 IEEE（电气和电子工程师协会）指定，所有的计算机都支持 IEEE 浮点数标准。

本文后面都只针对 32 位单精度浮点数，对应 Java 中的 `Float`。先来看维基百科上的一张图：



![](https://user-gold-cdn.xitu.io/2019/3/6/16953995094a46d1?w=618&h=125&f=png&s=4662)

这张图描述了单精度浮点数在内存中具体的二进制表示方法：

* sign : 符号位，`1` 位 。`0` 表示正数， `1` 表示负数。用 `s` 表示
* exponent ： 阶码域，`8` 位。用 `E` 表示，通常 `E = exponent - 127`,`exponent` 为无符号数
* fraction : 尾数域，`23` 位。用 `M` 表示，通常 `M = 1.fraction`

通常情况下，一个浮点数可以表示如下：

```
V = (-1)^s * M * 2^E
```

以上图中的 `0.15625f` 为例。符号位为 `0`，表示为正数。阶码域为 `1111100`，等于十进制 `124`，则 阶码 `E = 124 - 127 = -3`。尾数域为 `01`，则 `M = 1.01`。代入公式得：

```
V = (-1)^0 * 1.01 * 2^-3 = 0.00101 b = 0.15625
```

注意，`* 2^-3`，等价于将小数点左移三位。

对于双精度浮点数来说，`exponent` 是 `11` 位，`fraction` 是 `52` 位。

关于浮点数的详细介绍可以阅读 `《深入理解计算机系统》` 第二章第四节的相关内容。下面就进入 `Float` 的源码部分。

## 类声明

```java
public final class Float extends Number implements Comparable<Float>{}
```

不可变类，继承了 `Number` 类，实现了 `Comparable` 接口。

## 字段

```java
private final float value;
private static final long serialVersionUID = -2671257302660747028L;
public static final Class<Float> TYPE = (Class<Float>) Class.getPrimitiveClass("float");
```

`final` 修饰的 `value` 字段保证其不可变性，`value` 也是 `Float` 类所包装的浮点数。

```java
// 0 11111111 00000000000000000000000
public static final float POSITIVE_INFINITY = 1.0f / 0.0f;
// 1 11111111 00000000000000000000000
public static final float NEGATIVE_INFINITY = -1.0f / 0.0f;
```
正无穷和负无穷。阶码域都为 `1`，尾数域都为 `0`。

```java
// 0 11111111 10000000000000000000000
public static final float NaN = 0.0f / 0.0f;
```
`Not a number`，非数字。阶码域都为 `1`，尾数域不全为 `0`。

```java
// 0 11111110 11111111111111111111111
public static final float MAX_VALUE = 0x1.fffffeP+127f;
```

最大值。阶码域为 `11111110`，即 `127`。按公式计算，`V = 1.11...1 * 2^127`。

```java
/*
 * 0 00000001 00000000000000000000000
 * 最小的规格化数（正数）
 */
public static final float MIN_NORMAL = 0x1.0p-126f; // 1.17549435E-38f

/*
 * 0 00000000 00000000000000000000001
 * 最小的非规格化数（正数）
 */
public static final float MIN_VALUE = 0x0.000002P-126f;
```

这里出现了两个新名词，`规格化数` 和 `非规格化数`。上文中一直在说 `通常情况下`，这个通常情况指的就是 `规格化数`。那么什么是规格化数呢？阶码域  `exponent != 0 && exponent != 255` ，即阶码域即不全为 `0`，也不全为 `1`，这样的浮点数就成为规格化数。对于规格化数，有如下规则：

```
E = exponent - 127
M = 1.fraction
V = (-1)^s * M * 2^E
```

阶码域全为 `0` 的浮点数是 `非规格化数`。对于非规格化数，对应规则也发生了改变：

```
E = 1 - 127 = -126
M = 0.fraction
V = (-1)^s * M * 2^E
```

浮点数的计算方法并没有发生改变，阶码 `E` 和尾数 `M`的计算方法与规格化数不同了。非规格化数有两个用途，第一，它可以表示 `0`。由于规格化的尾数域 `M = 1.fraction`，所以规格化数是没法表示零值的。除了符号位外，其他域全为 `0`，就表示 `0.0f`。根据符号位的不同，还有 `+0.0f` 和 `-0.0f`，它们被认为是不同的。第二，非规格数的存在使得浮点数可能表示的数值分布更加均匀的接近于 `0.0`，它可以表示那些非常接近于 0 的数。

```java
public static final int MAX_EXPONENT = 127; // 指数域（阶码）最大值
public static final int MIN_EXPONENT = -126; // 指数域（阶码）最小值
public static final int SIZE = 32; // float 占 32 bits
public static final int BYTES = SIZE / Byte.SIZE; // float 占 4 bytes
```

## 构造函数

```java
public Float(float value) {
     this.value = value;
}

public Float(double value) {
    this.value = (float)value;
}

public Float(String s) throws NumberFormatException {
    value = parseFloat(s);
}
```

`Float` 有三个构造函数。第一个直接传入 `float` 值。第二个传入 `double` 值，再强转 `float`。第三个传入 `String`，调用 `parseFloat()` 函数转换成 `float`。下面就来看看这个 `parseFloat` 函数。

## 方法

### parseFloat(String)

```java
public static float parseFloat(String s) throws NumberFormatException {
   return FloatingDecimal.parseFloat(s);
}
```

调用了 `FloatDecimal` 的 `parseFloat(String)` 方法。这个方法源码相当的长，逻辑也比较复杂，我也只是大概看了一下流程。我就不贴源码了，捋一下大致流程：

* 首先取出符号位，判断正数还是负数
* 判断是否为 `NaN`
* 判断是否为 `Infinity`
* 判断是否是以 `0x` 或 `0X` 开头的十六进制浮点数。若是，调用 `parseHexString()` 方法处理
* 跳过开头的无效的 `0`
* 循环取出各位数字。注意若包含 `e` 或者 `E`,需要注意科学计数法的处理
* 根据取得的字符数组等信息构建 `ASCIIToBinaryBuffer` 对象，调用其 `floatValue()` 方法，获取最终结果

这块源码看的一知半解，有功夫再慢慢跟进。`String` 转 `float` 的方法除此之外，还有 `valueOf()` 方法。

### valueOf(String)

```java
    public static Float valueOf(String s) throws NumberFormatException {
        return new Float(parseFloat(s));
    }
```

没啥好说的，还是调用 `parseFloat()` 方法。

下面看一下 `float` 转 `String` 的相关方法。

### toString()

```java
public String toString() {
    return Float.toString(value);
}

public static String toString(float f) {
    return FloatingDecimal.toJavaFormatString(f);
}
```

最终调用了 `FloatDecimal` 的 `toJavaFormatString()` 方法。这个方法也是源码相当长，逻辑很复杂。首先会通过 `floatToRawIntBits()` 方法转换成其符合 `IEEE 754` 标准的二进制形式对应的 `int` 值，再转换为相应的十进制字符串。

最后看一下 `Float` 中提供的其他一些方法。

### isNaN()

```java
public static boolean isNaN(float v) {
    return (v != v);
}
```

这个判断很有意思，`v != v`。据此我们可以推断出，对于任意不是 `NaN` 的 `v` ，必定满足 `v == v`。对于为 `NaN` 的 `v`，必定满足 `v != v`。

### isInfinite()

```java
public boolean isInfinite() {
    return isInfinite(value);
}

public static boolean isInfinite(float v) {
    return (v == POSITIVE_INFINITY) || (v == NEGATIVE_INFINITY);
}
```

判断是否为正无穷或负无穷。

### isFinite()

```java
public static boolean isFinite(float f) {
    return Math.abs(f) <= FloatConsts.MAX_VALUE;
}
```

判断浮点数是否是一个有限值。

### Number 接口方法

```java
public byte byteValue() {   return (byte)value;     }
public short shortValue() { return (short)value;    }
public int intValue() { return (int)value;      }
public long longValue() {   return (long)value; }
public float floatValue() { return value;   }
public double doubleValue() {   return (double)value;   }
```

### floatToRawIntBits(float)

```java
public static native int floatToRawIntBits(float value);
```

这是一个 `native` 方法，将 `float` 浮点数转换为其 `IEEE 754` 标准二进制形式对应的 `int` 值。因为 `float` 和 `int` 都是占 32 位，所以每一个 float 总有对应的 int 值。具体实现在 `native/java/lang/Float.c` 中：

```c
JNIEXPORT jint JNICALL
Java_java_lang_Float_floatToRawIntBits(JNIEnv *env, jclass unused, jfloat v)
{
    union {
        int i;
        float f;
    } u;
    u.f = (float)v;
    return (jint)u.i;
}
```

`union` 是一种数据结构，它能在同一个内存空间中储存不同的数据类型，也就是说同一块内存，它可以表示  `float` , 也可以表示 `int`。通过 `union` 将 `float` 转换为其二进制对应的 `int` 值。

### floatToIntBits(float)

```java
public static int floatToIntBits(float value) {
    int result = floatToRawIntBits(value);
    // Check for NaN based on values of bit fields, maximum
    // exponent and nonzero significand.
    if ( ((result & FloatConsts.EXP_BIT_MASK) ==
          FloatConsts.EXP_BIT_MASK) &&
         (result & FloatConsts.SIGNIF_BIT_MASK) != 0)
        result = 0x7fc00000;
    return result;
}
```

基本等同于 `floatToRawIntBits()` 方法，区别在于这里对于 `NaN` 作了检测，如果结果为 `NaN`, 直接返回 `0x7fc00000`，也就是 Java 中的 `Float.NaN`。乍看一下，这不是在多此一举吗？如果是 `NaN` 就直接返回 `NaN`。还记得前面对 `NaN` 的说明吗，**阶码域都为 `1`，尾数域不全为 `0`**，所以 `IEEE 754` 中的 `NaN` 并不是一个固定的值，而是一个值域，但是在 Java 中将 `Float.NaN` 定义为了 `0x7fc00000`，相应二进制为 `0 11111111 10000000000000000000000`。所以方法参数中的 `NaN` 值并不一定就是 `0x7fc00000`。从检测 `NaN` 的条件中也可以看出一二：

```java
((result & FloatConsts.EXP_BIT_MASK) == FloatConsts.EXP_BIT_MASK) &&
(result & FloatConsts.SIGNIF_BIT_MASK) != 0
```

前半段是检测阶码域的。`FloatConsts.EXP_BIT_MASK` 值为 `0x7F800000`, 二进制为 `0 11111111 00000000000000000000000`,若满足 `(result & FloatConsts.EXP_BIT_MASK) == FloatConsts.EXP_BIT_MASK`，则 `result` 阶码域必定全为 `1`。

后半段是检测尾数域的。`FloatConsts.SIGNIF_BIT_MASK` 值为 `0x007FFFFF`，二进制为 `0 00000000 11111111111111111111111`，若要满足 `(result & FloatConsts.SIGNIF_BIT_MASK) != 0`, 则 `result` 尾数域不全为 `0` 即可。

根据这里两个检测条件也可以知道这里的 `NaN` 并不是一个固定的值。但是 `Float.NaN` 又是一个固定的值，那么如何获取其他不同的 `NaN` 呢？答案就是 `intBitsToFloat(int)` 方法。

### intBitsToFloat(int)

```java
public static native float intBitsToFloat(int bits);

Java_java_lang_Float_intBitsToFloat(JNIEnv *env, jclass unused, jint v)
{
    union {
        int i;
        float f;
    } u;
    u.i = (long)v;
    return (jfloat)u.f;
}
```

`native` 方法，也是通过联合体 `union` 来实现的。只要参数中的 `int` 值满足 `IEEE 754` 对于 `NaN` 的标准，就可以产生值不为 `Float.NaN` 的 `NaN` 值了。

### hashCode()

```java
@Override
public int hashCode() {
    return Float.hashCode(value);
}

public static int hashCode(float value) {
    return floatToIntBits(value);
}
```

`hashCode()` 函数直接调用 `floatToIntBits()` 方法，返回其二进制对应的 `int` 值。

### equals()

```java
public boolean equals(Object obj) {
    return (obj instanceof Float)
           && (floatToIntBits(((Float)obj).value) == floatToIntBits(value));
}
```

`equals` 的条件是其 `IEEE 754` 标准的二进制形式相等。

## 总结

`Float` 就说到这里了。这篇源码解释不是很多，主要说明了 `Float` 在内存中的二进制形式，也就是 `IEEE 754` 标准。了解了 `IEEE 754`，对于浮点数也就了然于心了。最后再推荐一下 **《深入理解计算机系统》** 中 `2.4` 节关于浮点数的介绍。

> 文章同步更新于微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解，欢迎关注！

![](https://user-gold-cdn.xitu.io/2019/3/30/169cf046d9579e78?w=258&h=258&f=jpeg&s=27711)
