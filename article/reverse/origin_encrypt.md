近日听到坊间传闻，儿子早恋了，给小姐姐塞 “情书” 被班主任当场截获。儿子却不以为然，理直气壮的反驳：“现在凡事都得讲究证据，老师您不能污蔑我！这根本不是情书呀！” 。无奈之下，老师把情书转交给老父亲我，叫我自己处理。

## 凯撒密码

老父亲我是一位程序员，打开 “情书” 一看，好家伙。果然是亲生的（当然不是指早恋。。），写个情书都浓浓的程序员气息。


> LoryhwkuhhwklqjvwkhvxqwkhprrqdqgbrxWkhvxql
viruwkhgdbwkhprrqlviruwkhqljkwdqgbrxiruhyhu


别人可能看不懂，老父亲还能不知道亲生儿子的伎俩么。凭他那点三脚猫功夫，最多来个古典加密。密文里面重复出现的 `wkh` 一下就露了馅。这家伙大概就是把明文的所有字母都按一定数目挪了一挪。比如所有字母都往右挪一位，`a` 变成 `b`, `b` 变成 `c`,用这种方式处理一下 `I love you`，密文就是 `J mpwf zpv`，自然就看不懂在说什么了。

对于这种古老的加密方式，暴力破解就足够了。就拿密文中频繁出现的 `wkh` 作突破口，虽然我不知道应该往回挪动几位，可是总共就 26 个英文字母，我试个 26 次也就出来了。

| 密文 | 移动位数 | 原文 |
| :------: | :------: | :------: |
| wkh | 1 | vhg |
| wkh | 2 | uif |
| wkh | 3 | the |

往回挪三位的时候，就出现了 `the`，终于像个英文单词了。可能挪动位数就是 3，对整个密文都往回挪 3 位尝试一下：

> IlovethreethingsthesunthemoonandyouThesuni
sforthedaythemoonisforthenightandyouforever


加点空格和标点，

> I love three things:the sun ,the moon and you.
>
> The sun is for the day ,the moon is for the night
>
> and you forever.

恩，诗不错。。一顿男女混合双打是少不了了。

故事说完了，来总结一下知识。上面说的加密方式其实是密码学史上很著名的一种加密方式。这个加密方法是以罗马共和时期恺撒的名字命名的，叫做 **凯撒密码**，当年恺撒曾用此方法与其将军们进行联系。其挪动字母的位数我们称之为 **密钥空间** ，密钥空间越大，越难通过暴力破解。而凯撒密码的密钥空间是有限的，很容易遭到暴力破解。

## 简单替换密码

吃一堑，长一智。儿子的第二封情书又来了。

> cjqtqtxxwzqtwnfqtxlskwnfqtxcwqxzzxewfxgjsxwhwlsxqbumbgfu
bzekfzxwelbukbazjewmxqtwqygbllbelwzblxjnqtxfxxrlbuektxwzq

从密文中重复出现的单词 `qtx` 等来看，有点类似凯撒密码，但是作为程序员的儿子，肯定不会犯同样的错误了。基于凯撒密码的弱点，密钥空间不足，这次采取的加密方式肯定不能通过暴力破解来解密了。很容易想象到同属古典加密的简单替换密码，依次将明文中的每一个字母按照替换表换成另一个字母。乍看和凯撒密码一样，都是逐个字母替换成其他字母。但是由于简单替换密码并非按一定位数直接挪动，它的替换表中的字母是是随机对应的，仍以 26 个字母为例，`a` 可能对应 `x`，`b` 可能对应 `n`，总之，26 个字母乱序对应。那么，简单替换密码的密钥空间是多少呢？`a` 可以对应 26 个字母，`b` 可以对应 25 个字母，依次类推，密钥空间总数就是 `26 * 25 * 24 * 23 * ...`，大概是 `4*10^26`，所以通过暴力破解是不大可能解密的。

这里要介绍一种新的解密方式，叫 **频率分析**。在简单替换密码中，由于替换表在一次加密中是固定的，所以明文字母和密文字母也总是一一对应的，所以根据密文字母出现的频率，结合日常明文中字母出现的概率，总能看出一些端倪。下面先统计一下密文中字母出现的概率：

| 出现次数 | 密文字母 |
| :------: | :------: |
| 1 | a、h、r、y |
| 2 | c、m |
| 3 | g、n、s |
| 4 | j、k、u |
| 6 | e、f |
| 8 | l、t、z |
| 9 | b |
| 11 | q |
| 12 | w |
| 16 | x |

明文字母出现的概率其实是有一定规律的，`e` 是英语中最常用的字母，其出现频率为八分之一。最常用的 9 个字母 e，t，a，o，n，i，r，s 和 h 。全部英语单词中有一半以上是以 t，a，o，s 或 w 开头的。仅 10 个单词（the，of，and，to，a，in，that，it，is 和 I）就构成标准英语文章四分之一以上的篇幅。所以当密文长度越长，其实就越容易漏出破绽。当然上面的例子很短，只是为了方便说明频率分析法。

一般将密文中出现频率最高的字母替换为 `e` 总是没错的，这里先把 `x` 换成 `E` 。为了区分密文和明文，明文使用大写字母表示:


> cjqt**qtE**Ewzqtwnf**qtE**lskwnf**qtE**cwqEzzEewfEgjsEwhwlsEqbumbgfu
bzekfzEwelbukbazjewmEqtwqygbllbelwzblEjn**qtE**fEErlbuektEwzq

上面的密文已将所有 `x` 都替换成了 `E`。含有 `e` 的英文中最常见的单词就是 `the` 了，而且上面密文中也频繁出现了 `qtE`（粗体标注）, 不妨假设这就是 `the`，可以得到密文 `q` 对应明文 `t`，密文`t` 对应明文 `h`，继续替换密文中相应字母：

> cjTH**THE**EwzTHwnfTHElskwnf**THE**cwTEzzEewfEgjsEwhwlsETbumbgfu
bzekfzEwelbukbazjewmETHwTygbllbelwzblEjn**THE**fEErlbuekHEwzT

接下来就得靠经验（瞎蒙）了，第 5 到 第 12 个字母， `THEEwzTH` 后面的 `EwzTH` 应该是一个单词，想想什么单词是 `exxth`，很容易想到 `earth`，可以得到密文 `w` 对应明文 `a`，密文`z` 对应明文 `r`，继续替换：

> cjTH THE EARTH Anf THE lskAnf THE cATERREeAfEgjsEAhAlsETbumbgfu
bRekfREAelbukbaRjeAmETHATygbllbelARblEjn THE fEErlbuek HEART

`cjTH THE EARTH Anf THE lskAnf` ,`with the ... and  the ...`, 这里的`chTH` 应该是 `with`, `Anf` 应该是 `and`, 用`W` 、`I`、 `D` 分别替换`c`、`j`、 `f`:

> WITH THE EARTH AND THE lsk AND THE WATER REeADEgIsEAhAlsETbumbgDu
bRekDREAelbukbaRIeAmETHATygbllbelARblEIN THE DEErlbuek HEART

`REeADE` 大概也只能是 `remade` 了。再次替换;

> WITH THE EARTH AND THE lsk AND THE WATERREMADEgIsEAhAlsETbumbgDu
bRMkDREAMlbukbaRIMAmETHATygbllbMlARblEIN THE DEErlbuMk HEART

结尾 `Mk HEART` 很容易想到 `My heart`,用 `Y` 代替 `k` :

> WITH THE EARTH AND THE lsY AND THE WATER REMADE gIsEAhAlsET **bu** mbgDu
bR MY DREAMl **bu** YbaRIMAmETHATygbllbMlARblEIN THE DEErl **bu** MY HEART

一时没有头绪了，再来看看重复字符串，发现了一个 `bu`，上面已标注。还是看看结尾来猜测，`xx my heart`，很容易想到 `of my heart`，`o` 在明文中也是高频字母，这里的 `b` 在密文中出现了 9 次，也是高频字母，因此很有可能就是 `of`, 替换进去看一下：

> WITH THE EARTH AND THE lsY AND THE WATER REMADE gIsEAhAlsET OF mOgD FOR MY DREAMl OF **YOaR IMAmE** THAT ygOllOMlAROlEIN THE DEErl OF MY HEART

果然又可以认出几个单词，上面加粗标注的 `YOaR IMAmE` 应当是 `your image`，继续替换：

> WITH THE EARTH AND THE lsY AND THE WATER REMADE gIsEAhAlsET OF GOgD FOR MY DREAMl OF YOUR IMAGE THAT ygOllOMlAROlE IN THE DEErl OF MY HEART

`OF GOgD FOR MY DREAMl` ，`go什么d`，除了金子 `gold` 好像也没什么单词了。`dream` 后面加个字母，除了加复数 s 好像也没什么选择。再替换这两个字母：

> WITH THE EARTH AND THE SsY AND THE WATER REMADE gIsEAhASsET OF GOLD FOR MY DREAMS OF YOUR IMAGE THAT yLOSSOMS A ROSE IN THE DEErS OF MY HEART

`s什么y`，`sky`, `什么lossoms`，很适合查字典，`blossoms`。`K` 和 `B` 代替 `s` 和 `y` :

> WITH THE EARTH AND THE SKY AND THE WATER REMADE gIKE A hASKET OF GOLD FOR MY DREAMS OF YOUR IMAGE THAT BLOSSOMS A ROSE IN THE DEErS OF MY HEART

大致内容就全部出来了，手动渲染一下。

> With the earth and the sky and the water,
>
> remade, like a casket of gold.
>
> For my dreams of your image that blossoms
>
> a rose in the deeps of my heart.

看起来频率分析法很麻烦，但是对于专业破译者，其丰富的破解经验可以快速进行破译。从公元前开始，简单替换密码在几百年前的时间里一直被用于秘密通信。然而在阿拉伯学者发明频率分析法之后，这种密码就很容易破解了。

## 总结

凯撒密码是一种将明文字母按一定位数挪动形成密文的加密方法，因其密钥空间有限，可以暴力破解。

简单替换密码是一种将明文字母按替换表逐位替换形成密文的加密方法，其密钥空间极大，不可以暴力破解，但可以使用频率分析法进行破译。

凯撒密码实际上也是一种简单替换密码。

儿子再次败下阵来，今天是一顿女子单打，老父亲用脑过度。睡前感觉于心不忍，去看看儿子，见笔记本开着，上面写了四个大字，**对称加密** ! 睡梦中的儿子好似露出了狡黠的笑容。

```
预知后事如何，且听下回分解。（以上故事纯属虚构，我儿子 3 岁。）
```

> 文章首发微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解。
>
> 更多 JDK 源码解析，扫码关注我吧！

![](https://user-gold-cdn.xitu.io/2019/4/27/16a5f352eab602c4?w=2800&h=800&f=jpeg&s=178470)
