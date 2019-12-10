上篇文章 [走进 JDK 之 Enum](https://juejin.im/post/5cb874e2f265da0387339a6c) 提到过，枚举很适合用来实现单例模式。实际上，在 Effective Java 中也提到过（果然英雄所见略同）：

> 单元素的枚举类型经常成为实现 Singleton 的最佳方法 。

首先什么是单例？就一条基本原则，**单例对象的类只会被初始化一次**。在 Java 中，我们可以说在 JVM 中只存在该类的唯一一个对象实例。在 Android 中，我们可以说在程序运行期间，该类有且仅有一个对象实例。说到单例模式的实现，你们肯定信手拈来，什么懒汉，饿汉，DCL，静态内部类，门清。在说单例之前，考虑下面几个问题：

* 你的单例线程安全吗?
* 你的单例反射安全吗？
* 你的单例序列化安全吗？

今天，我就来钻钻牛角尖，看看你们的单例是否真的 “单例”。

## 单例的一般实现

### 饿汉式

```java
public class HungrySingleton {

    private static final HungrySingleton mInstance = new HungrySingleton();

    private HungrySingleton() {
    }

    public static HungrySingleton getInstance() {
        return mInstance;
    }
}
```
私有构造器是单例的一般套路，保证不能在外部新建对象。饿汉式在类加载时期就已经初始化实例，由于类加载过程是线程安全的，所以饿汉式默认也是线程安全的。它的缺点也很明显，我真正需要单例对象的时机是我调用 `getInstance()` 的时候，而不是类加载时期。如果单例对象是很耗资源的，如数据库，socket 等等，无疑是不合适的。于是就有了懒汉式。

### 懒汉式

```java

public class LazySingleton {

    private static LazySingleton mInstance;

    private LazySingleton() {
    }

    public static synchronized LazySingleton getInstance() {
        if (mInstance == null)
            mInstance = new LazySingleton();
        return mInstance;
    }
}
```

实例化的时机挪到了 `getInstance()` 方法中，做到了 lazy init ，但也失去了类加载时期初始化的线程安全保障。因此使用了 `synchronized` 关键字来保障线程安全。但这显然是一个无差别攻击，管你要不要同步，管你是不是多线程，一律给我加锁。这也带来了额外的性能消耗。这点问题肯定难不倒程序员们，于是，双重检查锁定(DCL, Double Check Lock) 应运而生。

### DCL

```java
public class DCLSingleton {

    private static DCLSingleton mInstance;

    private DCLSingleton() {
    }

    public static DCLSingleton getInstance() {
        if (mInstance == null) {                    // 1
            synchronized (DCLSingleton.class) {     // 2
                if (mInstance == null)              // 3
                    mInstance = new DCLSingleton(); // 4
            }
        }
        return mInstance;
    }
}
```

`1` 处做第一次判断，如果已经实例化了，直接返回对象，避免无用的同步消耗。`2` 处仅对实例化过程做同步操作，保证单例。`3` 处做第二次判断，只有 `mInstance` 为空时再初始化。看起来时多么的完美，保证线程安全的同时又兼顾性能。但是 DCL 存在一个致命缺陷，就是重排序导致的多线程访问可能获得一个未初始化的对象。

首先记住上面标记的 4 行代码。其中第 4 行代码 `mInstance = new DCLSingleton();` 在 JVM 看来有这么几步：

> 1. 为对象分配内存空间
> 2. 初始化对象
> 3. 将 mInstance 引用指向第 1 步中分配的内存地址

在单线程内，在不影响执行结果的前提下，可能存在指令重排序。例如下列代码：

```java
int a = 1;
int b = 2;
```

在 JVM 中你是无法确保这两行代码谁先执行的，因为谁先执行都不影响程序运行结果。同理，创建实例对象的三部中，第 2 步 **初始化对象** 和 第 3 步 **将 mInstance 引用指向对象的内存地址** 之间也是可能存在重排序的。

> 1. 为对象分配内存空间
> 2. 将 mInstance 引用指向第 1 步中分配的内存地址
> 3. 初始化对象

这样的话，就存在这样一种可能。线程 A 按上面重排序之后的指令执行，当执行到第 2 行 **将 mInstance 引用指向对象的内存地址** 时，线程 B 开始执行了，此时线程 A 已为 `mInstance` 赋值，线程 B 进行 DCL 的第一次判断 `if (mInstance == null)` ,结果为 `false`，直接返回 `mInstance` 指向的对象，但是由于重排序的缘故，对象其实尚未初始化，这样就出问题了。还挺绕口的，借用 《Java 并发编程艺术》 中的一张表格，会对执行流程更加清晰。

| 时间  | 线程 A | 线程 B |
| :---: |---|---|
|  t1 |  A1: 分配对象的内存空间  |     |
|  t2 |  A3: 设置 mInstance 指向内存空间  |     |
|  t3 |    |  B1: 判断 mInstance 是否为空   |
|  t4 |    |  B2: 由于 mInstance 不为空，线程 B 将访问 mInstance 指向的对象   |
|  t5 |  A2: 初始化对象  |     |
|  t6 |  A3: 访问 mInstance 引用的对象  |     |

`A3` 和 `A2` 发生重排序导致线程 B 获取了一个尚未初始化的对象。

说了半天，该怎么改？其实很简单，禁止多线程下的重排序就可以了，只需要用 `volatile` 关键字修饰 `mInstance` 。在 JDK 1.5 中，增强了 volatile 的内存语义，对一个volatile 域的写，happens-before 于任意后续对这个 volatile 域的读。volatile 会禁止一些处理器重排序，此时 DCL 就做到了真正的线程安全。

### 静态内部类模式

```java
public class StaticInnerSingleton {

    private StaticInnerSingleton(){}

    private static class SingletonHolder{
        private static final StaticInnerSingleton mInstance=new StaticInnerSingleton();
    }

    public static StaticInnerSingleton getInstance(){
        return SingletonHolder.mInstance;
    }
}
```

鉴于 DCL 繁琐的代码，程序员又发明了静态内部类模式，它和饿汉式一样基于类加载时器的线程安全，但是又做到了延迟加载。`SingletonHolder` 是一个静态内部类，当外部类被加载的时候并不会初始化。当调用 `getInstance()` 方法时，才会被加载。

枚举单例暂且不提，放在最后再说。先对上面的单例模式做个检测。

## 真的是单例？

还记得开头的提问吗？

* 你的单例线程安全吗?
* 你的单例反射安全吗？
* 你的单例序列化安全吗？

上面大篇幅的论述都在说明线程安全。下面看看反射安全和序列化安全。

### 反射安全

直接上代码，我用 DCL 来做测试：

```java
public static void main(String[] args) {

    DCLSingleton singleton1 = DCLSingleton.getInstance();
    DCLSingleton singleton2 = null;

    try {
        Class<DCLSingleton> clazz = DCLSingleton.class;
        Constructor<DCLSingleton> constructor = clazz.getDeclaredConstructor();
        constructor.setAccessible(true);
        singleton2 = constructor.newInstance();
    } catch (Exception e) {
        e.printStackTrace();
    }

    System.out.println(singleton1.hashCode());
    System.out.println(singleton2.hashCode());

}
```

执行结果：

```
1627674070
1360875712
```

很无情，通过反射破坏了单例。如何保证反射安全呢？只能以暴制暴，当已经存在实例的时候再去调用构造函数直接抛出异常，对构造函数做如下修改：

```java
private DCLSingleton() {
    if (mInstance!=null)
        throw new RuntimeException("想反射我，没门！");
}
```

上面的测试代码会直接抛出异常。

### 序列化安全

将你的单例类实现 `Serializable` 持久化保存起来，日后再恢复出来，他还是单例吗？

```java
public static void main(String[] args) {

    DCLSingleton singleton1 = DCLSingleton.getInstance();
    DCLSingleton singleton2 = null;

    try {
        ObjectOutput output=new ObjectOutputStream(new FileOutputStream("singleton.ser"));
        output.writeObject(singleton1);
        output.close();

        ObjectInput input=new ObjectInputStream(new FileInputStream("singleton.ser"));
        singleton2= (DCLSingleton) input.readObject();
    } catch (Exception e) {
        e.printStackTrace();
    }
    System.out.println(singleton1.hashCode());
    System.out.println(singleton2.hashCode());

}
```

执行结果：

```
644117698
793589513
```

不堪一击。反序列化时生成了新的实例对象。要修复也很简单，只需要修改反序列化的逻辑就可以了，即重写 `readResolve()` 方法，使其返回统一实例。

```java
protected Object readResolve() {
    return getInstance();
}
```

脆弱不堪的单例模式经过重重考验，进化成了完全体，延迟加载，线程安全，反射安全，序列化安全。全部代码如下：

```java
public class DCLSingleton implements Serializable {

    private static DCLSingleton mInstance;

    private DCLSingleton() {
        if (mInstance!=null)
            throw new RuntimeException("想反射我，没门！");
    }

    public static DCLSingleton getInstance() {
        if (mInstance == null) {
            synchronized (DCLSingleton.class) {
                if (mInstance == null)
                    mInstance = new DCLSingleton();
            }
        }
        return mInstance;
    }

    protected Object readResolve() {
        return getInstance();
    }
}
```

## 枚举单例

枚举看到 DCL 就开始嘲笑他了，“你瞅瞅你那是啥，写个单例费那大劲呢？” 于是撸起袖子自己写了一个枚举单例：

```java
public enum EnumSingleton {
    INSTANCE;
}
```

DCL 反问，“你这啥玩意，你这就是单例了？我来扒了你的皮看看 ！” 于是 DCL 掏出 jad ，扒了 Enum 的衣服，拉出来示众：

```java
public final class EnumSingleton extends Enum {

    public static EnumSingleton[] values() {
        return (EnumSingleton[])$VALUES.clone();
    }

    public static EnumSingleton valueOf(String s) {
        return (EnumSingleton)Enum.valueOf(test/singleton/EnumSingleton, s);
    }

    private EnumSingleton(String s, int i) {
        super(s, i);
    }

    public static final EnumSingleton INSTANCE;
    private static final EnumSingleton $VALUES[];

    static {
        INSTANCE = new EnumSingleton("INSTANCE", 0);
        $VALUES = (new EnumSingleton[] {
            INSTANCE
        });
    }
}
```

我们依次来检查枚举单例的线程安全，反射安全，序列化安全。

首先枚举单例无疑是线程安全的，类似饿汉式，`INSTANCE` 的初始化放在了 static 静态代码段中，在类加载阶段执行。由此可见，枚举单例并不是延时加载的。

对于反射安全，又要掏出上面的检测代码了，根据 `EnumSingleton` 的构造器，需要稍微做些改动：

```java
public static void main(String[] args) {

    EnumSingleton singleton1 = EnumSingleton.INSTANCE;
    EnumSingleton singleton2 = null;

    try {
        Class<EnumSingleton> clazz = EnumSingleton.class;
        Constructor<EnumSingleton> constructor = clazz.getDeclaredConstructor(String.class,int.class);
        constructor.setAccessible(true);
        singleton2 = constructor.newInstance("test",1);
    } catch (Exception e) {
        e.printStackTrace();
    }

    System.out.println(singleton1.hashCode());
    System.out.println(singleton2.hashCode());

}
```

结果直接报错，错误日志如下：

```
java.lang.IllegalArgumentException: Cannot reflectively create enum objects
	at java.lang.reflect.Constructor.newInstance(Constructor.java:417)
	at singleton.SingleTest.main(SingleTest.java:16)
```

错误发生在 `Constructor.newInstance()` 方法，又要从源码中找答案了，在 `newInstance()` 源码中，有这么一句：

```java
if ((clazz.getModifiers() & Modifier.ENUM) != 0)
            throw new IllegalArgumentException("Cannot reflectively create enum objects");
```

如果是枚举修饰的，直接抛出异常。和之前的对抗反射的手段一致，压根就不给你反射。所以，枚举单例也是天生反射安全的。

最后枚举单例也是序列化安全的，上篇文章中已经说明过，你可以运行测试代码试试。

看起来枚举单例的确是个不错的选择，代码简单，又能保证绝大多数情况下的单例实例唯一。但是真正在开发中大家好像用的并不多，更多的可能应该是枚举在 Java 1.5 中才添加，大家默认已经习惯了其他的单例实现方式。

## 代码最少的单例？

说到枚举单例代码简单，Kotlin 第一个站出来不服了。我敢说第一，谁敢说第二，给你们献丑了：

```java
object KotlinSingleton { }
```
jad 反编译一下：

```java
public final class KotlinSingleton {

    private KotlinSingleton(){
    }

    public static final KotlinSingleton INSTANCE;

    static {
        KotlinSingleton kotlinsingleton = new KotlinSingleton();
        INSTANCE = kotlinsingleton;
    }
}
```

可以看到，Kotlin 的单例其实也是饿汉式的一种，不钻牛角尖的话，基本可以满足大部分需求。

吹毛求疵的谈了谈单例模式，可以看见要完全的保证单例还是有很多坑点的。在开发中并没有必要钻牛角尖，例如 Kotlin 默认提供的单例实现就是饿汉式而已，其实已经可以满足绝大多数的情况了。

由枚举引申出了这么一篇文章，大家姑且可以当做娱乐看一看，交个朋友。


![](https://user-gold-cdn.xitu.io/2019/4/20/16a36652611a46e2?w=2800&h=800&f=jpeg&s=178470)

> 秉心说，专注 Java/Android 原创知识分享，LeetCode 题解，每周一篇阅读分享，欢迎扫码关注！
> 后台回复 “群聊” ，加入秉心说读者群，一个有趣的、有问必答的交流群，最重要的，还有不定期红包哦！  
