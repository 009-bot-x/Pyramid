> [走进 JDK 之 String](https://juejin.im/post/5ca30c31f265da30c1724a04)
>
> [你并不了解 String](https://juejin.im/post/5ca5c51451882544114cdc95)

今天是 `String` 系列最后一篇了，字符串的拼接。日常开发中，字符串拼接是很常见的操作，一般常用的有以下几种：

* 直接使用 `+` 拼接
* 使用 `String` 的 `concat()` 方法
* 使用 `StringBuilder` 的 `append()` 方法
* 使用 `StringBuffer` 的 `append()` 方法

那么，这几种方法有什么不同呢？具体性能如何？下面进行一个简单的性能测试，代码如下：

```java
public class StringTest {

    public static void main(String[] args) {
        int count = 1000;
        String word = "Hello, ";
        StringBuilder builder = new StringBuilder("Hello,");
        StringBuffer buffer = new StringBuffer("Hello,");
        long start, end;

        start = System.currentTimeMillis();
        for (int i = 0; i < count; i++) {
            word += "java";
        }
        end = System.currentTimeMillis();
        System.out.println("String + : " + (end - start));

        word = "Hello, ";
        start = System.currentTimeMillis();
        for (int i = 0; i < count; i++) {
            word = word.concat("java");
        }
        end = System.currentTimeMillis();
        System.out.println("String.concat() : " + (end - start));

        word = "Hello, ";
        start = System.currentTimeMillis();
        for (int i = 0; i < count; i++) {
            builder.append("java");
        }
        word = builder.toString();
        end = System.currentTimeMillis();
        System.out.println("StringBuilder : " + (end - start));

        word = "Hello, ";
        start = System.currentTimeMillis();
        for (int i = 0; i < count; i++) {
            buffer.append("java");
        }
        word = buffer.toString();
        end = System.currentTimeMillis();
        System.out.println("StringBuffer : " + (end - start));
    }
}
```

运行结果如下所示：

|  | 1k | 1w | 10w | 100w |
| :------: | :------: | :------: | :------: |:------: |
| + | 11 | 397 | 20191 | 720286 |
| concat | 3 | 72 | 5671 | 763612 |
| StringBuilder | 0 | 0 | 3 | 17 |
| StringBuffer | 1 | 1 | 4 | 36 |

以上都是执行一次的结果，可能不太严谨，但还是能反映问题的。执行次数越多，性能差距越明显，`StringBuilder` > `StringBuffer` > `contact` > `+` 。关于其中原因，我想很多人应该都知道。下面从源码角度分析一下这几种字符串拼接方式。

## +

使用 `+` 拼接字符串是效率最低的一种方式吗？首先，我们要知道 `+` 具体是怎么拼接字符串的。对于这种我们不知道具体原理的时候，`javap` 是你的好选择。从最简单的一行代码开始：

```java
String str = "a" + "b";
```

这样写其实并不行，智能的编译器看到 `"a"+"b"` 就知道你要干啥了，所以你编译出来就是 `String str = "ab"`，我们稍作修改就可以了：

```java
String a = "a";
String str = a + "b";
```

`javap` 看一下字节码：

```
 0: ldc           #2    // String a
2: astore_1
3: new           #3     // class java/lang/StringBuilder
6: dup
7: invokespecial #4     // Method java/lang/StringBuilder."<init>":()V
10: aload_1
11: invokevirtual #5    // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
14: ldc           #6    // String b
16: invokevirtual #5    // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
19: invokevirtual #7    // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
22: astore_2
23: return
```

可以看到编译器自动将 `+` 转换成了 `StringBuilder.append()` 方法，拼接之后再调用 `StringBuilder.toString()` 方法转换成字符串。既然这样的话，那岂不是应该和 `StringBuilder` 的执行效率一样了？别忘了，上面的测试代码使用 `for` 循环模拟频繁的字符串拼接操作。使用 `+` 的话，在每一次循环中，都将重复下列操作：

* 新建 `StringBuilder` 对象
* 调用 `StringBuilder.append()` 方法
* 调用 `StringBuilder.toString()` 方法，该方法会通过 `new String()` 创建字符串

几万次循环下来，你看看创建了多少中间对象，怪不得这么慢，别人要么以空间换时间，要么以时间换空间。这家伙倒好，即浪费时间，又浪费空间。所以，在频繁拼接字符串的情况下，尽量避免使用 `+` 。那么，它存在的意义何在呢？有的时候我们就是要拼接两个字符串，使用 `+` ，直截了当。

## String.concat()

```java
public String concat(String str) {
    int otherLen = str.length();
    if (otherLen == 0) {
        return this; // str 为空直接返回 this
    }
    int len = value.length;
    char buf[] = Arrays.copyOf(value, len + otherLen);
    str.getChars(buf, len);
    return new String(buf, true);
}

void getChars(char dst[], int dstBegin) {
    System.arraycopy(value, 0, dst, dstBegin, value.length);
}
```

先构建新的字符数组 `buf[]`，再利用 `System.arraycopy()` 挪来挪去，最后 `new String()` 构建字符串。比 `+` 少了创建 `StringBuilder` 的过程，但每次循环中，又要重新创建字符数组，又要重新 `new` 字符串对象，频繁拼接的时候效率还是不是很理想。

再提一点，当传入 `str` 长度为 0 时，直接返回 `this`。这好像是 `String` 中唯一一个返回 `this` 的地方了。

## append()

`StringBuilder` 和 `StringBuffer` 其实是很像的，它两频繁拼接字符串的效率远胜于 `+` 和 `concat`。当循环执行 `10w` 次，分别耗时 `3ms` 、`4ms`, `StringBuilder` 还比 `StringBuffer` 快那么一点。至于为什么，`Read the fucking source code !`

先看看 `StringBuilder.append()` ：

```java
@Override
public StringBuilder append(String str) {
    super.append(str);
    return this;
}
```

并没有什么实际逻辑，直接调用了父类的 `append()` 方法。看一下 `StringBuilder` 的类声明：

```java
public final class StringBuilder
    extends AbstractStringBuilder
    implements java.io.Serializable, CharSequence{}
```

`StringBuilder` 继承了 `AbstractStringBuilder` 类，`StringBuferr` 其实也是。所以它们实际上调用的都是是 `AbstractStringBuilder.append()`：

```java
public AbstractStringBuilder append(String str) {
    if (str == null)
        return appendNull(); // 1
    int len = str.length();
    ensureCapacityInternal(count + len); // 2
    str.getChars(0, len, value, count); // 3
    count += len;
    return this;
}
```

代码中出现了两个变量，`value` 和 `count`，先来看看它们是干嘛的。

```java
    /**
     * The value is used for character storage.
     */
    char[] value;

    /**
     * The count is the number of characters used.
     */
    int count;
```

`value` 是一个字符数组，用来保存字符。它可以自动扩容，在后面的代码中你将会看到。`count` 是已使用的字符的数量，注意并不是 `vale[]` 的长度。再回到 `append()` 方法，分三部分来解析。

### appendNull(String)

当 `append()` 的参数为 `null` 时调用，它并不是什么都不添加，而是正如它的方法名那样，追加了 `null` 字符串。

```java

private AbstractStringBuilder appendNull() {
    int c = count;
    ensureCapacityInternal(c + 4);
    final char[] value = this.value;
    value[c++] = 'n';
    value[c++] = 'u';
    value[c++] = 'l';
    value[c++] = 'l';
    count = c;
    return this;
}
```

### ensureCapacityInternal(int)

```java
private void ensureCapacityInternal(int minimumCapacity) {
    // overflow-conscious code
    if (minimumCapacity - value.length > 0) {
        value = Arrays.copyOf(value,newCapacity(minimumCapacity));
    }
}
```

`ensureCapacityInternal()` 方法用来确保 `value[]` 的容量足以拼接参数中的字符串。如果容量不够，将调用 `Arrays.copyOf(value,newCapacity(minimumCapacity))` 对 `value[]` 进行扩容，`newCapacity(minimumCapacity)` 就是字符数组的新长度。

```java
private int newCapacity(int minCapacity) {
    // overflow-conscious code
    // 新容量等于旧容量乘以 2 再加上 2
    int newCapacity = (value.length << 1) + 2;
    if (newCapacity - minCapacity < 0) {
        newCapacity = minCapacity;
    }
    return (newCapacity <= 0 || MAX_ARRAY_SIZE - newCapacity < 0)
        ? hugeCapacity(minCapacity)
        : newCapacity;
}

private int hugeCapacity(int minCapacity) {
    if (Integer.MAX_VALUE - minCapacity < 0) { // overflow
        // 如果需求容量大于 Integer 最大值，直接抛出 OOM
        throw new OutOfMemoryError();
    }
    return (minCapacity > MAX_ARRAY_SIZE)
        ? minCapacity : MAX_ARRAY_SIZE;
}
```

基本的扩容逻辑是，新的数组大小是原来的两倍再加上 2，但是有个最大值 `MAX_ARRAY_SIZE`，其值是 `Integer.MAX_VALUE - 8`，减去 8 是因为一些虚拟机会在数组中保留一些头信息。当然，一般在程序中也达不到这个最大值。如果我们直接和虚拟机说，我需要一个大小为 `Integer.MAX_VALUE` 的新数组，那会直接抛出 `OOM`。

### getChars()

新数组创建好了，那么剩下的就是拼接字符串了。

```
str.getChars(0, len, value, count);
count += len;
```

`str` 是要拼接的字符串，是不是对这个 `getChars()` 方法很眼熟。仔细看过 `String` 源码的话，应该对这个方法还有印象。

```java
public void getChars(int srcBegin, int srcEnd, char dst[], int dstBegin) {
    if (srcBegin < 0) {
        throw new StringIndexOutOfBoundsException(srcBegin);
    }
    if (srcEnd > value.length) {
        throw new StringIndexOutOfBoundsException(srcEnd);
    }
    if (srcBegin > srcEnd) {
        throw new StringIndexOutOfBoundsException(srcEnd - srcBegin);
    }
    System.arraycopy(value, srcBegin, dst, dstBegin, srcEnd - srcBegin);
}
```

进行一些边界判断之后，利用 `System.arraycopy()` 拼接字符串。

看完这三部分，也就完成了一次字符串拼接。回想一下，在大量拼接字符串的过程中，`append()` 把时间都花在了哪里？数组扩容和 `System.arraycopy()` 操作，的确比 `+` 和 `concat()` 不停的 `new` 对象效率高多了。

还记得 `StringBuffer` 虽然也同样快，但是比 `StringBuilder` 慢了一些吧！来看看 `StringBuffer` 的实现：

```java
@Override
public synchronized StringBuffer append(String str) {
    toStringCache = null;
    super.append(str);
    return this;
}
```

逻辑是完全一致的，但是多了 `synchronized` 关键字，用来保证线程安全。所以会比 `StringBuilder` 耗时一些。关于 `StringBuilder` 和 `StringBuffer` 之间的区别，除了 `synchronized` 关键字就没有了。

## 总结

* `+` 和 `String.concat()` 只适合少量的字符串拼接操作，频繁拼接时性能不如人意
* `StringBuilder` 和 `StringBuffer` 在频繁拼接字符串时性能优异
* `StringBuilder` 不能保证线程安全。因此，在确定单线程执行的情况下，`StringBuilder` 是最优解
* `StringBuffer` 通过 `synchronized` 保证线程安全，适合多线程环境下使用。


> 文章首发于微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解，欢迎扫码关注！


![](https://user-gold-cdn.xitu.io/2019/3/30/169cf046d9579e78?w=258&h=258&f=jpeg&s=27711)
