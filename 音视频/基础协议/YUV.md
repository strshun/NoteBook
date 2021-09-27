## YUV 详解（YUV444, YUV422, YUV420, YV12, NV12, NV21）
### 1. YUV 格式分为两大类
* planar

  对于planar的YUV格式，先连续存储所有像素点的Y，紧接着存储所有像素点的U，随后是所有像素点的V。

* packed

  对于packed的YUV格式，每个像素点的Y,U,V是连续交*存储的。

YUV，分为三个分量，“Y”表示明亮度（Luminance或Luma），也就是灰度值；而“U”和“V” 表示的则是色度（Chrominance或Chroma），作用是描述影像色彩及饱和度，用于指定像素的颜色。

用三个图来直观地表示采集的方式吧，以黑点表示采样该像素点的Y分量，以空心圆圈表示采用该像素点的UV分量。

 ![img](https://strsh-image-md.oss-cn-shanghai.aliyuncs.com/image/08141454_ACE9.jpg)

先记住下面这段话，以后提取每个像素的YUV分量会用到。

1. YUV 4:4:4采样，每一个Y对应一组UV分量。   frameSize = frameWidth * frameHeight * 3      
2. YUV 4:2:2采样，每两个Y共用一组UV分量。   frameSize = frameWidth * frameHeight * 2
3. YUV 4:2:0采样，每四个Y共用一组UV分量。   frameSize = frameWidth * frameHeight * 1.5

### 2, 存储方式

 下面我用图的形式给出常见的YUV码流的存储方式，并在存储方式后面附有取样每个像素点的YUV数据的方法，其中，Cb、Cr的含义等同于U、V。

**（1） YUVY 格式 （属于YUV422）**

 ![img](https://strsh-image-md.oss-cn-shanghai.aliyuncs.com/image/08141455_yH58.png)

YUYV为YUV422采样的存储格式中的一种，相邻的两个Y共用其相邻的两个Cb、Cr，分析，对于像素点Y'00、Y'01 而言，其Cb、Cr的值均为 Cb00、Cr00，其他的像素点的YUV取值依次类推。 

**（2） UYVY 格式 （属于YUV422）**

 ![img](https://strsh-image-md.oss-cn-shanghai.aliyuncs.com/image/08141455_BHdN.png)

UYVY格式也是YUV422采样的存储格式中的一种，只不过与YUYV不同的是UV的排列顺序不一样而已，还原其每个像素点的YUV值的方法与上面一样。

 

**（3） YUV422P（属于YUV422）**

YUV422P也属于YUV422的一种，它是一种Plane模式，即平面模式，并不是将YUV数据交错存储，而是先存放所有的Y分量，然后存储所有的U（Cb）分量，最后存储所有的V（Cr）分量，如上图所示。其每一个像素点的YUV值提取方法也是遵循YUV422格式的最基本提取方法，即两个Y共用一个UV。比如，对于像素点Y'00、Y'01 而言，其Cb、Cr的值均为 Cb00、Cr00。

 ![img](https://strsh-image-md.oss-cn-shanghai.aliyuncs.com/image/141625_Nmib_1024767.png)

**（4）YV12，YU12格式（属于YUV420）**

YU12和YV12属于YUV420格式，也是一种Plane模式，将Y、U、V分量分别打包，依次存储。其每一个像素点的YUV数据提取遵循YUV420格式的提取方式，即4个Y分量共用一组UV。注意，上图中，Y'00、Y'01、Y'10、Y'11共用Cr00、Cb00，其他依次类推。

 ![img](https://strsh-image-md.oss-cn-shanghai.aliyuncs.com/image/141726_CZLj_1024767.png)

**（5）NV12、NV21（属于YUV420）**

NV12和NV21属于YUV420格式，是一种two-plane模式，即Y和UV分为两个Plane，但是UV（CbCr）为交错存储，而不是分为三个plane。其提取方式与上一种类似，即Y'00、Y'01、Y'10、Y'11共用Cr00、Cb00

 ![img](http://static.oschina.net/uploads/space/2014/1208/141744_Wsvd_1024767.png)



YUV420 planar数据， 以720×488大小图象YUV420 planar为例，

其存储格式是： 共大小为(720×480×3>>1)字节，

分为三个部分:Y,U和V

Y分量：  (720×480)个字节 

U(Cb)分量：(720×480>>2)个字节

V(Cr)分量：(720×480>>2)个字节

三个部分内部均是行优先存储，三个部分之间是Y,U,V 顺序存储。

即YUV数据的0－－720×480字节是Y分量值，     

720×480－－720×480×5/4字节是U分量   

720×480×5/4 －－720×480×3/2字节是V分量。

### 格式相互转换
https://blog.csdn.net/leixiaohua1020/article/details/50534150