## 正则表达式系列 （三）

前文链接：

本文介绍了与文本处理密切相关的Unicode概念，以及简单探索正则的匹配原理。

### Unicode

##### 什么是Unicode

在没有 Unicode 之前，全世界各个地区的文字编码各不相同，中国大陆用`GB2312`，美国用ASCII。这些编码标准的作用都是，给一个字符分配一个编号，形成码值与字符的映射。

但随着互联网发展，跨地区交流时，同样的码值对应着不同地区的字符，假设编号1对应中文的“我”，而编号1对应英文的“a”，这样不就无法解码了

所以Unicode标准出现，收录了全世界几乎所有的字符，统一分配编号

##### 组成结构

Unicode 分为 `UCS` 和 `UTF` 两部分

`UCS`就是个超大的容器，编号从0到1114111（一百多万个，到目前只分配了30万左右），收录全世界的字符。这个大容器又被分为17个小容器，称为平面，每一个平面大小是65536。平面编号从0到16，其中0号平面又被称为`BMP`（和`.bmp`文件没有任何关系）。日常使用的绝大部分字符都在这个平面里

`UTF`则是传输协议，熟悉`H.26x`这种视频编码协议的话，就很容易理解一组数据的存储和传输是两码事。存储可以不必太在意大小的问题，因为存储介质的成本已经下降很多。但传输一般都尽可能压缩大小，因为目前的通讯技术做不到忽略文件大小的程度。

所以Unicode设计了一系列传输协议，现在最常用的是 `UTF-8` 。它的基本思想是常用字符用1字节编码，不常用字符则逐渐用更多的字节，直到4个字节。规则如下

`U+ 0000 ~ U+ 007F: 0XXXXXXX`  //这个字符层又被称为ASCII字符，英文字母在这个层级
`U+ 0080 ~ U+ 07FF: 110XXXXX 10XXXXXX`
`U+ 0800 ~ U+ FFFF: 1110XXXX 10XXXXXX 10XXXXXX`  //中文字符一般在这个层级
`U+ 10000 ~ U+ 1FFFF: 11110XXX 10XXXXXX 10XXXXXX 10XXXXXX`

##### Unicode 属性

`UCS`的每个字符除了编号，还有一些属性，比如它的含义是表意符号还是标点，它来自哪个语系等等

所以Unicode定义三种属性来具体描述一个字符

- Unicode Property  字符的功能
- Unicode Script  字符的书写系统
- Unicode Block  字符所在的区间（人为划分，甚至有个区间叫作`Yijing_Hexagram_Symbols`收录了64个卦象字符）

有一些编程语言，支持根据这些属性匹配对应的字符集合，比如\p{P}可以匹配所有标点符号，包括中文的和英文的。但如果想使用，还是务必查阅对应语言的文档细则。

### 匹配原理

##### 有限状态机

这是计算理论中的一种模型，其有四个条件：

- 有限个状态
- 有一个状态转移函数
- 有一个开始状态
- 有一个结束状态

一个经典的例子是卖饮料的自动售货机。假设有一台售货机只接受5元的纸币，其中饮料价格都是整数，则根据余额的多少有6个状态：0，1，2，3，4，5。当输入一个5元纸币时，即为开始状态，当买完饮料并退还余款后，为结束状态。

<img src="C:\Users\19956875375\AppData\Roaming\Typora\typora-user-images\image-20211014131317169.png" alt="image-20211014131317169" style="zoom:25%;" />

##### `DFA 和 NFA`

正则的匹配过程就是一个有限状态机的状态转移过程

正则引擎实现了一套规则，能解析正则表达式并生成对应的状态机模型，这个过程在某些语言中被理解成正则表达式的编译过程

以正则表达式`a(bb)+a`为例，其有限状态机模型如下，Si表示状态，箭头表示状态转移，箭头上的字符表示状态转移条件。假设扫描到某个字符时正位于`S1`状态，此时扫描到的字符是`a`，那么状态转移失败，直接退出。

<img src="C:\Users\19956875375\AppData\Roaming\Typora\typora-user-images\image-20211014132653339.png" alt="image-20211014132653339" style="zoom: 50%;" />

**同一个正则表达式，其对应的有限状态机不为一**。下面这两个模型和上面的等价

<img src="C:\Users\19956875375\AppData\Roaming\Typora\typora-user-images\image-20211014133618483.png" alt="image-20211014133618483" style="zoom:50%;" />

<img src="C:\Users\19956875375\AppData\Roaming\Typora\typora-user-images\image-20211014133328458.png" alt="image-20211014133328458" style="zoom:50%;" />

仔细观察上面三个模型可以发现，对于一个固定的待匹配字符串，使用第一个模型时，扫描到任意一个位置，此时状态机的状态是确定的，也就是一定会走到这个状态。而对于后两个模型，其状态转移是存在分叉的，比如第三个模型的`S2`状态，接受一个b既可以转移到`S3`，也可以转到`S1`，所以对于一个固定的待匹配串，用后两种模型无法预测其转移过程

在计算理论中，**将第一种能确定状态的有限状态机称为`DFA`，后两种称为`NFA`**

不同的正则引擎构建状态机时会选择不同的类型，一般分类如下

<img src="C:\Users\199568~1\AppData\Local\Temp\WeChat Files\666653810ee6780acb479436b6d59d0.jpg" alt="666653810ee6780acb479436b6d59d0" style="zoom:35%;" />

##### 回溯

由上表看出，主流编程语言（除了Go）都选择`NFA`，为什么呢

因为正则中很多功能依赖状态转移的路径，所以匹配过程中需要记录所有可能的状态。而`NFA`的分支结构，就对应着各种可能的状态，而`DFA`则没有这个能力。（这也是为什么Go的正则不支持环视等一系列神奇的功能）

保存可能状态后，匹配过程就是个`DFS`搜索的过程，遇到匹配失败后，回退到上一步选择尝试其他可能，这个过程就叫作回溯

以一个简单例子说明。正则表达式`a.*b`匹配`a123b`的过程如下。

左侧是正则表达式，右侧是待匹配串。小人的位置代表当前匹配位置。因为 . * 的组合几乎能匹配所有字符，所以当左侧小人走到 `. * `后，这个状态下扫描到任何字符都不会转移，所以小人不动，直到待匹配串匹配完了，此时发现左侧小人还没走到末尾，说明匹配失败。

此时就要发生回溯，回退到上一步，尝试在此处停止`. *`的匹配。发现最后一个字符b和正则表达式能匹配上，所以左侧小人顺利转移状态到结束状态。右侧也匹配完成。

![image-20211014184459043](C:\Users\19956875375\AppData\Roaming\Typora\typora-user-images\image-20211014184459043.png)

这个例子也提示我们，尽量不要使用`. *`，因为匹配时几乎总是先把整个字符串全匹配完，然后再一步步回溯。这里比较幸运，只回溯一步就匹配成功了。如果待匹配串换成`ab123456789`那岂不要是要回溯9次！

