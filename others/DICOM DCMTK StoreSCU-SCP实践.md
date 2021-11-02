> 本文记录了利用DCMTK实践DICOM的StoreScu StoreScp的作业
>
> 主要目的是修改StoreScp，实现DICOM的图像几何变换

## DCMTK的编译

请参考  https://www.jianshu.com/p/b06349d609ba

如果遇到问题，请参考 https://sfumecjf.github.io/awesome_dcmtk/1-hello_world/

## DCMTK的工程结构

按照上述指引操作完成后，应该有以下目录

- 源代码：这个是从官网直接下的原始代码

<img src="E:\ljj的博客\pictures source\星愿浏览器截图20211102144648@2x.png" style="zoom:33%;" />

- VS 工程目录：这个是通过CMAKE配置好依赖库后，建立的VS工程，如果需要修改代码，可以在这里。这里的代码是源代码的拷贝

<img src="E:\ljj的博客\pictures source\星愿浏览器截图20211102144803@2x.png" style="zoom:33%;" />

- CMAKE 编译结果：这是用CMAKE工具编译源代码后，生成的结果，其中子目录bin下包含一系列exe是可用的

<img src="E:\ljj的博客\pictures source\星愿浏览器截图20211102144925@2x.png" style="zoom:33%;" />

如果你想通过修改DCMTK的代码实现新功能，有两种方法：

- 修改源代码，然后用CMAKE重新编译，生成结果如上述例子中的 `dcmtk_lib`
- 在VS工程下修改，利用VS编译，生成结果在 `VS工程目录\bin\Debug`

## STORE命令

store命令涉及的接口程序是`storescu.exe 和 storescp.exe`，它们分别对应的源代码文件是`storescu.cc和storescp.cc`

通过VS打开`storescp.cc`，简单分析其中的函数可以发现除了main函数作为入口，就只有下面两个函数比较关键

```c++
//真正执行Store命令的入口
static OFCondition storeSCP(
  T_ASC_Association *assoc,
  T_DIMSE_Message *msg,
  T_ASC_PresentationContextID presID)
    
//在执行Store的各个环节执行回调，指示执行的进度
static void storeSCPCallback(
    void *callbackData,
    T_DIMSE_StoreProgress *progress,
    T_DIMSE_C_StoreRQ *req,
    char * /*imageFileName*/, DcmDataset **imageDataSet,
    T_DIMSE_C_StoreRSP *rsp,
    DcmDataset **statusDetail)
```

由于我们的目的是对像素矩阵进行几何变换，所以必须要先等待图像真正被接收成功后。阅读上述两个函数的注释可以发现，Callback函数在图像接收结束后回调，把图像存到一个文件中，所以只需要在存储之前更改像素矩阵即可

```c++
//storeSCPCallback函数代码
//一共四个if代码块，前三个都是其他时间点的回调
//第四个是图像传输完成后的回调
if (progress->state == DIMSE_StoreEnd) {
	...
    //不难发现只有这一处在保存文件
    OFCondition cond = cbdata->dcmff->saveFile(
        fileName.c_str(), xfer, opt_sequenceType, 	opt_groupLength,
        opt_paddingType, OFstatic_cast(Uint32, opt_filepad), OFstatic_cast(Uint32, opt_itempad),
        (opt_useMetaheader) ? EWM_fileformat : EWM_dataset);  
	...
}
```

所以只需要在上述关键代码之前插入几何变换代码即可

```c++
// store file either with meta header or as pure dataset
OFLOG_INFO(storescpLogger, "storing DICOM file: " << fileName);
if (OFStandard::fileExists(fileName))
{
    OFLOG_WARN(storescpLogger, "DICOM file already exists, overwriting: " << fileName);
}
      //修改
      OFLOG_DEBUG(storescpLogger, "开始翻转");
      unsigned short *trulyPixData = NULL; //像素数组的指针
      unsigned short rows; //行数
      unsigned short cols; //列数
      DcmElement* myPixelData = NULL; //像素数据对应的element
      DcmElement* rows_elem = NULL; //行数对应的element
      DcmElement* cols_elem = NULL; //列数对应的element
	  //imageDataSet是DcmDataset类型的指针，指向一个Dicom文件的Data Set区域
	  //Data Set是一个类似键值对的Map形式，DCM_PixelData是对应的键
      (*imageDataSet)->findAndGetElement(DCM_PixelData, myPixelData); 
      (*imageDataSet)->findAndGetElement(DCM_Rows, rows_elem);
      (*imageDataSet)->findAndGetElement(DCM_Columns, cols_elem);
	 
      OFLOG_DEBUG(storescpLogger,"raw data length:"<<myPixelData->getLength());
	  //从对应的element中取出真正的数据
      myPixelData->getUint16Array(trulyPixData);
      rows_elem->getUint16(rows);
      cols_elem->getUint16(cols);
      
      OFLOG_DEBUG(storescpLogger, "行数："<< rows);
      OFLOG_DEBUG(storescpLogger, "列数："<< cols);

      //上下翻转
      //因为DICOM的图像是从左到右，从上到下的组织形式
      //所以只需要头尾两个指针，依次进行每行的交换即可实现上下翻转
      unsigned short* prev = trulyPixData;
      unsigned short* next = trulyPixData + (rows - 1) * cols;
      for (int row = 0; row < rows / 2; row++) {
          for (int col = 0; col < cols; col++) {
              unsigned short tmp = *prev;
              *prev = *next;
              *next = tmp;

              prev++;
              next++;
          }
          next = next - cols * 2;
      }
      OFLOG_DEBUG(storescpLogger, "翻转结束");
      //修改

OFCondition cond = cbdata->dcmff->saveFile(
    fileName.c_str(), xfer, opt_sequenceType, opt_groupLength,
    opt_paddingType, OFstatic_cast(Uint32, opt_filepad), OFstatic_cast(Uint32, opt_itempad),     
    (opt_useMetaheader) ? EWM_fileformat : EWM_dataset);
```

> 上述代码在我的测试数据上能正常运行，但不一定能在你的数据上work，因为DICOM每个像素的编码长度不一定是16bit的，所以上述代码的trulyPixData的类型需要按需调整
>
> 上述的实现参考了 https://www.cxyzjd.com/article/WHU_Kevin_Lin/77114298