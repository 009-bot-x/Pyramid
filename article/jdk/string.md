> 文中相关源码： [String.java](https://github.com/lulululbj/jdk8u/blob/master/jdk/src/share/classes/java/lang/String.java)

今天来说说 `String`。

贯穿全文，你需要始终记住这句话，**`String 是不可变类`** 。其实前面说过的所有基本数据类型包装类都是不可变类，但是在 `String` 的源码中，`不可变类` 的概念体现的更加淋漓尽致。所以，在阅读 `String` 源码的同时，抽丝剥茧，你会对不可变类有更深的理解。

## 什么是不可变类 ？

首先来看一下什么是不可变类？`Effective Java 第三版` 第 17 条 `使不可变性最小化` 中对 `不可变类` 的解释：

> 不可变类是指其实例不能被修改的类。每个实例中包含的所有信息都必须在创建该实例的时候就提供，并在对象的整个生命周期 (lifetime) 内固定不变 。
>  
> 为了使类成为不可变，要遵循下面五条规则：
> 1. **不要提供任何会修改对象状态的方法**（也称为设值方法） 。
> 2. **保证类不会被扩展。** 为了防止子类化，一般做法是声明这个类成为 final 的。
> 3. **声明所有的域都是 final 的。**
> 4. **声明所有的域都为私有的。** 这样可以防止客户端获得访问被域引用的可变对象的权限，并防止客户端直接修改这些对象 。  
> 5. **确保对于任何可变组件的互斥访问。** 如果类具有指向可变对象的域，则必须确保该类的客户端无法获得指向这些对象的引用 。 并且，永远不要用客户端提供的对象引用来初始化这样的域，也不要从任何访问方法（ accessor）中返回该对象引用 。 在构造器、访问方法和 readObject 方法（详见第 88 条）中请使用保护性拷贝（ defensive copy ）技术（详见第50 条） 。

根据这五条原则，来品尝一下 `String.java` 吧！

## 类定义

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {}
```

对应原则第二点 `保证类不会被扩展`，使用 `final` 修饰。此外：
* 实现了 `Serializable` 接口，具备序列化能力
* 实现了 `Comparable` 接口，具备比较对象大小能力，根据单字符的大小比较。
* 实现了 `CharSequence` 接口，表示是一个字符序列，实现了该接口下的一些方法。

## 字段

```java
private final char value[]; // 储存字符串
private int hash; // 哈希值，默认为 0
private static final long serialVersionUID = -6849794470754667710L; // 序列化标识
```

看起来 `String` 是一个独立的对象，其实它是使用基本数据类型的数组 `char[]` 实现的。作为使用者，我们不需要打开 `String` 的黑匣子，直接根据它的 API 使用就可以了，这正是 Java 的封装性的体现。但是作为开发者，我们就有必要一探究竟了。

`private final char value[]` , 对应原则中第三条和第四条，`声明所有的域都是 final 的` ，`声明所有的域都为私有的`。看到这里，你大概明白了一点为什么 `String` 不可变。因为真正用来存储字符串的字符数组是 `final` 修饰的，是不可变的。

## 构造函数

`String` 的构造函数很多，大致可以分为以下四种：

### 无参构造

```java
public String() {
    this.value = "".value;
}
```

无参构造默认构建一个空字符串。鉴于 `String` 是不可变类，所以此构造器并没有什么意义，一般你也不会去构建一个不可变的空字符串对象。

### 参数是 `byte[]`

```java
public String(byte bytes[]) {}
public String(byte bytes[], int offset, int length) {}
public String(byte bytes[], Charset charset) {}
public String(byte bytes[], String charsetName) {}
public String(byte bytes[], int offset, int length, Charset charset) {}
public String(byte bytes[], int offset, int length, String charsetName) {}
```

已经废弃的就不再列举了。上面这些构造函数都差不多，最后都是调用 `StringCoding.decode()` 方法将字节数组转换为字符数组，再赋值给 `value[]`。这里要注意一点，参数未指定编码格式的话，默认使用系统的编码格式，如果没有获取到系统编码格式，则使用 `ISO-8859-1` 格式。

### 参数是 `char[]`

参数是 `char[]` 的构造函数有 3 个，逐个看一下：

```java
public String(char value[]) {
    this.value = Arrays.copyOf(value, value.length);
}
```

为了保证不可变性，并没有直接赋值，`this.value = value`。而是使用 `Arrays.copy()` 方法将参数中的字符数组内容拷贝到 `value[]` 中。防止参数中字符数组的改变破坏了不可变性。

第二个：

```java
public String(char value[], int offset, int count) {
    if (offset < 0) {
        throw new StringIndexOutOfBoundsException(offset);
    }
    if (count <= 0) {
        if (count < 0) {
            throw new StringIndexOutOfBoundsException(count);
        }
        if (offset <= value.length) {
            this.value = "".value;
            return;
        }
    }
    // Note: offset or count might be near -1>>>1.
    if (offset > value.length - count) {
        throw new StringIndexOutOfBoundsException(offset + count);
    }
    this.value = Arrays.copyOfRange(value, offset, offset+count);
}
```

和上面的构造函数一样，只是截取了参数中字符数组的一部分来构建字符串。

第三个：

```java
/*
    * Package private constructor which shares value array for speed.
    * this constructor is always expected to be called with share==true.
    * a separate constructor is needed because we already have a public
    * String(char[]) constructor that makes a copy of the given char[].
    *
    * 仅当前包可使用。
    * 直接将 this.value 指向参数中的 char[]，不再进行 copy 操作
    * 性能好，节省内存，外包不可使用，也不会破坏不可变性
    */
    String(char[] value, boolean share) {
        // assert share : "unshared not supported";
        this.value = value;
    }
```

这里的 `share` 一般只能为 `true`，虽然并没有使用到。增加这个参数是为了和第一个构造函数区分开来，表示 `value[]` 共享了参数中的字符数组，因为这里是直接赋值的，并没有使用 `Arrays.copy()`。那这不是破坏了 `String` 的不可变性吗？其实并没有，因为你根本没法调用这个构造函数，它的包私有的。但是在 JDK 内部你可以发现它的身影，

![](https://user-gold-cdn.xitu.io/2019/4/2/169dce8705f1bd65?w=1156&h=762&f=png&s=99404)

没有了 `copy` 操作，大幅提高了效率。但是为了保证不可变性，外部是不能调用的。

### 其他构造函数

```java
// 基于代码点
public String(int[] codePoints, int offset, int count) {}

// 基于 StringBuffer，需要同步
public String(StringBuffer buffer) {
    synchronized(buffer) {
        this.value = Arrays.copyOf(buffer.getValue(), buffer.length());
    }
}

// 基于 StringBuilder，不需要同步
public String(StringBuilder builder) {
    this.value = Arrays.copyOf(builder.getValue(), builder.length());
}
```

## 方法

回头再看一下 `String` 的不可变性，`value[]` 是 `private final` 修饰的，这样就真的可以保证不可变吗？

```java
final char[] value = {'a','b','c'};
value[1] = 'd';
```

这是不是就轻而易举的打破了不可变性？`final value[]` 只能保证其引用不能再指向其他内存地址，但是其真正的值还是可以改变的。所以仅仅通过一个 `final` 是无法保证其值不变的，如果类本身提供方法修改实例值，那就没有办法保证不变性了。对应原则中第一条，`不要提供任何会修改对象状态的方法`，`String` 百分之百做到了这一点，它没有对外提供任何可以修改 `value` 的方法。

在 `String` 中有许多对字符串进行操作的函数，例如 `substring` `concat` `replace` `replaceAll` 等等，这些函数是否会修改类中的 value 域呢？下面就来看一看源码。

### substring(int beginIndex)

```java
public String substring(int beginIndex) {
    if (beginIndex < 0) {
        throw new StringIndexOutOfBoundsException(beginIndex);
    }
    int subLen = value.length - beginIndex;
    if (subLen < 0) {
        throw new StringIndexOutOfBoundsException(subLen);
    }
    // beginIndex 不为 0， 返回一个 新的 String 对象
    return (beginIndex == 0) ? this : new String(value, beginIndex, subLen);
}
```

### concat(String str)

```java
public String concat(String str) {
    int otherLen = str.length();
    if (otherLen == 0) {
        return this;
    }
    int len = value.length;
    char buf[] = Arrays.copyOf(value, len + otherLen);
    str.getChars(buf, len);
    return new String(buf, true); // 返回新的 String 对象
}
```

### replace(char oldChar, char newChar)

```java
public String replace(char oldChar, char newChar) {
    if (oldChar != newChar) {
        int len = value.length;
        int i = -1;
        char[] val = value; /* avoid getfield opcode */

           while (++i < len) {
               if (val[i] == oldChar) {
                   break;
               }
           }
           if (i < len) {
               char buf[] = new char[len];
               for (int j = 0; j < i; j++) {
                   buf[j] = val[j];
               }
               while (i < len) {
                   char c = val[i];
                   buf[i] = (c == oldChar) ? newChar : c;
                   i++;
               }
               return new String(buf, true); // 返回新的 String 对象
           }
    }
    return this;
}
```

`String` 类的方法实现都相对简单，但是无一例外，它们绝对不会去修改 `value[]` 的值，需要返回 `String` 对象的话，都会重新 `new` 一个。正像原则第五条中所说的，确保对于任何可变组件的互斥访问。 如果类具有指向可变对象的域，则必须确保该类的客户端无法获得指向这些对象的引用。

## String.intern()

```java
public native String intern();
```

这个方法比较特殊，是个本地方法。如果该字符串在常量池中已经存在，直接返回其引用。如果不存在，存入常量池再返回其引用。在下一篇文章中会进行详细介绍。

其他方法的源码就不列举了，感兴趣的可以到我上传的 jdk 源码 看看，[String.java](https://github.com/lulululbj/jdk8u/blob/master/jdk/src/share/classes/java/lang/String.java)，添加了部分注释。

## 不可变类的好处

从头到尾都在说不可变类，那么它有哪些好处呢？

* 不可变对象比较简单。
* 不可变对象本质上是线程安全的，它们不要求同步。不可变对象可以被自由地共享。
* 不仅可以共享不可变对象，甚至也可以共享它们的内部信息。
* 不可变对象为其他对象提供了大量的构件。无论是可变的还是不可变的对象。
* 不可变对象无偿地提供了失败的原子性。

不可变类真正唯一的缺点是，对于每个不同的值都需要一个单独的对象。所以当需要大量字符串对象的时候，`String` 就成了性能瓶颈，这也催生了 `StringBuffer` 和 `StringBuilder`。后面会单独分析。

## String 真的不可变吗 ？

学习就是自己不断打自己脸的过程。真的没有办法修改 String 对象的值吗？答案肯定是否定的，反射机制可以做到很多平常做不到的事情。

```java
String str = "123";
System.out.println(str);
Field field = String.class.getDeclaredField("value");
field.setAccessible(true);
char[] value = (char[]) field.get(str);
value[1] = '3';
System.out.println(str);
```

执行结果：

```
123
133
```

通过反射，的确修改了 `value[]` 的值。

## 总结

借着 `String` 源码，说了说 `不可变类`。简单总结一下 `String` 做了哪些措施来保证不可变性：

* `value[]` 使用 `private final` 修饰
* 构造函数中复制实参的值给 `value[]`
* 不对外提供任何修改 `value[]` 值的方法
* 需要返回 `String` 的方法，绝不返回原对象，都是重新 `new` 一个 `String` 返回

下一篇还是写 `String` , 说说 `String` 在内存中的位置和字符串常量池的一些知识，以及 `String` 相关的常见面试题。

> 文章首发于微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解，欢迎关注！


![](https://user-gold-cdn.xitu.io/2019/3/30/169cf046d9579e78?w=258&h=258&f=jpeg&s=27711)
