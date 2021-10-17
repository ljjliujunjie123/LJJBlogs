> 本文记录了我对`DICOM` 3.0标准中，关于`DICOM`文件格式的浅薄理解
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

<img src="C:\Users\19956875375\AppData\Roaming\Typora\typora-user-images\image-20211013170901465.png" alt="image-20211013170901465" style="zoom:50%;" />

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

注意，前面的序言和标识符其实也是两个element，只不过比较特殊，所以单拿出来讲了

![image-20211013174016600](C:\Users\19956875375\AppData\Roaming\Typora\typora-user-images\image-20211013174016600.png)

![image-20211013174033373](C:\Users\19956875375\AppData\Roaming\Typora\typora-user-images\image-20211013174033373.png)

##### Tag

`DICOM`标准对信息有很多层分类，其中有两层是这样的，第一层是Group，第二层是Element

Tag = (Group Number, Element Number) 单位是16进制

标准把所有可能的element都列成表格，配上对应的Tag，这样开发者就不能自定义属性了，所有人必须用Tag访问规定的属性

**注意这里的Tag仅仅是个编号，并不是信息的类型，`VR`才是信息的类型，`DICOM`规定了20多种`VR`**

![image-20211013175002842](C:\Users\19956875375\AppData\Roaming\Typora\typora-user-images\image-20211013175002842.png)

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

## Data Set

事实上前面那么多，和真正我们关心的病人数据没有任何关系......

data set的组织形式和前面一样，也是键值对的样式，一个tag标记一个属性，后面跟着真正的值

不过data set的tag种类就非常非常多，列表见https://exiftool.org/TagNames/DICOM.html

下面介绍一些重要的tag

- tag: `7FE0,0010 PixelData` 像素数据开始的标志

- Patient相关

  ![image-20211013183856860](C:\Users\19956875375\AppData\Roaming\Typora\typora-user-images\image-20211013183856860.png)

- Study相关

  ![image-20211013184027844](C:\Users\19956875375\AppData\Roaming\Typora\typora-user-images\image-20211013184027844.png)

- Series相关

  ![image-20211013184045194](C:\Users\19956875375\AppData\Roaming\Typora\typora-user-images\image-20211013184045194.png)

- Image相关

  ![image-20211013184143835](C:\Users\19956875375\AppData\Roaming\Typora\typora-user-images\image-20211013184143835.png)

- tag `FFFC,FFFC  DataSetTrailingPadding  DataSet`的结束标志，有`DICOM` File Service管理

