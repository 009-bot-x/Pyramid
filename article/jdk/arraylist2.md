上篇文章 [走进 JDK 之 ArrayList（一）](https://juejin.im/post/5cc070a2f265da037a3cee37) 简单分析了 ArrayList 的源码，文末留下了一个问题，`modCount` 是干啥用的？下面我们通过一个小例子来引出今天的内容。

```java
public static void main(String[] args){
    List<String> list= new ArrayList<>();
    list.add("java");
    list.add("kotlin");
    list.add("dart");

    for (String s:list){
        if (s.equals("dart"))
            list.remove(s);
    }
}
```

大多数人应该都这么干过，然后得到一个鲜红的 `ConcurrentModificationException`，具体错误堆栈信息如下：

```java
Exception in thread "main" java.util.ConcurrentModificationException
	at java.util.ArrayList$Itr.checkForComodification(ArrayList.java:909)
	at java.util.ArrayList$Itr.next(ArrayList.java:859)
	at collection.ArrayListTest.main(ArrayListTest.java:15)
```

报错位置是 `ArrayList` 的内部类 `Itr` 中的 `checkForComodification()` 方法。至于如何调用到这个方法的，我们首先得知道上面的代码中发生了什么。看字节码的话太麻烦了又不容易理解，推荐一个反编译神器 `jad`，javac 编译得到 class 文件之后执行如下命令：

```bash
./jad ArrayListTest.class
```

得到 ArrayListTest.jad 文件，直接用文本编辑器打开即可：

```java
public class ArrayListTest {

    public ArrayListTest() { }

    public static void main(String args[]) {
        ArrayList arraylist = new ArrayList();
        arraylist.add("java");
        arraylist.add("kotlin");
        arraylist.add("dart");
        Iterator iterator = arraylist.iterator(); // 1
        do {
            if(!iterator.hasNext()) // 2
                break;
            String s = (String)iterator.next(); // 3
            if(s.equals("dart"))
                arraylist.remove(s);
        } while(true);
    }
}
```

从反编译得到的代码我们可以发现，增强型 for 循环只是一个语法糖而已，编译器帮我们进行了处理，其实是调用了迭代器来进行循环。着重看一下上面标注的三句代码，是整个迭代过程的核心。

第一句，获取 ArrayList 的迭代器。

```java
public Iterator<E> iterator() {
    return new Itr();
}
```

`AbstractList` 中定义了一个迭代器 `Itr`，但是它的子类 ArrayList 并没有直接使用父类的迭代器，而是自己定义了一个优化版本的 `Itr`。循环体中第二句代码首先会判断是否 `hasNext()`，存在的话调用 `next` 获取元素，不存在的话跳出循环。增强型 for 循环的基本实现就是这样的。`hasNext()` 和 `next()` 方法源码如下：

```java
private class Itr implements Iterator<E> {
    int cursor;       // index of next element to return
    int lastRet = -1; // index of last element returned; -1 if no such
    int expectedModCount = modCount;

    Itr() {}

    public boolean hasNext() {
        return cursor != size;
    }

    @SuppressWarnings("unchecked")
    public E next() {
        checkForComodification(); // 并发检测
        int i = cursor;
        if (i >= size) // 判断是否越界
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;
        if (i >= elementData.length) // 再次判断，如果越界，可能是并发修改导致
            throw new ConcurrentModificationException();
        cursor = i + 1;
        return (E) elementData[lastRet = i];
    }

    ......
    // 省略其他代码
    }
```

`cursur` 表示当前游标位置，`hasNext()` 方法就是根据 cursor 是否等于集合大小 `size` 判断是否还有下一个元素。成员变量中有个 `expectedModCount`，定义如下：

```java
int expectedModCount = modCount;
```

终于发现了 `modCount` 的踪影，它被赋值给了 `expectedModCount` 变量，字面意思就是 `期望的修改次数`。具体它有什么用，接着看 `next()` 方法中的第一行代码，调用了 `checkForComodification()` 方法，这是用来做并发检测的：

```java
final void checkForComodification() {
    if (modCount != expectedModCount) // 在迭代的过程中 modCount 发生了改变
        throw new ConcurrentModificationException();
}
```

异常就是这样抛出来的，`modCount` 和 `expectedModCount` 不相等，即实际的修改次数与期望的修改次数不相等。`expectedModCount` 是在迭代器初始化的过程中赋值的，其值等于 `modCount`。在迭代过程中又不相等了，那就只可能是在迭代过程中修改了集合，造成了 `modCount` 变化。那么，哪些操作会导致 `modCount` 发生变化呢？JDK 源码注释中做了以下说明（modCount 在 AbstractList 中声明）：

> The number of times this list has been structurally modified. Structural modifications are those that change the size of the list, or otherwise perturb it in such a fashion that iterations in progress may yield incorrect results.

集合的结构修改次数。结构修改指的是集合大小的变化。所以只要是涉及到增加或者删除元素的方法，都要改变 `modCount`。以 ArrayList 的 remove() 方法为例：

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

通过 `modCount++` 使其自增 1。

由于 ArrayList 并不是线程安全的，一边迭代一边改变集合，的确可能导致多线程下代码表现不一致。可能有人会有这样的疑问，文章开头的测试代码并没有涉及到并发操作啊，为什么还是抛出了异常？这就是集合的 `fail-fast(快速失败)` 机制。

`fail-fast` 错误机制并不保证错误一定会发生，但是当错误发生的时候一定可以抛出异常。它不管你是不是真的并发操作，只要可能是并发操作，就给你提前抛出异常。针对非线程安全的集合类，这是一种健壮的处理方式。但是你如果真的想在单线程中这样操作应该怎么办？没关系，让 `modCount` 和 `expectedModCount` 相等就完事了，ArrayList 的迭代器为我们提供了这样的 `add()` 和 `remove()` 方法：

```java
public void add(E e) {
    checkForComodification();

    try {
        int i = cursor;
        ArrayList.this.add(i, e); // add 之后要修改 modCount
        cursor = i + 1;
        lastRet = -1;
        expectedModCount = modCount;
    } catch (IndexOutOfBoundsException ex) {
        throw new ConcurrentModificationException();
    }
}

public void remove() {
    if (lastRet < 0)
        throw new IllegalStateException();
    checkForComodification();

    try {
        ArrayList.this.remove(lastRet); // remove 之后要修改 modCount
        cursor = lastRet;
        lastRet = -1;
        expectedModCount = modCount;
    } catch (IndexOutOfBoundsException ex) {
        throw new ConcurrentModificationException();
    }
}
```

上面的代码实现在修改了集合结构之后都会给 `expectedModCount` 重新赋值，使其与 `modCount` 相等。修改一下文章开头的测试代码：

```java
public static void main(String[] args){
    List<String> list= new ArrayList<>();
    list.add("java");
    list.add("kotlin");
    list.add("dart");

//  for (String s:list){
//      if (s.equals("dart"))
//          list.remove(s);
//  }

    Iterator<String> iterator=list.iterator();
    while (iterator.hasNext()){
        String s= iterator.next();
        if (s.equals("dart"))
            iterator.remove();
    }
}
```

这样就不会再报错了。

最后最后再给你出一道题，仔细看一下：

```java
public static void main(String[] args) {
    List<String> list = new ArrayList<>();
    list.add("java");
    list.add("kotlin");
    list.add("dart");

    for (String s : list) {
        if (s.equals("kotlin"))
            list.remove(s);
    }
}
```

如果没看出来和文章开头那道题的区别，那就再翻上去仔细观察一下。之前我们要删的是 `dart`，集合中的最后一个元素。现在要删的是 `kotlin`，集合中的第二个元素。执行结果会怎么样？你要是精通脑筋急转弯的话，肯定能给出正确答案。没错，这次成功删除了元素并且没有任何异常。这是为什么呢？删除 `dart` 就报异常，删除 `kotlin` 就没问题，这是歧视 `dart` 吗。再把迭代器的代码掏出来：

```java
public boolean hasNext() {
    return cursor != size;
}

@SuppressWarnings("unchecked")
public E next() {
    checkForComodification(); // 并发检测
    int i = cursor;
    if (i >= size) // 判断是否越界
        throw new NoSuchElementException();
    Object[] elementData = ArrayList.this.elementData;
    if (i >= elementData.length) // 再次判断，如果越界，可能是并发修改导致
        throw new ConcurrentModificationException();
    cursor = i + 1;
    return (E) elementData[lastRet = i];
}
```

集合中添加了 3 个元素，所以初始化迭代器之后，`expectedModCount = modCount = 3`，`cursor` 此时为 0 。先来分析文章开头的代码，删除集合中最后一个元素的情况：

* 执行完第一次循环，`cursor` 为 1，未产生删除操作，`modCount` 为 3，`expectedModCount` 为 3，`size` 为 3。`cursor != size`，`hasNext()` 判断还有元素。
* 执行完第二次循环，`cursor` 为 2，仍未产生删除操作，`modCount` 为 3，`expectedModCount` 为 3，`size` 为 3。`cursor != size`，`hasNext()` 判断还有元素。
* 执行完第三次循环，`cursor` 为 3，由于产生删除了操作，`modCount` 为 4，`expectedModCount` 仍为 3，`size` 变为 2。`cursor != size`，`hasNext()` 判断还有元素，继续迭代，其实已经没有元素了。
* 继续迭代，调用 `next()` 方法，此时 `expectedModCount != modCount`，直接抛出异常。

|循环次数 | cursor | modCount | expectedModCount | size |
| :---: |:---: |:---: |:---: |:---: |
| 1 | 1 | 3 | 3 | 3 |
| 2 | 2 | 3 | 3 | 3 |
| 3 | 3 | 4 | 3 | 2 |

再来看看删除 `kotlin` 的执行流程：

* 执行完第一次循环，`cursor` 为 1，未产生删除操作，`modCount` 为 3，`expectedModCount` 为 3，`size` 为 3。`cursor != size`，`hasNext()` 判断还有元素。
* 执行完第二次循环，`cursor` 为 2，产生删除操作，`modCount` 为 4，`expectedModCount` 为 3，`size` 为 2。`cursor == size`，`hasNext()` 判断没有元素了，不再调用 `next()` 方法。

并不是 `fail-fast` 失效了，仅仅只是恰好 `cursor == size`，`hasNext()` 方法误以为集合中已经没有元素了，其实还有一个元素。循环两次之后就终止循环了，不再调用 `next()` 方法，也就不存在并发检测了。

|循环次数 | cursor | modCount | expectedModCount | size |
| :---: |:---: |:---: |:---: |:---: |
| 1 | 1 | 3 | 3 | 3 |
| 2 | 2 | 3 | 3 | 2 |


本文由一个 `ConcurrentModificationException` 的例子，顺藤摸瓜，解析了 ArrayList 迭代器的源码，同时说明了 Java 集合框架的 `fail-fast` 机制。最后也验证了增强型 for 循环中删除元素并不是百分之百会触发 `fail-fast`。

`ArrayList` 就说到这里了，下一篇来看看 `List` 中同样重要的 `LinkedList`。

> 文章首发微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解。
>
> 更多 JDK 源码解析，扫码关注我吧！

![](https://user-gold-cdn.xitu.io/2019/4/27/16a5f352eab602c4?w=2800&h=800&f=jpeg&s=178470)
