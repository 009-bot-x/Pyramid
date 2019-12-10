## 什么是枚举

什么是枚举？说实话，在我这些年的开发生涯中，用过枚举的次数大概两只手都可以数的过来。当然你不能说枚举一无是处，只能说是我对 Java 理解的还不够深刻，在可以使用枚举的时候并没有去使用。

假设我有两个孩子（其实不用假设），每到周末他们都不知道去上什么辅导班。于是我就写了一个简单的程序来告诉他们，伪代码如下：

```java
public final static int DAVID = 0;
public final static int MARRY = 1;

public void order(int who){
    switch (who){
        case DAVID:
            System.out.println("大卫，去练武术！");
            break;
        case MARRY:
            System.out.println("玛丽，去练跳舞!");
            break;
         default:
            System.out.println("今天休息");
    }
}
```

然后我告诉 David 你输入 0，告诉 Marry 你输入 1，就可以了。不久之后，辅导班老师就指点问候我了，您家的两个孩子呢？这个气的我呀，立马回家看了看日志，两个孩子除了 0 和 1，其他数字都输齐了。

由此可见，这样直接使用 int 常量无法限定用户的输入，你让它输 0 或 1，它偏偏输个 45678。从代码可读性来说，参数是个 int 值，并不是那么直观的就可以看出来应该输入什么。无奈之下，我只得掏出 《Java 编程思想》，来治治这两个熊孩子。下面是我优化的程序：

```java

public enum Child { DAVID, MARY }

public  void order(Child who) {
    switch (who) {
        case DAVID:
            System.out.println("大卫，去练武术！");
            break;
        case MARY:
            System.out.println("玛丽，去练跳舞!");
            break;
        default:
            System.out.println("今天休息");
    }
}
```

实际上已经可以删除 `default` 了，因为参数是枚举类 `Child`，输入范围已经被限定为我定义的枚举，输入其他值将无法通过编译。两个熊孩子终于可以愉快的去上课了。

枚举是 Java 1.5 中新增的引用类型，指由一组固定的常量组成合法值的类型，其实上面的例子并不那么适合枚举，例如一年四季，一周七天这样的，更加适合枚举。相比使用 int 常量来定义，枚举具有类型安全和可读性良好的优势。《Effective Java》中也鼓励 `用 enum 代替 int 常量`。

除此之外，还可以给枚举添加字段和方法，例如，我想给每个孩子加上姓名，可以这么做：

```java
public enum Child {

    DAVID("David"), MARY("Marry");

    private String name;

    Child(String name){
        this.name=name;
    }

    public static void main(String[] args){
        for (Child child:Child.values()){
            System.out.println(child.ordinal()+" "+child.name);
        }
    }
}
```

注意，枚举的定义只能放在第一行，如果你把 ` DAVID("David"), MARY("Marry");` 放在其他位置，是无法通过编译的。另外别忘了，最后一个枚举常量后面要加分号。

`values()` 会枚举出 Child 中定义的所有枚举常量。打印结果如下：

```
0 David
1 Marry
```

是不是比之前的 int 常量那种方式强大多了。

## 源码解析

### Enum

走进 JDK 系列，那必然是少不了源码解析的。

#### 类定义

```java
public abstract class Enum<E extends Enum<E>>
        implements Comparable<E>, Serializable {}
```

`Enum` 是一个抽象类，我们自己定义的枚举类都继承了 `Enum`。实现了 `Comparable` 接口，重写了 `compare` 的逻辑。实现了 `Serializable` 接口，可以序列化。但是枚举对序列化作了一定的限制，在序列化的时候仅仅是将枚举对象的 name 属性输出到结果中，反序列化的时候则是通过 `Enum.valueOf()` 方法来查找枚举对象。同时，编译器是不允许任何对这种序列化机制的定制的，因此禁用了 `writeObject` 、`readObject` 、`readObjectNoData` 、`writeReplace` 和 `readResolve` 等方法。因此，用枚举来实现单例模式的话，是反序列化安全的，因为即使反序列化也不会生成新的对象。

#### 字段

```java
private final String name; // 枚举实例的名称，就是枚举声明中的名称
private final int ordinal; // 在枚举声明中的次序，从 0 开始
```

枚举类就只有这两个字段。`name` 就是枚举声明的名字，比如 `DAVID` 的 `name` 就是 `DAVID`。`ordinal` 就是声明中的次序，之所以在 `switch` 中可以使用枚举，就是因为编译器会自动调用枚举的 `ordinal()` 方法。

#### 构造函数

```java
protected Enum(String name, int ordinal) {
    this.name = name;
    this.ordinal = ordinal;
}
```

受保护的，你不能调用枚举类的构造函数。

#### 方法

```java
public final String name() {
    return name; // 返回 name
}

public final int ordinal() {
    return ordinal; // 返回 ordinal
}

public String toString() {
    return name; // 可以重写使得枚举类返回一个用户友好的名字，默认返回 name
}

public final boolean equals(Object other) {
    return this==other; // 枚举的 equals 和 == 是等价的
}

protected final Object clone() throws CloneNotSupportedException {
    throw new CloneNotSupportedException(); // 枚举不支持 clone，单例
}

public final int compareTo(E o) {
    Enum<?> other = (Enum<?>)o;
    Enum<E> self = this;
    if (self.getClass() != other.getClass() && // optimization
        self.getDeclaringClass() != other.getDeclaringClass())
        throw new ClassCastException();
    return self.ordinal - other.ordinal; // 实际上是比较 ordinal
}

// 根据指定的枚举类型和名称返回枚举实例，在反序列化中会使用
public static <T extends Enum<T>> T valueOf(Class<T> enumType,String name) {
    T result = enumType.enumConstantDirectory().get(name);
    if (result != null)
        return result;
    if (name == null)
        throw new NullPointerException("Name is null");
    throw new IllegalArgumentException(
        "No enum constant " + enumType.getCanonicalName() + "." + name);
}
```

还记得之前使用过的 `values()` 方法吗，用来遍历枚举的。找遍了 `Enum.java` 也没有看到这个方法，既然父类中没有这个方法，那么一定是在子类中声明的了。下面我们来验证一下。

### Enum 子类

我们是怎么定义枚举的，

```java
public enum Child {
    DAVID("David"), MARY("Marry");
    private String name;
    Child(String name){
        this.name=name;
    }
}
```

并没有显示的去继承 `Enum`，而是使用了 `enum` 关键字，虽然没有使用 `class` 关键字，但其实它还是一个类，只是编译器帮我们做了中间步骤。之前有说过，知其然不知其所以然的时候，`javap` 是你的好帮手。这次我们不 javap 了，毕竟字节码可读性不是那么的好。我们使用 `jad` 反编译 `Child.class` 文件，得到结果如下:

```java
// Decompiled by Jad v1.5.8e. Copyright 2001 Pavel Kouznetsov.
// Jad home page: http://www.geocities.com/kpdus/jad.html
// Decompiler options: packimports(3)
// Source File Name:   Child.java

package enums;

public final class Child extends Enum {

    public static Child[] values() {
        return (Child[])$VALUES.clone();
    }

    public static Child valueOf(String s) {
        return (Child)Enum.valueOf(enums/Child, s);
    }

    private Child(String s, int i, String s1) {
        super(s, i);
        name = s1;
    }

    public static final Child DAVID;
    public static final Child MARY;
    private String name;
    private static final Child $VALUES[];

    static {
        DAVID = new Child("DAVID", 0, "David");
        MARY = new Child("MARY", 1, "Marry");
        $VALUES = (new Child[] {
            DAVID, MARY
        });
    }
}
```

看到这个东西，你就应该全明白了。说起来叫枚举，其实就是普通的类，只是我们只需要使用 `enum` 关键字，编译器就会帮我们做好一切。枚举中声明的变量都是 `static final` 的，且在 static 代码块中进行初始化，并存入对象数组 `$VALUES`。所以枚举实例的创建默认是线程安全的。除此之外，编译器还自动生成了 `values()` 方法，返回 `$VALUES` 数组的克隆。

说到这里，你应该对枚举的原理很清楚了。枚举的种种特性都特别契合单例模式，天生的线程安全和反序列化安全，这都是其他单例模式所不具备的。但是在我所见过的代码中，真正使用枚举去做单例的好像少之又少。具体的原因有待考究。

## 真的要使用枚举吗？

![](https://user-gold-cdn.xitu.io/2019/4/18/16a3115a3bd7b197?w=720&h=215&f=jpeg&s=47782)

站在 Android 开发者的角度，实际上官方是不建议我们使用枚举的。

> 枚举占用的空间通常是静态常量的两倍。你应该严格避免在 Android 中使用枚举。

其实我并不是完全赞同。`MVP` 多了那么多接口和类，我们应该使用吗？在如今的手机内存下，如果你的应用发生了 `OOM`，我想枚举应该不是罪魁祸首吧。只要不过度使用，站在增强代码可读性，保证类型安全的角度，用枚举代替静态常量肯定是个好选择。当然如果你说你要追求极致的性能，那倒不必使用枚举了。

> 文章首发微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解，欢迎关注！

![](https://user-gold-cdn.xitu.io/2019/3/30/169cf046d9579e78?w=258&h=258&f=jpeg&s=27711)
