> 文中相关源码：
>
> [Long.java](https://github.com/lulululbj/jdk8u/blob/master/jdk/src/share/classes/java/lang/Long.java)

上一篇文章 [走进 JDK 之 Integer](https://juejin.im/post/5c76ad1ae51d4572c95835d0) 解析了 `Integer.java`，而 `Long.java` 和 `Integer.java` 的源码结构几乎是一模一样的，所以这篇文章会写的比较简略，没有细读过 `Integer.java` 源码的可以先看一下我的上一篇文章。这里就简单介绍一下 `Long` 以及源码中和 `Integer` 的细微区别。

## 类声明

```java
public final class Long extends Number implements Comparable<Long>{}
```

和 `Integer` 一样，不可变类，继承了 `Number` 类，实现了 `Comparable` 接口。

## 字段

```java
// long 最小值，-2^63
public static final long MIN_VALUE = 0x8000000000000000L;
// long 最大值，-2^63-1
public static final long MAX_VALUE = 0x7fffffffffffffffL;
public static final Class<Long> TYPE = (Class<Long>) Class.getPrimitiveClass("long");
private final long value;
// 以二进制补码形式表示 long 值所需的比特数
public static final int SIZE = 64;
// 以二进制补码形式表示 long 值所需的字节数,恒为 8
public static final int BYTES = SIZE / Byte.SIZE;
private static final long serialVersionUID = 4290774380558885855L;
```

## 构造函数

```java
public Long(long value) {
    this.value = value;
}

public Long(String s) throws NumberFormatException {
    this.value = parseLong(s, 10);
}
```

构造函数和 `Integer` 一样也是两个。第一个参数为 `long` 值。第二个参数为 `String` 字符串，通过 `parseLong()` 方法转换为 `long` 值。

## 方法

`Long.java` 中的方法实现几乎和 `Integer.java` 一致，这里不再一一分析源码了，仅列举一下出现的方法，也算对上一篇文章的总结。

### String 转 long

```java
public static int parseLong(String s)
public static long parseLong(String s, int radix)
public static long parseUnsignedLong(String s)
public static long parseUnsignedLong(String s, int radix)
public static Long decode(String nm)
public static Long valueOf(String s)
public static Long valueOf(String s, int radix)
public static Long getLong(String nm)
public static Long getLong(String nm, long val)
public static Long getLong(String nm, Long val)
```

如果你还记得 `IntegerCache` 的话，没错，同样也有一个 `LongCache` :

```java
private static class LongCache {
    private LongCache(){}

    static final Long cache[] = new Long[-(-128) + 127 + 1];

    static {
        for(int i = 0; i < cache.length; i++)
            cache[i] = new Long(i - 128);
    }
}
```

和 `IntegerCache` 一样，它也是缓存了 `-128` 到 `127`。

### int 转 String

```java
public static String toString(long i)
public static String toString(long i, int radix)
public static String toUnsignedString(long i)
public static String toUnsignedString(long i, int radix)
```

#### toString()

首先看一下 `toString(long i)` 方法：

```java
public static String toString(long i) {
    if (i == Long.MIN_VALUE)
        return "-9223372036854775808";
    int size = (i < 0) ? stringSize(-i) + 1 : stringSize(i);
    char[] buf = new char[size];
    getChars(i, size, buf);
    return new String(buf, true);
}
```

整体套路和 `Integer` 是一样的，但在 `stringSize()` 和 `getChars()` 的具体实现上有一些细微的不同。

```java
static int stringSize(long x) {
    long p = 10;
    for (int i=1; i<19; i++) {
        if (x < p)
            return i;
        p = 10*p;
    }
    return 19;
}
```

`Long.MAX_VALUE` 是 19 位数字，所以最多不会循环超过 19 次。原理和 `Integer` 的 `stringSize()` 方法是一样的，小于 `10` 就是 `1` 位数，小于 `100` 就是 `2` 位数..... 。还记得 `Integer` 中是怎么实现的吗？

```java
final static int [] sizeTable = { 9, 99, 999, 9999, 99999, 999999, 9999999,
                                  99999999, 999999999, Integer.MAX_VALUE };

// Requires positive x
static int stringSize(int x) {
    for (int i=0; ; i++)
        if (x <= sizeTable[i])
            return i+1;
}
```

使用了一个 `10` 个元素的数组。如果 `Long` 也使用同样的方法，那么就需要一个 `20` 个元素的数组。JDK 开发者可能是站在考虑内存使用的角度，并没有使用这种方法。

除了 `stringSize()` 外，`getChars()` 的实现也略有不同。`Long` 中的 `getChars()` 如下所示：

```java
static void getChars(long i, int index, char[] buf) {
    long q;
    int r;
    int charPos = index;
    char sign = 0;

    if (i < 0) {
        sign = '-';
        i = -i;
    }

    // Get 2 digits/iteration using longs until quotient fits into an int
    while (i > Integer.MAX_VALUE) {
        q = i / 100;
        // really: r = i - (q * 100);
        r = (int)(i - ((q << 6) + (q << 5) + (q << 2)));
        i = q;
        buf[--charPos] = Integer.DigitOnes[r];
        buf[--charPos] = Integer.DigitTens[r];
    }

    // Get 2 digits/iteration using ints
    int q2;
    int i2 = (int)i;
    while (i2 >= 65536) {
        q2 = i2 / 100;
        // really: r = i2 - (q * 100);
        r = i2 - ((q2 << 6) + (q2 << 5) + (q2 << 2));
        i2 = q2;
        buf[--charPos] = Integer.DigitOnes[r];
        buf[--charPos] = Integer.DigitTens[r];
    }

    // Fall thru to fast mode for smaller numbers
    // assert(i2 <= 65536, i2);
    for (;;) {
        q2 = (i2 * 52429) >>> (16+3);
        r = i2 - ((q2 << 3) + (q2 << 1));  // r = i2-(q2*10) ...
        buf[--charPos] = Integer.digits[r];
        i2 = q2;
        if (i2 == 0) break;
    }
    if (sign != 0) {
        buf[--charPos] = sign;
    }
}
```

这里分隔了三段循环体，分别以 `Integer.MAX_VALUE` 和 `65536` 为分界线，而 `Integer` 仅以 `65536` 为界分了两段循环体。我们先不看这多出来的一段，来看一下 `Long` 里面第二段循环体的实现：

```java
// Get 2 digits/iteration using ints
int q2;
int i2 = (int)i;
while (i2 >= 65536) {
    q2 = i2 / 100;
    // really: r = i2 - (q * 100);
    r = i2 - ((q2 << 6) + (q2 << 5) + (q2 << 2));
    i2 = q2;
    buf[--charPos] = Integer.DigitOnes[r];
    buf[--charPos] = Integer.DigitTens[r];
}
```

看似和 `Integer` 一样，但别忘了这里是 `Long`。`Long` 占用 `8` 个字节，而 `int` 只占用 `4`个字节。所以，当 `long` 型的值小于 `Integer.MAX_VALUE` 时，可以强转为 `int`, 从而节省内存。看到这，你应该明白第一个循环体的含义了，当 `i > Integer.MAX_VALUE` 时，强转 `int` 会溢出，就只能当做 `long` 处理了。

### toUnsignedString()

`Integer.toUnsignedString(int,int)`

```java
public static String toUnsignedString(int i, int radix) {
    return Long.toUnsignedString(toUnsignedLong(i), radix);
}
```

上面是 `Integer` 中 `toUnsignedString()` 的实现, 直接调用 `Long` 的相应方法。来看一下 `Long` 是如何实现这个方法的：

```java
public static String toUnsignedString(long i, int radix) {
    if (i >= 0)
        return toString(i, radix);
    else {
        switch (radix) {
        case 2:
            return toBinaryString(i);

        case 4:
            return toUnsignedString0(i, 2);

        case 8:
            return toOctalString(i);

        case 10:
            /*
             * We can get the effect of an unsigned division by 10
             * on a long value by first shifting right, yielding a
             * positive value, and then dividing by 5.  This
             * allows the last digit and preceding digits to be
             * isolated more quickly than by an initial conversion
             * to BigInteger.
             */
            long quot = (i >>> 1) / 5;
            long rem = i - quot * 10;
            return toString(quot) + rem;

        case 16:
            return toHexString(i);

        case 32:
            return toUnsignedString0(i, 5);

        default:
            return toUnsignedBigInteger(i).toString(radix);
        }
    }
}
```

把 `long` 型数值转换为无符号字符串。`long` 的最大值是 `2^63-1`，而 `unsigned long` 最大值为 `2^64-1`。所以正整数对于 `toUnsignedString()` 方法来说，仍可以当做有符号数处理，直接调用 `toString(long i,int radix)` 处理。

#### 二、四、八、十六进制

对于负数来说，根据进制的不同，采取不同的处理。`radix` 为 `2` 、`4` 、`8` 、`16` 、`32`时，分别调用了 `toBinaryString()` 、`toUnsignedString0()` 、`toOctalString()` 、`toHexString()` 、`toUnsignedString0()` 方法，其实这些方法最后都调用了 `toUnsignedString0(long i,int shift)` 方法，其中 `raidx = 1 << shift` ：

```java
static String toUnsignedString0(long val, int shift) {
    // assert shift > 0 && shift <=5 : "Illegal shift value";
    int mag = Long.SIZE - Long.numberOfLeadingZeros(val);
    int chars = Math.max(((mag + (shift - 1)) / shift), 1);
    char[] buf = new char[chars];

    formatUnsignedLong(val, shift, buf, 0, chars);
    return new String(buf, true);
}
```

逐行分析一下：

```java
int mag = Long.SIZE - Long.numberOfLeadingZeros(val);
```

`mag` 表示该数字表示为二进制实际需要的位数（去除高位的 0）。`numberOfLeadingZeros()` 方法之前说过，表示该数字以二进制补码形式表示时最高位 1 前面的位数。比如 `16L`，二进制为 `0001 0000`, 最高位 1 在第 `4` 位，则前面有 `59` 个 0，表示 `16L` 实际需要的位数是 `Long.SIZE - 59 = 5`，即 `10000`。当然，对于负数来说，符号位为 `1`，所以永远需要 `64` 位来表示。

```java
int chars = Math.max(((mag + (shift - 1)) / shift), 1);
```

根据 `mag` 和 `shift` 获取要表示的字符串的字符数。仍以上面的 `16L` 为例，`mag` 等于 5，若 `shift` 为 `1`，计算出 `chars` 等于 5。若 `shift` 等于 `4`，即以十六进制表示，计算出 `chars` 等于 2，因为 `16` 对应的十六进制是 `0x10`，表示为字符的话只需要两个字符。

```java
static int formatUnsignedLong(long val, int shift, char[] buf, int offset, int len) {
   int charPos = len;
   int radix = 1 << shift; // 进制
   int mask = radix - 1; // 掩码，二进制都为 1
   do {
       buf[offset + --charPos] = Integer.digits[((int) val) & mask]; // 取出 val 中对应进制的最后一位
       val >>>= shift; // 无符号右移 shift，高位补 0，移走上一步已经取出的最后一位数据
   } while (val != 0 && charPos > 0);

   return charPos;
}
```

逐行分析：

```java
int charPos = len;
```

`charPos` 表示填充字符在数组中的位置。初始值为 `len` ，不难想到还是从数组尾部开始填充的，所以循环体中的内容肯定是取出 `unsigned val` 的最后一位数字。

```java
int radix = 1 << shift; // 进制
int mask = radix - 1; // 掩码，二进制都为 1
```

这里先记住一点，掩码 `mask` 的二进制有效数字都是 `1`。

```
二进制   ： mask = 1 = 0b0001
八进制   ： mask = 7 = 0b0111
十六进制  ： mask = 15 = 0b1111
```

最后看循环体中的内容：

```java
do {
    buf[offset + --charPos] = Integer.digits[((int) val) & mask]; // 取出 val 中对应进制的最后一位
    val >>>= shift; // 无符号右移 shift，高位补 0，移走上一步已经取出的最后一位数据
} while (val != 0 && charPos > 0);
```

由于掩码 `mask` 的特殊性，`((int) val) & mask` 必定是 `val` 转换成对应进制中的最后一位数字，将其塞入字符数组中。然后执行 `val >>>=shift`，无符号右移都是高位填 0，移动 `shift` 位，正好移除了对应进制的最后一位数。以此循环，直到全部填 0，字符数组也就填满了。

以 `31L` 为例，其二进制表示为 `0001 1111`,十六进制表示为 `0x1f`,将其转换为十六进制字符串应该为 `1f`，其中 `radix` 为 `16`，`shift` 为 `4`，`mask` 为 `15`。

* 第一次循环，`((int) val) & mask` 等于 `0001 1111 & 0000 1111`， 值为 `1111`，对应 `Integer.digits` 中字符为 `'f'`。然后将 `val` 无符号右移四位，得到新的值为 `0000 0001`，进入第二次循环。

* 第二次循环，`val` 与 `mask` 与运算，`0000 0001 & 0000 1111`,值为 `0001`,对应字符为 `'1'`。再将 `val` 无符号右移四位，其值为 `0`,结束循环。

这样就得到最后的无符号十六进制字符串为 `'1f'`。

#### 十进制

`toUnsignedString(long value,int radix)` 对十进制进行了单独处理：

```java
case 10:
    /*
     * We can get the effect of an unsigned division by 10
     * on a long value by first shifting right, yielding a
     * positive value, and then dividing by 5.  This
     * allows the last digit and preceding digits to be
     * isolated more quickly than by an initial conversion
     * to BigInteger.
     */
    long quot = (i >>> 1) / 5;
    long rem = i - quot * 10;
    return toString(quot) + rem;
```

注释的大概意思是，我们可以将数字先无符号右移一位得到一个正值，再除以 `5`, 以此来代替用无符号数除以 `10`。这样可以比通过初始化 `BigInteger` 更快的分离出最后一个数字。粗看代码，大概步骤应该是将 `i` 分成一个 `long` 值 `quot` 和最后一个数字 `rem`，对于 `quot` 其实是有符号的，直接调用 `toString()` 方法，最后和 `rem` 拼接起来即可。大致思路是这样，其中具体的位运算还没有看的太懂。

#### 其他进制

```java
default:
    return toUnsignedBigInteger(i).toString(radix);
```

对于其他进制，一律通过 `BigInteger` 进行处理，这里不展开分析，后面说道 `BigInteger` 时再说。

### 位运算

位运算和 `Integer` 是完全一致的。

```java
long highestOneBit(long i) : 返回以二进制补码形式，取左边最高位 1，后面全部填 0 表示的 long 值
long lowestOneBit(long i)  : 与 highestOneBit() 相反，取其二进制补码的右边最低位 1，其余填 0
int numberOfLeadingZeros(long i) : 返回左边最高位 1 之前的 0 的个数
int numberOfTrailingZeros(long i): 返回右边最低位 1 之后的 0 的个数
int bitCount(long i) : 二进制补码中 1 的个数
long rotateRight(long i, int distance) ： 将 i 的二进制补码循环右移 distance（注意与普通右移不同的是，右移的数字会移到最左边）
long rotateLeft(long i, int distance)  ： 与 rotateRight 相反
long reverse(long i) ： 反转二进制补码
int signum(long i)  ： 正数返回 1，负数返回 -1,0 返回 0
long reverseBytes(long i) ： 以字节为单位反转二进制补码
```

## 总结

复习了 `Intger` 中的内容，补充了 `Integer` 中没提到的 `toUnsignedString()` 方法。关于整数差不多就说到这了，后面有机会再分析一下 `BigInteger`。下篇文章来说说 `Float`，现在不妨思考这样一个问题，`0.3f - 0.2f` 等于多少，写下你的答案，再打开 IDEA 运行一下吧！

> 文章同步更新于微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解，欢迎关注！

![](https://user-gold-cdn.xitu.io/2019/3/30/169cf046d9579e78?w=258&h=258&f=jpeg&s=27711)
