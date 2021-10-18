> 本文记录了我对`DICOM` 3.0标准中，关于`DICOM`文件格式的浅薄理解
>
> 参考文章
>
> http://dicom.nema.org/medical/dicom/current/output/pdf/part05.pdf
>
> http://dicom.nema.org/medical/dicom/current/output/pdf/part06.pdf
>
> http://dicom.nema.org/medical/dicom/current/output/pdf/part10.pdf
>
> https://blog.csdn.net/wenzhi20102321/article/details/75127101
>
> https://blog.csdn.net/wenzhi20102321/article/details/75127362
>
> https://blog.csdn.net/Marksinoberg/article/details/112108526

<img src="E:\ljj的博客\pictures source\image-20211013170901465.png" style="zoom: 50%;" />

## File Meta Info

> The File Meta Information includes identifying information on the encapsulated Data Set. This header consists of a 128 byte `FilePreamble`, followed by a 4 byte `DICOM` prefix, followed by the File Meta Elements shown in Table 7.1-1. This header shall be present in every `DICOM` file

文件信息头包含了一些特征信息，这些信息用于识别后续的存放像素数据的Data Set。

文件头由三个部分构成

- 128字节的前导序码，或者叫序言，一般固定不变的
- 4字节的标识符，标志这是个``DICOM``文件，如果校验失败，直接拒绝读取
- 不确定长度的File Meta Elements

### 序言

> The File Preamble is available for use as defined by Application Profiles or specific implementations. This Part of the `DICOM` Standard does not require any structure for this fixed size Preamble. It is not required to be structured as a `DICOM` Data Element with a Tag and a Length. It is intended to facilitate access to the images and other data in the `DICOM` file by providing compatibility with a number of commonly used computer image file formats. Whether or not the File Preamble contains information, the `DICOM` File content shall conform to the requirements of this Part and the Data Set shall conform to the SOP Class specified in the File Meta Information.
>
> If the File Preamble is not used by an Application Profile or a specific implementation, all 128 bytes shall be set to `00H`. This is intended to facilitate the recognition that the Preamble is used when all 128 bytes are not set as specified above.
>
> The File Preamble may for example contain information enabling a multi-media application to randomly access images stored in a `DICOM` Data Set. The same file can be accessed in two ways: by a multi-media application using the preamble and by a `DICOM` Application that ignores the preamble.

翻译成人话就是：

- 本标准没有对序言部分做任何限制，即便这128个字节里全是乱码，也不应该影响后续的Data Set。之所以预留这128个字节，是为了开发者新增功能时对其他图片格式的兼容，或者是一些应用的特定配置要求。

- 如果你没有用这128个字节做一些特别的功能，务必把它们全设成`00H`。这样便于程序识别
- 举个使用的例子：我希望用自己开发的一款多媒体显示器 `LJJ Demo`，去查看一个`dcm`文件，但我可能并没有严格实现`DICOM`协议的细节，但我又想强行打开。那么我可以在序言里加入一些字段声明该`dcm`文件允许 `LJJ Demo` 查看

### 标识符

> The four byte `DICOM` Prefix shall contain the character string "`DICM`" encoded as uppercase characters of the ISO 8859 `G0` Character Repertoire. This four byte prefix is not structured as a `DICOM` Data Element with a Tag and a Length

就是四个大写字母，`DCIM`

### File Meta Elements  文件可变元素集合

注意，前面的序言和标识符其实也是两个element，只不过比较特殊，没有定义到Tag列表中。所以单拿出来讲了

![image-20211013174016600](E:\ljj的博客\pictures source\image-20211013174016600.png)

![image-20211013174033373](E:\ljj的博客\pictures source\image-20211013174033373.png)

##### Tag

`DICOM`标准对信息有很多层分类，其中有两层是这样的，第一层是Group，第二层是Element

Tag = (Group Number, Element Number) 单位是16进制

标准把所有可能的element都列成表格，配上对应的Tag，这样开发者就不能自定义属性了，所有人必须用Tag访问规定的属性

**注意这里的Tag仅仅是个编号，并不是信息的类型，`VR`才是信息的类型，`DICOM`规定了27种`VR`**

![image-20211013175002842](E:\ljj的博客\pictures source\image-20211013175002842.png)

##### Type

> An attribute, encoded as a Data Element, may or may not be required in a Data Set, depending on that Attribute Data Element Type. The Data Element Type of an Attribute of an Information Object Definition or an Attribute of a SOP Class Definition is used to specify whether that Attribute is mandatory or optional. The Data Element Type also indicates if an Attribute is conditional (only mandatory under certain conditions).

说人话就是：所有的属性都应该根据重要程度，划分到不同的等级。Type就是等级标志

- Type 1：必须存在，且信息长度不能为0，也就是不可空
- Type `1C`：在特定条件下，要求和1相同。但如果不满足条件，则该数据在Data Set中不允许出现
- Type 2：必须存在，但其信息长度在某些情况下可以为0
- Type `2C`：在特定条件下，要求和2相同。但如果不满足条件，则该数据在Data Set中不允许出现
- Type 3：可选存在，长度也随意。为空时视为不存在即可

这里的各种"特定条件"，manual里写得也很含糊不清，只有自己遇到了再体会吧

##### Description

挑一个最重要的，其余的`RTFM`

Transfer Syntax `UID`  (0002,0010)

> Uniquely identifies the Transfer Syntax used to encode the following Data Set. This Transfer Syntax does not apply to the File Meta Information. Note It is recommended to use one of the `DICOM` Transfer Syntaxes supporting explicit Value Representation encoding to facilitate interpretation of File Meta Element Values. `JPIP` Referenced Pixel Data Transfer Syntaxes are not used

这个字段决定了普通Tag的读取方式：

- 小端序/大端序。就是高位在前还是低位在前
- `VR`的读取是显式的（也就是直接能从`DCM`文件中读到的），还是隐式的（根据Tag自己查表）

### `VR` &&  `VM`

> The Value Representation of a Data Element describes the data type and format of that Data Element's Value(s). 

简单的理解：**编程语言一般都有 int 类型，Float类型等等，`VR`就是类型的意思，只不过 `DICOM` 比较多，有27种类型**

完整的`VR`定义列表见 http://dicom.nema.org/medical/dicom/current/output/html/part05.html#sect_6.2

![](E:\ljj的博客\pictures source\image-20211018141608048.png)

`VR`受到传输句法 transfer syntax 的影响，分显式和隐式。一旦约定好了，整个文件的所有属性都是统一的形式

> The Value Multiplicity of a Data Element specifies the number of Values that can be encoded in the Value Field of that Data Element.

简单理解：`VM`规定了一个 element 可不可以是一个列表。是1时，表示这个element就是个单一的元素，为1-n时，表示这个element可以是一个列表，包含1-5个子元素。所有的File Meta Info的element都是`VM` = 1的

**Example:**

下面是从Data Set对应的Tag列表里，找的一个 Tag = (0010,2154) `VR` = SH, `VM` = 1-n 的属性。

很容易理解，病人的手机号 

![](E:\ljj的博客\pictures source\image-20211018143314249.png)

翻一下`VR`列表可以查到SH的含义

![](E:\ljj的博客\pictures source\image-20211018144401543.png)

这里可以看到它限制了最长16个字符，我们中文手机号是11位，所以理论上 Patient's Telephone Numbers 这个字段在中国地区只能写入一个手机号。所以理论上其实它的 `VM` = 1-n 在中国没用。。。（不过中国座机号是6位的，可以写两个）

## Data Set

> A Data Set is constructed by encoding the values of Attributes specified in the Information Object Definition (`IOD`) of a Real-World Object. A Data Set represents an instance of a real world Information Object. A Data Set is constructed of Data Elements. Data Elements contain the encoded Values of Attributes of that object. 
>
> data set是一个集合，集合里的元素是对现实物体的属性的编码值。一个data set就代表了现实的一个物体。

事实上前面那么多，和真正我们关心的病人数据没有任何关系......

### Data Element

data element 的组织形式和前面一样，也是键值对的样式，一个tag标记一个属性，后面跟着真正的值

![DICOM Data Set and Data Element Structures](http://dicom.nema.org/medical/dicom/current/output/html/figures/PS3.5_7.1-1.svg)

##### Tag

data set的tag种类非常多，列表见 https://exiftool.org/TagNames/DICOM.html

下面介绍一些重要的tag

- tag: `7FE0,0010 PixelData` 像素数据开始的标志

- Patient相关

  ![image-20211013183854420](E:\ljj的博客\pictures source\image-20211013183854420.png)

- Study相关

  ![image-20211013183856860](E:\ljj的博客\pictures source\image-20211013183856860.png)

- Series相关

  ![image-20211013184027844](E:\ljj的博客\pictures source\image-20211013184027844.png)

- Image相关

  ![image-20211013184045194](E:\ljj的博客\pictures source\image-20211013184045194.png)

- tag `FFFC,FFFC  DataSetTrailingPadding  DataSet`的结束标志，有`DICOM` File Service管理

##### element fields and structure

一个element的各个部分，各自的长度由下表规定，其中显式/隐式 `VR` 前文已提到

显式`VR`

`VR` 一共有27种，不同类别对应的结构不一样

![](E:\ljj的博客\pictures source\image-20211018135306530.png)

隐式`VR`

![](E:\ljj的博客\pictures source\image-20211018135350101.png)

##### Patient Name `VR=PN` `VM=1`

先贴一下 `PN` 的类型说明

![](E:\ljj的博客\pictures source\image-20211018150143082.png)

大致意译一下官方文档

> A character string encoded using a 5 component convention. The character code 5CH (the BACKSLASH "\" in ISO-IR 6) shall not be present, as it is used as the delimiter between values in multi-valued data elements. The string may be padded with trailing spaces. For human use, the five components in their order of occurrence are: family name complex, given name complex, middle name, name prefix, name suffix.
>
> 代表人名的字符串应该由5个组件构成，里面不能有转移字符。字符串末尾可能有空白字符的填充。表示人名时，五个组件依次是家族名字，个人名字，中间名，名前缀，名后缀（欧美的命名法则....）
>
> Any of the five components may be an empty string. The component delimiter shall be the caret "^" character (5EH). There shall be no more than four component delimiters, i.e., none after the last component if all components are present. Delimiters are required for interior null components. Trailing null components and their delimiters may be omitted. Multiple entries are permitted in each component and are encoded as natural text strings, in the format preferred by the named person.
>
> 任何组件都应该是可空的，组件间的分隔符是"^"，即便某个组件是空，你也应该加上分隔符。最后一个组件如果非空则末尾不用跟分隔符，所以分隔符最多不超过4个。
>
> For the purpose of writing names in ideographic characters and in phonetic characters, up to 3 groups of components (see [Annex H](http://dicom.nema.org/medical/dicom/current/output/html/part05.html#chapter_H), [Annex I](http://dicom.nema.org/medical/dicom/current/output/html/part05.html#chapter_I) and [Annex J](http://dicom.nema.org/medical/dicom/current/output/html/part05.html#chapter_J)) may be used. The delimiter for component groups shall be the equals character "=" (3DH). There shall be no more than two component group delimiters, i.e., none after the last component group if all component groups are present. The three component groups of components in their order of occurrence are: an alphabetic representation, an ideographic representation, and a phonetic representation.
>
> 对于用象形文字或者拼音文字的字符串，最多只需要用三个组件。组件的分隔符也改成"="。所以此时分隔符最多不超过2个。最后一个组件非空时末尾不用跟分隔符。三个组件依次是字母，象形，和音标。

- 每个组件不超过64个字符
- 组件内内容的编码应该使用 0008,0005 的属性规定的字符集

![](E:\ljj的博客\pictures source\image-20211018150800451.png)

## 总结

**`.dcm`文件的组成形式是键值对的形式，键是 Tag，值是 `VR` 规定的类型的对象**

但官方文档还有很多格式相关的规定，比如 element的嵌套，多个`dcm`文件的组织等等。