往期目录：

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
> [Android 逆向笔记 —— 一个简单 CrackMe 的逆向总结](https://juejin.im/post/5ce52ed951882532df70ab58)

## 概述

我们在解压缩 APK 文件之后，会看到一个叫做 `resources.arsc` 的文件，它的格式称之为 **ARSC 文件格式** 。那么它的作用是什么呢？大家对 R 文件肯定都十分熟悉，它存储了资源的 ID。在打包过程中，但凡使用到资源的地方都是使用这个 ID 来代替的。ARSC 文件就是一个资源索引表，它可以帮助系统根据资源 ID 快速找到资源。

当我们使用 ApkTool 反编译的时候，会在 `res/value` 目录下生成一个 `public.xml` 文件，里面就记录了资源项及其对应的 ID，如下图所示：


![](https://user-gold-cdn.xitu.io/2019/5/24/16ae9b6ebefee93f?w=904&h=599&f=png&s=184268)

虽然没有细看过 ApkTool 的源码，但我猜测这应该就是根据 ARSC 文件解析出来的。关于 ARSC 的文件结构，网上有一张很好的图片，拿过来给大家看一下：

![](https://user-gold-cdn.xitu.io/2019/5/24/16ae9b85b2f4e918?w=1796&h=1706&f=png&s=269227)

如果觉得有点绕，可以对照我画的思维导图来阅读后面的文章：


![](https://user-gold-cdn.xitu.io/2019/5/24/16ae9b95d7e92507?w=2333&h=2833&f=png&s=681137)

ARSC 文件格式的数据结构在 AOSP 中也有相应的定义，位于 [ResourceType.h](http://androidxref.com/9.0.0_r3/xref/frameworks/base/libs/androidfw/include/androidfw/ResourceTypes.h) 文件中 。整体上可以分为下面三大块：

* `ResTableHeader :`  文件头
* `ResStringPool ：` 资源项值字符串池
* `ResTablePackage ：` 数据块

其中 `ResTablePackage` 项最为复杂，包含了 ARSC 文件的数据块内容。其他两块内容较为简单。下面就来一一解析。

## ResTableHeader

```c++
struct ResTable_header
{
    struct ResChunk_header header;

    // The number of ResTable_package structures.
    uint32_t packageCount;
};
```

这里的 header 是 `ResChunk_header` 类型，我们先来看一下这个类，它在 ARSC 文件的其他部分也会出现很多次。其实 ARSC 文件和 AndroidManifest.xml 文件有一些类似，也是由一个一个 Chunk 组成的。每一个 Chunk 都有固定的 ResTable_header，具体格式如下：

```c++
struct ResChunk_header
{
    // Type identifier for this chunk.  The meaning of this value depends
    // on the containing chunk.
    uint16_t type;

    // Size of the chunk header (in bytes).  Adding this value to
    // the address of the chunk allows you to find its associated data
    // (if any).
    uint16_t headerSize;

    // Total size of this chunk (in bytes).  This is the chunkSize plus
    // the size of any data associated with the chunk.  Adding this value
    // to the chunk allows you to completely skip its contents (including
    // any child chunks).  If this value is the same as chunkSize, there is
    // no data associated with the chunk.
    uint32_t size;
};

```

`type` 是该 Chunk 的标识符，不同的 Chunk 都有自己的标识符。`headerSize` 表示当前 Chunk Header 的大小。`size` 表示当前 Chunk 的大小。

ResChunkHeader 的结构还是很简单的，我们再回到 ResTableHeader。它除了 `header` 字段之外，还有一个 `packageCount` 字段，表示 ARSC 文件 ResTablePackage 的个数，即数据块的个数，通常是 1。

## ResStringPool

`ResTableHeader` 后面紧接着的就是 `ResStringPool`，存放了 APK 中所有资源项值的字符串内容。我这里就不贴 ResStringPool 的结构图了，建议阅读的时候直接对照着我上面给的思维导图，或者对照着 010 Editor 的解析结果。

ResStringPool 也有一个头，叫做 `ResStringPoolHeader`，其格式如下：

```c++
struct ResStringPool_header
{
    struct ResChunk_header header;

    // Number of strings in this pool (number of uint32_t indices that follow
    // in the data).
    uint32_t stringCount;

    // Number of style span arrays in the pool (number of uint32_t indices
    // follow the string indices).
    uint32_t styleCount;

    // Flags.
    enum {
        // If set, the string index is sorted by the string values (based
        // on strcmp16()).
        SORTED_FLAG = 1<<0,

        // String pool is encoded in UTF-8
        UTF8_FLAG = 1<<8
    };
    uint32_t flags;

    // Index from header of the string data.
    uint32_t stringsStart;

    // Index from header of the style data.
    uint32_t stylesStart;
};
```

有六个字段，来分别看一下：

* `header :` ResChunkHeader，其 type 是 `RES_STRING_POOL_TYPE`
* `stringCount :` 字符串个数
* `styleCount :` 字符串样式个数
* `flags :` 字符串的属性，可取值包括0x000(UTF-16),0x001(字符串经过排序)、0X100(UTF-8)和他们的组合值
* `stringsStart :` 字符串内容偏移量
* `stylesStart :` 字符串样式内容偏移量

`ResStringPoolHeader` 之后跟着的是两个偏移量数组 `stringOffsets` 和 `styleOffsets`，分别是字符串内容偏移量数组和字符串样式内容偏移量数组。上面提到的偏移量都是相对整个 ResStringPool 的。根据起始偏移量和每个字符串的偏移量数组，我们就可以获取到所有字符串了。注意这里的字符串并不是纯粹的字符串，它也是有结构的。 `u16len` 和 `u8len`，分别代表 `UTF-8` 和 `UTF-16` 下的字符串长度。那么如何区分呢？之前的 ResStringPoolHeader 中的 `flags` 属性就标记了编码格式。如果是 utf-8，则字符串以 `0x00` 结尾，开头前两个字节分别表示 u8len 和 u16len。如果是 utf-16，则字符串以 `0x0000` 结尾，开头前两个字节表示 `u16len`，没有 `u8len` 字段。

下面简单看一下解析代码：

```java
private ResStringPoolHeader parseStringPoolType(List<String> stringPoolList) {
    int currentPosition = reader.getCurrentPosition();
    ResStringPoolHeader stringPoolHeader = new ResStringPoolHeader();
    try {

        stringPoolHeader.parse(reader);
        List<Integer> stringOffsets = new ArrayList<>(stringPoolHeader.stringCount);
        for (int i = 0; i < stringPoolHeader.stringCount; i++) {
            int offset = reader.readInt();
            stringOffsets.add(offset);
        }

        List<Integer> styleOffsets = new ArrayList<>();
        for (int i = 0; i < stringPoolHeader.styleCount; i++) {
            styleOffsets.add(reader.readInt());
        }

        int position = reader.getCurrentPosition();
            for (int i = 0; i < stringPoolHeader.stringCount; i++) {
                int length = 0;
                int skipLength = 0;
                if (stringPoolHeader.flags == ResStringPoolHeader.UTF8_FLAG) {
                    int u16len = reader.read(position + stringOffsets.get(i), 1)[0];
                    int u8len = reader.read(position + stringOffsets.get(i), 1)[0];
                    length = u8len;
                    skipLength = 1; // 如果是 utf-8，则字符串以 0x00结尾
                } else {
                    int u16len =reader.readUnsignedShort();
                    length = u16len;
                    skipLength = 2; // 如果是 utf-16，则字符串以 0x0000结尾
                }
                String string = "";
                try {
                    string = new String(reader.read(position + stringOffsets.get(i) + 2, skipLength*length));
                    reader.skip(skipLength);
                } catch (Exception e) {
                    log("   parse string[%d] error!", i);
                }

                stringPoolList.add(string);
                log("   stringPool[%d]: %s", i, string);
            }

            for (int i = 0; i < stringPoolHeader.styleCount; i++) {
                int index = reader.readInt();
                int firstChar = reader.readInt();
                int lastChar = reader.readInt();
                ResSpanStyle resSpanStyle = new ResSpanStyle(index, firstChar, lastChar);
                log(resSpanStyle.toString());
                reader.skip(4); // 0xffff
            }
            reader.moveTo(currentPosition + stringPoolHeader.resChunkHeader.size);
            return stringPoolHeader;
    } catch (IOException e) {
        log("   parse string pool type error!");
    }
    return null;
}
```

我拿我自己的 [Wanandroid](https://github.com/lulululbj/wanandroid) 安装包解压得到的 ARSC 文件做测试，一共打印了 `2411` 个字符串，如下图所示：



![](https://user-gold-cdn.xitu.io/2019/5/25/16aef52652700a9a?w=776&h=311&f=png&s=51912)

看完了字符串池，接下来就是最最重要的资源数据块 `ResTablePackage` 了。

## ResTablePackage

从文章开头给出的思维导图就可以看出来，`ResTablePackage` 占据了 ARSC 文件内容的大半壁江山。ResTablePackage 又可以分为五小块，如下所示：

* `ResTablePackageHeader :` 头信息
* `typeStrings :` 资源类型字符串池
* `keyStrings ：` 资源项名称字符串池
* `ResTableTypeSpec ：` 资源表规范
* `ResTableType ：` 资源表类型配置

下面来一一进行解析。

### ResTablePackageHeader

其实我只是为了保持名称统一才取了个名字叫 ResTablePackageHeader，但是在 AOSP 中是叫做 `ResTable_package`，其内容如下所示：

```c++
struct ResTable_package
{
    struct ResChunk_header header;

    // If this is a base package, its ID.  Package IDs start
    // at 1 (corresponding to the value of the package bits in a
    // resource identifier).  0 means this is not a base package.
    uint32_t id;

    // Actual name of this package, \0-terminated.
    uint16_t name[128];

    // Offset to a ResStringPool_header defining the resource
    // type symbol table.  If zero, this package is inheriting from
    // another base package (overriding specific values in it).
    uint32_t typeStrings;

    // Last index into typeStrings that is for public use by others.
    uint32_t lastPublicType;

    // Offset to a ResStringPool_header defining the resource
    // key symbol table.  If zero, this package is inheriting from
    // another base package (overriding specific values in it).
    uint32_t keyStrings;

    // Last index into keyStrings that is for public use by others.
    uint32_t lastPublicKey;

    uint32_t typeIdOffset;
};
```

* `header :` ResChunkHeader , 其 type 是 `RES_TABLE_PACKAGE_TYPE`
* `id :` 包的 ID, 等于 Package Id,一般用户包的 Package Id 为 `0X7F`, 系统资源包的 Package Id 为 `0X01`。
* `name :` 包名
* `typeStrings ：`  资源类型字符串池在 ResTablePackage 中的偏移量
* `lastPublicType ：` 一般资源类型字符串资源池的元素个数
*  `keyStrings ：` 资源名称字符串池在 ResTablePackage 中的偏移量
*  `lastPublicKey ：` 一般指资源项名称字符串资源池的元素个数。
*  `typeIdOffset ：` 未知，值为 0

### typeStrings

`typeStrings` 是资源类型字符串池，既然是资源类型，很容易就想到 `string` 、`layout` 、`drawable` 、`mipmap` 等等，这些都是资源类型。说直白点，就是通常写代码时候的 `R.` 后面跟的东西。`typeStrings` 就是一个 `ResStringPool`，所以它的解析方式和之前是一模一样的。直接看一下解析结果：

```
typeStrings:
   stringPool[0]: a n i m
   stringPool[1]: a n i m a t o r
   stringPool[2]: a t t r
   stringPool[3]: b o o l
   stringPool[4]: c o l o r
   stringPool[5]: d i m e n
   stringPool[6]: d r a w a b l e
   stringPool[7]: i d
   stringPool[8]: i n t e g e r
   stringPool[9]: i n t e r p o l a t o r
   stringPool[10]: l a y o u t
   stringPool[11]: m e n u
   stringPool[12]: m i p m a p
   stringPool[13]: r a w
   stringPool[14]: s t r i n g
   stringPool[15]: s t y l e
```

这里是 utf-16 编码的。

### keyStrings

`keyStrings` 是资源项名称字符串池，它也是 `ResStringPool`，就不再多说了，直接看解析结果：

```
keyStrings:
   stringPool[0]: abc_fade_in
   stringPool[1]: abc_fade_out
   stringPool[2]: abc_grow_fade_in_from_bottom
   stringPool[3]: abc_popup_enter
   stringPool[4]: abc_popup_exit
   stringPool[5]: abc_shrink_fade_out_from_bottom
   stringPool[6]: abc_slide_in_bottom
   stringPool[7]: abc_slide_in_top
   stringPool[8]: abc_slide_out_bottom
   stringPool[9]: abc_slide_out_top
   stringPool[10]: abc_tooltip_enter
   stringPool[11]: abc_tooltip_exit
   stringPool[12]: btn_checkbox_to_checked_box_inner_merged_animation
   stringPool[13]: btn_checkbox_to_checked_box_outer_merged_animation
   stringPool[14]: btn_checkbox_to_checked_icon_null_animation
   ...
   ...
   ...
   stringPool[2322]: Widget.Support.CoordinatorLayout
   stringPool[2323]: leak_canary_LeakCanary.Base
   stringPool[2324]: leak_canary_Theme.Transparent
```

资源项名称字符串池 keyStrings 之后是 `ResTableTypeSpec` 和 `ResTableType` ，它们是不定的交叉出现的。我们先来看看 `ResTableTypeSpec`。

### ResTableTypeSpec

`ResTableTypeSpec` 是资源表规范，用来描述资源项的配置差异性。系统根据不同设备的配置差异就可以加载不同的资源项。该部分数据结构对应结构体 `ResTable_typeSpec` :

```c++
struct ResTable_typeSpec
{
    struct ResChunk_header header;

    // The type identifier this chunk is holding.  Type IDs start
    // at 1 (corresponding to the value of the type bits in a
    // resource identifier).  0 is invalid.
    uint8_t id;

    // Must be 0.
    uint8_t res0;
    // Must be 0.
    uint16_t res1;

    // Number of uint32_t entry configuration masks that follow.
    uint32_t entryCount;

    enum {
        // Additional flag indicating an entry is public.
        SPEC_PUBLIC = 0x40000000
    };
};
```

* `header` : ResChunkHeader，其 type 是 `RES_TABLE_TYPE_SPEC_TYPE`
* `id` : 标识资源的 Type ID, Type ID 是指资源的类型 ID 。资源的类型有 animator、anim、color、drawable、layout、menu、raw、string 和 xml 等等若干种，每一种都会被赋予一个 ID
* `res0` : must be 0
* `res1` : must be 0
* `entryCount` : 等于本类型的资源项个数,指名称相同的资源项的个数

紧接着后面的是 entryCount 个 uint_32 数组，数组每个元素都是用来描述资源项的配置差异性的。

### ResTableType

`ResTableType` 是资源项的具体信息，包括资源项的名称，类型，值和配置等等。对应结构体 `ResTable_type` ：

```c++
struct ResTable_type
{
    struct ResChunk_header header;

    enum {
        NO_ENTRY = 0xFFFFFFFF
    };

    // The type identifier this chunk is holding.  Type IDs start
    // at 1 (corresponding to the value of the type bits in a
    // resource identifier).  0 is invalid.
    uint8_t id;

    // Must be 0.
    uint8_t res0;
    // Must be 0.
    uint16_t res1;

    // Number of uint32_t entry indices that follow.
    uint32_t entryCount;

    // Offset from header where ResTable_entry data starts.
    uint32_t entriesStart;

    // Configuration this collection of entries is designed for.
    ResTable_config config;
};
```

* `header` : ResChunkHeader，其 type 是 `RES_TABLE_TYPE_TYPE`
* `id` : 标识资源的 Type ID, Type ID 是指资源的类型 ID 。资源的类型有 animator、anim、color、drawable、layout、menu、raw、string 和 xml 等等若干种，每一种都会被赋予一个 ID
* `res0` : must be 0
* `res1` : must be 0
* `entryCount` : 资源项的个数
* `entryStart` ：资源项相对于本结构的偏移量
* `config` : 资源的配置信息

`config` 之后是一个大小为 entryCount 的 uint32_t 数组，用于描述资源项数据库的偏移量。这个偏移量数组之后是一个 `ResTableEntry[]`，我们再来看一下这块内容。

`ResTableEntry` 是资源项数据，对应结构体 `ResTable_entry` ：

```c++
struct ResTable_entry
{
    // Number of bytes in this structure.
    uint16_t size;

    enum {
        // If set, this is a complex entry, holding a set of name/value
        // mappings.  It is followed by an array of ResTable_map structures.
        FLAG_COMPLEX = 0x0001,
        // If set, this resource has been declared public, so libraries
        // are allowed to reference it.
        FLAG_PUBLIC = 0x0002,
        // If set, this is a weak resource and may be overriden by strong
        // resources of the same name/type. This is only useful during
        // linking with other resource tables.
        FLAG_WEAK = 0x0004
    };
    uint16_t flags;

    // Reference into ResTable_package::keyStrings identifying this entry.
    struct ResStringPool_ref key;
};
```

* `size` : 该结构体大小
* `flags` : 标志位
* `key` : 资源项名称在资源项名称字符串资源池的索引

根据 flags 的不同，后面的数据结构也有所不同。如果 flags 包含 `FLAG_COMPLEX(0x0001)`，则该数据结构是 `ResTableMapEntry`，`ResTableMapEntry` 是继承自 `ResTableEntry` 的，在原有结构上多了两个 uint32_t 字段 `parent` 和 `count`。`parent` 表示父资源项的 ID。`count` 表示接下来有多少个 `ResTableMap`。`ResTableMap` 结构如下所示：

```c++
struct ResTable_map
{
    ResTable_ref name; // 资源名称
    Res_value value; // 资源值
}
```

再来看看 `ResValue` ：

```c++
struct Res_value {
    uint16_t size;
    uint8_t res0;
    uint8_t dataType;
    data_type data;
```

以上就是 flags 包含 `FLAG_COMPLEX(0x0001)`时表示的 `ResTableMapEntry` 的结构。如果不包含的话，就直接是 `Res_value`。

## 总结

最后关于 Package 数据块的内容其实说的比较简略，因为个人觉得了解了解就可以了。如果想要深入学习的话，个人十分推荐老罗的一篇文章，[Android应用程序资源的编译和打包过程分析](https://blog.csdn.net/luoshengyang/article/details/8744683)，写的相当的详细。

最后还是给出解析代码地址，[ResParser](https://github.com/lulululbj/android-reverse/blob/master/Parser/src/main/luyao/parser/arsc/ResParser.java) 。

> 文章首发微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解。
>
> 更多 JDK 源码解析，扫码关注我吧！

![](https://user-gold-cdn.xitu.io/2019/4/27/16a5f352eab602c4?w=2800&h=800&f=jpeg&s=178470)
