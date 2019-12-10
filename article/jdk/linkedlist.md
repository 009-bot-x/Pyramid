> 走进 JDK 系列第 15 篇

## 概述

如果你了解链表的基本结构的话，`LinkedList` 的源码其实还是比较容易理解的。`LinkedList` 是基于双向链表实现的，与 `ArrayList` 不同的是，它在内存中不占用连续的内存空间，相连元素之间通过 “链” 来链接。对于单链表，每个节点有一个 **后继指针** 指向下一个节点。对于双向链表来说，除了后继指针外，它还要一个 **前驱指针** 指向前一个节点。那么，双向链表有什么好处呢？既然有了前驱指针，在遍历的时候就可以向前遍历，在下面的源码分析中可以看到，这是单链表所不具备的功能。

前面介绍 ArrayList 的时候说过，数组具备随机访问能力，其根据下标随机访问的时间复杂度是 `O(1)`。同样，为了保证内存的连续性，其 `插入` 和 `删除` 操作就相对低效的多。而链表正好与其相反，其不具备随机访问能力，但是 `插入` 和 `删除` 就相对高效，仅仅只需修改 `后继指针` 和 `前驱指针` 指向的节点即可。其插入和删除操作的时间复杂度均为 `O(1)`，但这个 `O(1)` 其实也不是很严谨。删除指定节点，还是删除值等于给定值的节点，单链表还是双向链表，其实时间复杂度的表现都是不一样的，下面的源码解析中也会有所体现。好了，关于链表就说这么多了，下面来进入 `LinkedList` 的源码分析。

首先看一下 `LinkedList` 的 UML 图：


![](https://user-gold-cdn.xitu.io/2019/5/8/16a97d62835b4d88?w=882&h=588&f=png&s=26294)

## 源码解析

### 类声明

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable {}
```

* 继承了抽象类 `AbstractSequentialList`，它提供了一些集合类型无关的基本方法的实现，如 `get` `set` `add` `remove` 等。通常实现它的集合类型不具备随机访问能力，这和 `AbstractList` 是相对立的。

* 实现了 `List` 接口
* 实现了 `Deque` 接口，说明也可以当做一个双端队列，源码中也实现了相关方法。
* 实现了 `Cloneable` 接口，提供克隆能力。和 ArrayList 一样，也是浅拷贝。
* 实现了 `Serializable` 接口，提供序列化能力

### 成员变量

```java
transient int size = 0; // 表示链表大小
transient Node<E> first; // 头结点
transient Node<E> last; // 尾节点
private static final long serialVersionUID = 876323262645176354L;
```

顺便看一下结点 `Node` 类的定义：

```java
private static class Node<E> {
    E item;
    Node<E> next; // 后继指针
    Node<E> prev; // 前驱指针

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

每个结点都包含指向前一个结点的前驱指针 `prev`，和指向后一个结点的后继指针 `next`。一般头结点的前驱指针和尾节点的后继指针都指向 null。

### 构造函数

```java
public LinkedList() { }

public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
```

第一个是默认的无参构造，会构建一个空链表。第二个根据参数中的集合 通过 `addAll()` 方法构造链表，链表中的元素顺序根据集合 `c` 的迭代器的迭代顺序。

### 方法

你可以翻一下 `LinkedList` 的 API 列表，提供了许许多多的方法，其实很多都是重复的。它实现了 `Deque` 接口中的方法，重写了 `AbstractSequentialList` 类的方法，总结一下就是各种增删改查操作，最后调用的都是自己的私有方法 `linkxxx/unlinkxxx` 等，很好的做到了隔离，也是一种值得学习的设计思想。下面就优先来看一下这些方法。

#### linkFirst(E e)

```java
// 将 e 置为头结点
private void linkFirst(E e) {
    final Node<E> f = first;
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null) // 如果链表为空， e 节点就是尾节点
        last = newNode;
    else
        f.prev = newNode; // 链表不为空，插入 e 节点，并将原头结点的 prev 指向 e 节点
    size++;
    modCount++;
}
```

链表为空的话，插入的 e 既是头结点也是尾节点。

链表不为空的话，就移动原头结点的前驱指针。

#### linkLast(E e)

```java
// 将 e 置为尾结点
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode; // 如果链表为空， e 节点就是头节点
    else
        l.next = newNode; // 链表不为空，插入 e 节点，并将原尾结点的 next 指向 e 节点
    size++;
    modCount++;
}
```

链表为空的话，插入的 e 既是头结点也是尾节点。

链表不为空的话，就移动原尾结点的后继指针。

#### linkBefore(E e, Node<E> succ)

```java
// 在节点 succ 前插入元素
void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ); // 构建新结点
    succ.prev = newNode; // 将 succ 的 prev 指针指向新结点
    if (pred == null) // pred 为 null，说明 succ 原来就是头结点，现在要更新头结点
        first = newNode;
    else
        pred.next = newNode; // 将 succ 的前一个结点的 next 指针指向新结点
    size++;
    modCount++;
}
```

原理也很简单，你可以想象成打上结的绳子，你只需要把 succ 结点打开，然后把需要插入的结点系上去就可以了，时间复杂度为 `O(1)`。当然，这是双向链表。对于单向链表还是 `O(1)` 吗？显然不是的，因为在单向链表中你没办法执行下面这行代码：

```java
final Node<E> pred = succ.prev;
```

也就是说你没办法直接拿到 succ 的前驱结点，也就没法直接将 succ 的前一个结点的 next 指针指向新结点。你只能通过遍历去获取前驱结点。所以，对于单链表来说，插入元素的时间复杂度还是 `O(n)`。

#### unlinkFirst(Node<E> f)

```java
// 移除头结点 f
private E unlinkFirst(Node<E> f) {
    // assert f == first && f != null;
    final E element = f.item;
    final Node<E> next = f.next;
    f.item = null;
    f.next = null; // help GC 头结点置空以便 GC
    first = next;
    if (next == null) // next 为空，说明链表原来只有一个元素
        last = null;
    else
        next.prev = null; // 将 next 的 prev 置空，此时 next 是头结点
    size--;
    modCount++;
    return element;
}
```

这里默认参数中的 f 结点就是头结点。如果链表中原本只有一个元素，那么头尾结点都要置空。如果多于一个元素，只要把头结点的下一个结点的前驱指针指向 null 就可以了。

#### unlinkLast(Node<E> l)

```java
// 移除尾节点 l
private E unlinkLast(Node<E> l) {
    // assert l == last && l != null;
    final E element = l.item;
    final Node<E> prev = l.prev;
    l.item = null;
    l.prev = null; // help GC 头结点置空以便 GC
    last = prev;
    if (prev == null) // prev 为空，说明链表原来只有一个元素
        first = null;
    else
        prev.next = null; // 将 prev 的 next 置空，此时 prev 是尾结点
    size--;
    modCount++;
    return element;
}
```

和 `unlinkfirst` 基本一致，只要把尾结点的前一个结点的后继指针指向 null 就可以了。

#### unlink(Node<E> x)

```java
// 移除指定非空节点 x
E unlink(Node<E> x) {
    // assert x != null;
    final E element = x.item;
    final Node<E> next = x.next;
    final Node<E> prev = x.prev;

    if (prev == null) { // x 是头结点
        first = next;
    } else {
        prev.next = next; // 将 x 的 next 指向 x 后面一个节点
        x.prev = null;
    }

    if (next == null) { // x 是尾节点
        last = prev;
    } else {
        next.prev = prev; // 将 x 的 prev 指向 x 前面一个节点
        x.next = null;
    }

    x.item = null;
    size--;
    modCount++;
    return element;
}
```

代码也比较简单，同时修改前一个结点的后继指针和后一个结点的前驱指针就可以了，要注意参数结点是头结点或者尾节点的特殊情况。时间复杂度也是 `O(1)`，对于单链表是 `O(n)`。

上面的插入和删除都是针对指定结点的，还有一种情况是针对指定值的。比如，对于一个存储 int 值的链表，我要删除值为 1 的结点，其时间复杂度还是 `O(1)` 吗？下面来看看 `remove(Object o)` 方法。

#### remove(Object o)

```java
public boolean remove(Object o) {
    if (o == null) { // 删除 null 元素
        for (Node<E> x = first; x != null; x = x.next) {
            if (x.item == null) {
                unlink(x);
                return true;
            }
        }
    } else { // 删除非 null 元素
        for (Node<E> x = first; x != null; x = x.next) {
            if (o.equals(x.item)) {
                unlink(x);
                return true;
            }
        }
    }
    return false;
}
```

很显然，对于删除值等于指定值的结点，时间复杂度也是 `O(n)`。循环遍历得到该结点之后再调用 `unlink()` 方法去删除。还要注意一点，该方法仅仅删除第一次出现的值等于指定值的结点，链表是允许重复元素的。

说完了插入和删除，我们再来看看查找。虽说链表的查找操作必然是 `O(n)` 的，但是 `LinkedList` 还是对查找操作做了相应的优化。下面来看一下 `get()` 方法。

#### get(int index)

```java
// 返回指定位置的元素
public E get(int index) {
    checkElementIndex(index); // 边界检查
    return node(index).item; // 虽然时间复杂度仍然是 O(n)，但只需遍历一半的链表
}
```

`checkElementIndex` 检测 index 是否越界。`node()` 方法用来获取 index 处的指定结点。

```java
Node<E> node(int index) {
    // assert isElementIndex(index);
    // 根据下标是否小于 size/2，每次只遍历半个链表
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

得益于双向链表的特性，`LinkedList` 的查找每次只需遍历半个链表，虽然时间复杂度还是 `O(n)`，但是是有实际上的性能提升的。

`LinkedList` 的方法就说这么多了，虽说大部分方法都没提到，但是剩下的方法基本都是依靠上面解析过的这些方法来实现的，也就没有单独拿出来说的必要了。我在源码文件中都进行了注释，感兴趣的可以到我的 Github 查看 [LinkedList.java](https://github.com/lulululbj/jdk8u/blob/master/jdk/src/share/classes/java/util/LinkedList.java) 。

## 总结

`LinkedList` 基于双向链表实现，内存中不连续，不具备随机访问，插入和删除效率较高，查找效率较低。使用上没有大小限制，天然支持扩容。

允许 null 值，允许重复元素。和 ArrayList 一样，也是 fail-fast 机制。在 [走进 JDK 之 ArrayList（二）](https://juejin.im/post/5cc58c51f265da035d0c858e) 中已经详细说明过 fail-fast 机制，这里就不再赘述了。

双向链表由于可以反向遍历，相较于单向链表在某些操作上具有性能优势，但是由于每个结点都需要额外的内存空间来存储前驱指针，所以双向链表相对来说需要占用更多的内存空间，这也是 **空间换时间** 的一种体现。

> 文章首发微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解。
>
> 更多 JDK 源码解析，扫码关注我吧！

![](https://user-gold-cdn.xitu.io/2019/4/27/16a5f352eab602c4?w=2800&h=800&f=jpeg&s=178470)
