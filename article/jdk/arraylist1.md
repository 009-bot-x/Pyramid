这篇本来是准备写 `Java 集合框架概述` 的，就是写起来效果不怎么样，可能是对整个 Java 集合框架还没有做到了然于心。所以还是先来源码分析，写完所有集合类的分析之后，再来总体概述。今天就从最最常用的 `ArrayList` 说起。

## 概述

`ArrayList` 是一种可以动态增长和缩减的线性表数据结构，允许重复元素，允许 null 值。基于动态数组实现，在内存中是连续的，这点和链表不同。另外，它不是线程安全的，与之相对应的同样基于动态数组实现的有序序列 `Vector` 则是线程安全的。

由于数组在内存中占用连续的内存空间，所以 `ArrayList` 具备随机访问能力，其根据下标随机访问的时间复杂度是 `O(1)`。同样，为了保证内存的连续性，其 **插入** 和 **删除** 操作就相对低效的多。在指定位置插入数据，就要将该位置之后的数据都往后挪，才能腾出空间。在指定位置删除数据，就要将该位置之后的数据全部往前挪，才能保证空间连续性。它们的平均时间复杂度都是 `O(n)`。

`ArrayList` 的使用还是比较简单的，下面还是带着两个问题看源码:

> ArrayList 初始大小是多少？它是如何动态扩容的？

## 源码解析

### 类声明

![](https://user-gold-cdn.xitu.io/2019/4/24/16a4fd7c770fd00a?w=869&h=385&f=png&s=16384)

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable {}
```

* `Collection` 是所有集合的根接口，定义了一些通用性的行为，抽象类 `AbstractCollection` 提供了部分集合类型无关的通用实现。`List` 接口针对有序集合扩展了 `Collection` 接口，抽象类 `AbstractList` 提供了部分默认实现，当然 `ArrayList` 并没有照单全收，更多的是重写提供了自己的实现。
* 实现了 `RandomAccess` 接口说明其支持快速随机访问，其实并没有实现任何方法，应该仅仅只是起一个标记的作用。
* 实现 `Cloneable` 接口，提供浅拷贝。
* 实现了 `Serializable` 接口，提供序列化能力，且重写了 `readObject()` 和 `writeObject()` 方法。

### 成员变量

```java
private static final int DEFAULT_CAPACITY = 10; // 默认初始容量
private static final Object[] EMPTY_ELEMENTDATA = {}; // 共享空数组
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {}; // 默认共享空数组
transient Object[] elementData; // 真正保存数据的数组
private int size; // 当前元素个数
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8; // 数组容量最大值
```

`elementData` 是真正用来保存数据的数组。关于它的默认大小，让人很容易搞错。一看到 `DEFAULT_CAPACITY` 为 10，让人情不自禁的认为我一旦新建了一个 `ArrayList`，它的默认大小就是 10。其实并不是这样的，后面看到构造函数的时候你就理解了。

数组的最大容量是 `Integer.MAX_VALUE - 8`，看到这个数字你应该很熟悉。`AbstractStringBuilder` 类用来存储字符的 `char[]` ，最大容量也是这个数字。考虑到一些虚拟机实现会保留数组对象的头信息，大于此值可能会导致 OOM ，注意只是可能。但是如果大于 `Integer.MAX_VALUE` 的话，就会直接抛出 OOM 。

### 构造函数

```java
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

`DEFAULTCAPACITY_EMPTY_ELEMENTDATA` 是一个空数组，所以当你执行 `List list = new ArrayList()` 时，实际上创建了一个空数组，并不是容量为 10 的数组。

```java
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+ initialCapacity);
    }
}
```

当我们可以预估到 ArrayList 需要容纳的元素数量时，我们可以直接指定数组大小 `initialCapacity`，避免后续自动扩容带来的性能损耗和空间浪费。`initialCapacity` 大小按如下规则：

* 大于 0 时，创建指定大小的数组
* 等于 0 时，使用成员变量 `EMPTY_ELEMENTDATA`，它是一个空数组
* 小于 0 时，直接抛出异常

```java
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // replace with empty array.
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
```

我们也可以用一个集合来初始化 ArrayList 。调用集合的 `toArray()` 方法转换为数组并赋给 `elementData`。如果传入的集合长度为 0，则将空数组 `EMPTY_ELEMENTDATA` 赋给 `elementData`。

### 方法

ArrayList 提供了插入，删除，清空，查找，遍历等基本集合操作。下面从 `add()` 开始，通过源码更加深刻的理解 ArrayList 的实现。

#### add()

```java
public void add(int index, E element) {
    rangeCheckForAdd(index); // 边界检测

    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1,
                         size - index); // 移动 index 之后的所有元素
    elementData[index] = element;
    size++;
}
```

`rangeCheckForAdd()` 这个方法在后面也会用到很多次，主要做边界检测，当 `index` 大于 `size` 或者小于 0 时，都会抛出 `IndexOutOfBoundsException` 异常。

第二步 `ensureCapacityInternal()` 的作用是保证集合的空间足以继续添加元素，空间不足时会自动扩容。这个方法很重要，可以说是 ArrayList 的核心了。我们来看一下到底是如何扩容的。

```java
   private void ensureCapacityInternal(int minCapacity) {
        ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
    }

    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity); // 扩容
    }
```

通过 `calculateCapacity()` 方法计算合适的最少空间：

```java
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity); // 如果当前是空数组，取 minCapacity 和 10 的较大值
    }
    return minCapacity;
}
```

如果初始化时没有指定集合大小，则取 `DEFAULT_CAPACITY`（等于10）和 `minCapacity` 的较大值。所以，如果我们构建了一个空 ArrayList，当我们添加第一个元素的时候，就会默认扩容至 10 。

当 `minCapacity` 大于当前数组长度时，就需要扩容了，`grow()` 方法就是扩容的具体实现：

```java
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length; // 原数组大小
    int newCapacity = oldCapacity + (oldCapacity >> 1); // 扩容至原来的 1.5 倍
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity); // 容量最大最大只能是 Integer.MAX_VALUE
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

每次扩容后为原来容量的 1.5 倍，所以当我们可以预估元素数量的时候，直接在构造函数中指定，就可以节约空间了。如果扩容后的新容量大于 `MAX_ARRAY_SIZE`，即 `Integer.MAX_VALUE - 8`，调用 `hugeCapacity()` 方法再做一次判断。

```java
private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError(); // 小于 0，即发生溢出，抛出 OOM
    return (minCapacity > MAX_ARRAY_SIZE) ? // 最大只可能为 Integer.MAX_VALUE
        Integer.MAX_VALUE : MAX_ARRAY_SIZE;
    }
```

最后使用 `Arrays.copyOf()` 方法创建新数组：

```java
public static <T,U> T[] copyOf(U[] original, int newLength, Class<? extends T[]> newType) {
    @SuppressWarnings("unchecked")
    T[] copy = ((Object)newType == (Object)Object[].class)
        ? (T[]) new Object[newLength]
        : (T[]) Array.newInstance(newType.getComponentType(), newLength);
    System.arraycopy(original, 0, copy, 0,
                    Math.min(original.length, newLength));
    return copy;
}
```

扩容完成之后，就可以愉快的添加元素了，直接给 `elementData[size++]` 赋值即可。

一个参数的 `add(E element)` 方法是在数组尾部添加元素，除此之外，ArrayList 还支持在指定位置添加元素，`add(int index, E element)`:

```java
public void add(int index, E element) {
    rangeCheckForAdd(index); // 边界检测

    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1,
                    size - index); // 移动 index 之后的所有元素
    elementData[index] = element;
    size++;
}
```

在指定位置 index 处插入一个元素，就需要把 index 后面的元素都依次往后移动，给要添加的元素腾出来位置，所以 ArrayList 的插入操作并不是那么的高效。

#### remove()

`remove()` 方法也有两个，第一个是移除指定位置的元素：

```java
public E remove(int index) {
    rangeCheck(index); // 边界检测

    modCount++;
    E oldValue = elementData(index);

    int numMoved = size - index - 1;
    if (numMoved > 0) // 移动 index 之后的所有元素
        System.arraycopy(elementData, index+1, elementData, index,
                        numMoved);
    elementData[--size] = null; // clear to let GC do its work

    return oldValue;
}
```

逻辑比较简单，将 index 之后的所有元素都依次往前移动，注意在完成移动之后，将集合尾部元素置空，以便 GC 回收。和插入一样，`ArrayList` 的删除也不是那么的高效，时间复杂度都是 `O(n)` 。

第二个是移除指定元素：

```java
public boolean remove(Object o) { // 如有多个，仅移除第一个
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}
```

这里要注意一点，当集合中存在重复元素时，无论是 null 还是其他对象，`remove()` 方法只会移除其中的第一个。这里用的移除用的是 `fastRemove()` 方法，其实和普通的 `remove()` 方法没什么区别，只是取消了边界检测，且没有返回值，所以更 fast 一点。

```java
/*
* Private remove method that skips bounds checking and does not
* return the value removed.
* 取消边界检查，且不返回 remove 掉的值
*/
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                        numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```

#### removeAll() && retainAll()

```java
public boolean removeAll(Collection<?> c) {
    Objects.requireNonNull(c);
    return batchRemove(c, false);
}
```

`removeAll()` 方法是移除所有包含在集合 c 中的元素，调用 `batchRemove()` 实现。

```java
public boolean retainAll(Collection<?> c) {
    Objects.requireNonNull(c);
    return batchRemove(c, true);
}
```

`retainAll()` 方法正好与 `removeAll()` 相反，是保留所有包含在集合 c 中的元素，移除其他元素，也是调用 `batchRemove()` 实现。

`batchRemove()` 方法应该是 ArrayList 中比较复杂的一个方法了，但是绝对值得仔细一看。

```java
/**
*
* @param c 集合
* @param complement 为 true 时，保留指定集合中的值，为 false 时，删除指定集合中的值
* @return 数组中重复的元素都会被删除，只要发生删除就会返回 true
*/
private boolean batchRemove(Collection<?> c, boolean complement) {
    final Object[] elementData = this.elementData;
    int r = 0, w = 0;
    boolean modified = false;
    try {
        // 遍历数组，并检查这个集合是否包含对应的值，移动要保留的值到数组前面，w 最终值为要保留的元素的数量
        // 也就是说，如果是 retainAll()，就将相同元素移动到数组前面。
        // 如果是 removeAll()，就将不同元素移动到数组前面
        for (; r < size; r++)
            if (c.contains(elementData[r]) == complement)
                elementData[w++] = elementData[r];
    } finally {
        // Preserve behavioral compatibility with AbstractCollection,
        // even if c.contains() throws.
        if (r != size) { // r != size，说明发生异常，循环未执行完成
            System.arraycopy(elementData, r,
                            elementData, w,
                            size - r); // 将 r 之后的元素移动过去
            w += size - r;
        }
        // w == size 说明保留全部元素，modified 返回 false
        if (w != size) {
            // clear to let GC do its work
            for (int i = w; i < size; i++)
                elementData[i] = null;
            modCount += size - w; // 更新 modCount
            size = w; // w 就是要保存的元素个数
            modified = true;
        }
    }
    return modified;
}
```

总之，不管你是 `removeAll()` 还是 `retainAll()`，我 `batchRemove()` 一律把要保留的元素移到前面，要删掉的元素扔后面，并记录下面要保留元素的个数。

#### 其他
**后面的方法都很简单直白，大致浏览一下就可以了。**

```java
// 获取集合大小
public int size() {
    return size;
}

// 判断集合是否为空
public boolean isEmpty() {
    return size == 0;
}

// 获取元素下标
public int indexOf(Object o) {
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}

// 设置指定位置的元素
public E set(int index, E element) {
    rangeCheck(index);

    E oldValue = elementData(index);
    elementData[index] = element;
    return oldValue;
}

// 获取指定位置的元素
public E get(int index) {
    rangeCheck(index);

    return elementData(index);
}

// 清空集合
public void clear() {
    modCount++;

    // clear to let GC do its work
    for (int i = 0; i < size; i++)
        elementData[i] = null;

    size = 0;
}
```

## 总结

简单总结一下 ArrayList：

* 基于动态数组实现，自动扩容每次增长为原来的 1.5 倍
* 在内存中是连续的，具备随机访问能力
* 根据下标获取元素的时间复杂度是 `O(1)`
* 添加元素和删除元素的平均时间复杂度是 `O(n)`
* 允许重复元素，允许 null 值，线程不安全

既然标题是 `走进 JDK 之 ArrayList（一）`，那么肯定还有二嘛。如果你有认真看 ArrayList 源码，你会发现一个经常出现的字段 `modCount`，字面意思就是修改次数。基本但凡涉及到修改集合的方法，大多都会执行 `modCount++` 操作，以 `clear()` 方法为例：

```java
public void clear() {
    modCount++;
    // clear to let GC do its work
    for (int i = 0; i < size; i++)
        elementData[i] = null;
    size = 0;
}
```

那么，这个 `modCount` 究竟有什么作用，这便是 `走进 JDK 之 ArrayList（二）` 所要详细说明的。

> 文章首发微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解。
>
> 更多 JDK 源码解析，扫码关注我吧！

![](https://user-gold-cdn.xitu.io/2019/4/27/16a5f352eab602c4?w=2800&h=800&f=jpeg&s=178470)
