本文将介绍的如何用**FFMPEG API**做视频解码。

视频解码，是将压缩后的视频（压缩格式如H264）通过对应解码算法还原为YUV视频流的过程；在计算机看来，首先输入一段01串（压缩的视频），然后进行大量的浮点运算，最后再输出更长的一段01串（还原的非压缩视频）。计算机内部可以进行浮点数计算的部件是CPU，目前市场上涌现了一批GPU和类GPU芯片，如Nvidia、海思芯片甚至Intel自家的核显。利用前者进行解码一般称为“**软解码**”，后者被称为“**硬解码**”，如果没有特殊指定，FFMPEG是用CPU进行解码的，即**软解**。

本文将介绍的是**软解**，也就是FFMPEG最通用的做法。



本文将介绍以下内容：

1. FFMPEG解码的通用流程以及每个步骤涉及的API详解，这一部分比较硬核，会涉及代码和api底层的解释；
2. FFMPEG解码的其他注意点。



“自古深情留不住，唯有套路得人心”，和很多工具一样，FFMPEG解码也是有套路的，这里先搬出业内大人物雷神（雷霄骅，可以从知乎的帖子-[如何看待雷霄骅之死？](https://www.zhihu.com/question/49211380/answer/114884629)-知道雷神在流媒体技术上做出的贡献之大，叹息缅怀）当年的解码流程图：

![img](https://pic4.zhimg.com/80/v2-114bdb92f34e7f577921c4c93edc3a13_720w.jpg)FFMPEG解码流程（雷神版）

上图出自其CSDN博客：[最简单的基于FFmpeg的解码器-纯净版](https://link.zhihu.com/?target=https%3A//blog.csdn.net/leixiaohua1020/article/details/42181571)。15年的文章，15年，FFMPEG还只有2.x，因此这个解码流程图目前已经不适用了。笔者结合网上最新的资料和自己实际开发使用API的情况，整理了出以下解码流程图（虚线框是可选部分）：

![img](https://pic1.zhimg.com/80/v2-8192d34b8eb09b4c839df97b3bd1c060_720w.jpg)FFMPEG解码流程



接下来详细解释FFMPEG解码的通用流程。

### **解码Step1. 连接和打开视频流**

**连接和打开视频流**必然是后续进行解码的关键，该步骤对应的API调用为：

- `int avformat_network_init(void)`
  官方文档建议加上`avformat_network_init()`，虽然这个不是必须的。笔者推荐阅读[FFMpeg](https://link.zhihu.com/?target=https%3A//blog.csdn.net/y_z_hyangmo/article/details/77579392)[ 源码分析（2）avformat_network_init()](https://link.zhihu.com/?target=https%3A//blog.csdn.net/y_z_hyangmo/article/details/77579392)深入了解该函数的内部源码，说白了，该函数会初始化和启动底层的**[TLS库](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/%E5%82%B3%E8%BC%B8%E5%B1%A4%E5%AE%89%E5%85%A8%E6%80%A7%E5%8D%94%E5%AE%9A)**，这也就解释了网上很多资料关于**如果要打开网络流的话，这个API是必须的**的说法了。
- `int avformat_open_input(AVFormatContext** ps, const char* filename, AVInputFormat* fmt, AVDictionary ** options)`
  `avformat_open_input()`官方说法是“打开并读取视频头信息”，该函数较为复杂，笔者还没有完全吃透他的每一行源码，大致了解其功能为**AVFormatContext内存分配。如果是视频文件，会探测其封装格式并将视频源装入内部buffer中；如果是网络流视频，则会创建socket等工作连接视频获取其内容，装入内部buffer中。最后读取视频头信息**。源码深入阅读的话，笔者推荐雷神的[FFmpeg](https://link.zhihu.com/?target=https%3A//blog.csdn.net/leixiaohua1020/article/details/44064715)[源代码简单分析：avformat_open_input()](https://link.zhihu.com/?target=https%3A//blog.csdn.net/leixiaohua1020/article/details/44064715)，以及简书上的[Avformat_open_input函数的分析之--HTTP篇](https://link.zhihu.com/?target=https%3A//www.jianshu.com/p/43a81861653f)。

以上，就完成了文件流的初步初始化工作。



补充说明：这一个步骤结束后，就可以调用API`av_dump_format()`打印文件的基本信息了，如文件时长、比特率、fps、编码格式等，信息大概如下：

> Input #0, avi, from '${input_video_file_name}': Metadata: encoder : Lavf57.83.100 Duration: 00:10:00.00, start: 0.000000, bitrate: 4196 kb/s Stream #0:0: Video: h264 (High) (H264 / 0x34363248), yuvj420p(pc, bt709, progressive), 1920x1080, 4194 kb/s, 12 fps, 12 tbr, 12 tbn, 24 tbc



有读者可能会问：“网上看到的大部分博客，第一句调用的是`av_register_all()`，该函数的作用是注册所有的编解码器、复用/解复用组件等，你为什么不提呢？” 原因很简单，**函数`av_register_all()`在FFMPEG4.0及以上版本中被弃用了，见[av_register_all() has been deprecated in ffmpeg 4.0](https://link.zhihu.com/?target=https%3A//github.com/leandromoreira/ffmpeg-libav-tutorial/issues/29)。（这都9102年了，赶快使用4.0以上的FFMPEG吧！）**



### 解码Step**2. 定位视频流数据**

无论是离线的还是在线的视频文件，相对正确的称呼应该是“多媒体”文件。要知道，这些文件一般不止有一路视频流数据，可能同时包括多路音频数据、视频数据甚至字幕数据等。因此我们在做解码之前，需要首先找到我们需要的视频流数据。

- `int avformat_find_stream_info(AVFormatContext** ic, AVDictionary ** options)`
  `avformat_find_stream_info()`进一步解析该视频文件信息，主要是指[AVFormatContext](https://link.zhihu.com/?target=https%3A//www.ffmpeg.org/doxygen/4.0/structAVFormatContext.html)结构体的`AVStream`。从雷神的[FFmpeg](https://link.zhihu.com/?target=https%3A//blog.csdn.net/leixiaohua1020/article/details/44084321)[源代码简单分析：avformat_find_stream_info()](https://link.zhihu.com/?target=https%3A//blog.csdn.net/leixiaohua1020/article/details/44084321)文章可以了解到，**该函数内部已经做了一套完整的解码流程，获取了多媒体流的信息。**请注意，一个视频文件中可能会同时包括视频文件和音频文件等多个媒体流，这也就解释了为什么后续还要遍历`AVFormatContext`的`streams`成员（类型是`AVStream`）做对应的解码。

注意，笔者认为视频流的基本信息，如fps、码率、视频长度等信息是在第一步的**`avformat_open_input`打开视频头文件**步骤中获取到的。但是如果视频文件是h264/mpeg裸流数据，可能没有头信息，无法获取。笔者在这里记一笔，后面继续深入研究。



### 解码Step**3. 准备解码器codec**

**codec是FFMPEG的灵魂，顾名思义，解码必须由解码器完成。**准备解码器的步骤包括：寻找合适的解码器 -> 拷贝解码器（optiona）-> 打开解码器。

- **寻找合适的解码器 - `AVCodec\* avcodec_find_decoder(enum AVCodecID id)`**
  `avcodec_find_decoder`是从codec库内返回与`id`匹配到的解码器。另外还有一个与其对应的寻找解码器的API-`AVCodec* avcodec_find_decoder_by_name(const char* name)`，这个函数是从codec库内返回名字为`name`的解码器，一般在硬解码时，会通过应解码器名字指定应解码器（硬解码的流程会更复杂些，往往还需要打开相关硬件的底层库驱动等，本文不会涉及）。
  **问题来了，`id`要怎么获取呢？上一步找到的`AVStream`中的成员变量`codecpar->codec_id`就是了，`codecpar`类型为`AVCodecParameters`。**网上很多资料上是`codec->codec_id`，`codec`类型为`AVCodecContext`，在FFMPEG3.4及以上版本中已经被弃用了，官方推荐使用`codecpar`。两者的区别，读者自行到FFMPEG官方doc上了解。
- **拷贝解码器 - `AVCodecContext\* avcodec_alloc_context3(const AVCodec\* codec)`和`int avcodec_parameters_to_context(const AVCodec\* codec, const AVCodecParameters\* par)`**
  `avcodec_alloc_context3()`创建了`AVCodecContext`，而`avcodec_parameters_to_context()`才真正执行了内容拷贝。`avcodec_parameters_to_context()`是新的API，替换了旧版本的`avcodec_copy_context()`。
- **打开解码器 - [avcodec_open2(AVCodecContext\* avctx, const AVCodec\* codec, AVDictionary \** options)](https://link.zhihu.com/?target=https%3A//ffmpeg.org/doxygen/2.8/group__lavc__core.html%23ga11f785a188d7d9df71621001465b0f1d)**
  源码阅读还是推荐雷神的[FFmpeg](https://link.zhihu.com/?target=https%3A//blog.csdn.net/leixiaohua1020/article/details/44117891)[源代码简单分析：avcodec_open2()](https://link.zhihu.com/?target=https%3A//blog.csdn.net/leixiaohua1020/article/details/44117891)，该函数主要服务于解码器，包括为其分配相关变量内存、检查解码器状态等。

以上，1-3的步骤，笔者统称为“解码初始化阶段”。至此，如果每一步的API返回值都OK的话，可以开始真正的解码工作了！

### 解码Step**4. 解码**

解码的核心是**重复进行取包、拆包解帧的工作**，这里说的包是FFMPEG非常重要的数据结构之一：`AVPacket`，帧是其中同样重要的数据结构：`AVFrame`。

- **AVPacket**
  该数据结构的介绍和分析网上资料很多，推荐阅读[FFMPEG](https://link.zhihu.com/?target=https%3A//www.jianshu.com/p/bb6d3905907e)[结构体：AVPacket解析](https://link.zhihu.com/?target=https%3A//www.jianshu.com/p/bb6d3905907e)，简言之，该结构保存了解码，或者说解压缩之前的多媒体数据，包括流数据本身和附加信息。
  `AVPacket`是由函数[int av_read_frame(AVFormatContext* s, AVPacket* pkt)](https://link.zhihu.com/?target=https%3A//ffmpeg.org/doxygen/2.8/group__lavf__decoding.html%23ga4fdb3084415a82e3810de6ee60e46a61)获取得到的，该函数的具体实现在新版本中做了改良，确保每次取出的一定是完整的帧数据，推荐深入阅读雷神的[ffmpeg](https://link.zhihu.com/?target=https%3A//blog.csdn.net/leixiaohua1020/article/details/12678577)[ 源代码简单分析 ： av_read_frame()](https://link.zhihu.com/?target=https%3A//blog.csdn.net/leixiaohua1020/article/details/12678577)和[FFMPEG](https://link.zhihu.com/?target=http%3A//lazybing.github.io/blog/2016/12/15/av_read_frame/)[ 源码分析：av_read_frame](https://link.zhihu.com/?target=http%3A//lazybing.github.io/blog/2016/12/15/av_read_frame/)。
  修正：AVPacket和GOP没有任何关系，仅仅是FFMPEG用以存储一段解码/解压缩之前的数据结构而已。笔者一度怀疑`AVPacket`其实是个GOP，但是一直没有找到官方或者“民间”有类似的说话；而且在做视频解码时，发现一个`AVPacket`解码出来一般只有一个frame，不符合H264中对GOP的定义。笔者在这里mark下，后续深度研究下。
- **AVFrame**
  该数据结构的介绍和分析网上资料也不少，推荐阅读[FFMPEG](https://link.zhihu.com/?target=https%3A//www.jianshu.com/p/18fa498eb19e)[结构体分析：AVFrame](https://link.zhihu.com/?target=https%3A//www.jianshu.com/p/18fa498eb19e)，简言之，该结构保存了解码后，即解压缩后的帧本身的数据和附加信息。
  **`AVFrame`在新版本中由函数**[int avcodec_send_packet(AVCodecContext* avctx, AVPacket* pkt)](https://link.zhihu.com/?target=https%3A//ffmpeg.org/doxygen/3.4/group__lavc__decoding.html%23ga58bc4bf1e0ac59e27362597e467efff3)**和`int avcodec_receive_frame(AVCodecContext\* avctx, AVFrame\* frame)`产生，前者真正地执行了解码操作，后者则是从缓存或者解码器内存中取出解压出来地帧数据。**老版本中用的是`avcodec_decode_video2()`，目前已经被弃用。
  此外需要注意的是，一般而言，一次`avcodec_send_packet()`对应一次`avcodec_receive_frame()`，但是也会有一次对应多次的情况。这个关系参考一个`AVPacket`对应一个或多个`AVFrame`。

------

### **III. FFMPEG解码的其他注意点**

FFMPEG的处理套路直接简单，可是想要应用到实际中，仅仅了解套路，还不足够！那么，**还需要注意哪些才可以搭建一套可实际使用的解码呢？**

### **1. 帧转码**

**软解得到的帧格式是YUV格式的**，具体格式可以存放在`AVFrame`的`format（类型为int）`成员中，打印出数值后，再到`AVPixelFormat`中查找具体是哪个格式。一般而言，大多是实际使用场景中，最常用的是RGB格式，因此接下来就以RGB举例说明如何做帧转码。注意，其他格式的做法也是一样的。

核心是调用`int sws_scale(struct SwsContext* c, ...)`，该函数接受的参数有一大堆，具体参数和对应的含义建议查询官网，该函数主要做了**尺寸缩放（scale）和转码（transcode）**工作，源码阅读推荐雷神的[FFmpeg](https://link.zhihu.com/?target=https%3A//blog.csdn.net/leixiaohua1020/article/details/44346687)[源代码简单分析：libswscale的sws_scale()](https://link.zhihu.com/?target=https%3A//blog.csdn.net/leixiaohua1020/article/details/44346687)。

第一个参数**struct** [SwsContext](https://link.zhihu.com/?target=https%3A//ffmpeg.org/doxygen/3.0/structSwsContext.html)* c，需要调用`struct SwsContext* sws_getContext(..., enum AVPixelFormat dstFormat, ...)`创建，该函数也是一堆参数，请自行官网查询，其中参数enum [AVPixelFormat](https://link.zhihu.com/?target=https%3A//ffmpeg.org/doxygen/3.0/pixfmt_8h.html%23a9a8e335cf3be472042bc9f0cf80cd4c5) *dstFormat*，指定了目标格式，随后调用`sws_scale()`后得到的目标帧就是*dstFormat*格式的。

因此，如果你的目标格式是RGB，只需要指定*dstFormat*为需要的RGB类型即可，FFMPEG中的RGB系列的有`AV_PIX_FMT_RGB24`、`AV_PIX_FMT_ARGB`等。

### **2. 帧输出**

除了考虑输出帧的格式，另一个实际的问题是：**解出来的帧放在哪儿，怎么放？**

**放在哪儿的问题看个人需求**，有些可能直接dump到磁盘，保存成本地视频文件或者一帧一帧的图片；在有些应用场景，解码可能只是系统最前端模块，此时可能需要存放到共享内存或者系统内存。

随之而来的是**怎么放的问题**，前者如保存成视频，可以通过`fopen()`创建视频文件，接着再解码的循环内部调用`fwrite()`将帧数据保存到文件，最后用`fclose()`关闭即可；后者一定涉及到需要把`AVFrame`的帧数据转化成`uint8_t*`/`unsigned char*`的操作，可以调用API函数`int av_image_copy_to_buffer()`达到这个目的。

还有一个API函数`int av_image_fill_arrays()`，它是把`AVFrame`的`data`成员关联到某个地址空间，如果有了`av_image_copy_to_buffer`，那么该函数是否还必须，这个问题笔者目前还没有了解，这里记一笔吧。

### **3. 刷新缓冲区**

在实际做解码工作时一定要注意**刷新缓冲区**！！！如果不这么做的话，最后解码出来的帧数目和实际视频帧数是对不齐的，会发现总是少了一些尾帧。原因就是FFMPEG内部有一个buffer，需要再把buffer的帧刷出来。其实做法也很简单，在解码的最后，将packet的`data`和`size`成员分别赋值为`nullptr`和`0`，这个时候缓冲区所有的帧数据都会被放进一个packet中，因此最后再进行一次解码就可以拿出所有的帧数据了。

### **4. 资源释放**

FFMPEG非常重要的一点，有些申请的变量一定要在结束前显示释放。具体哪些API的调用需要显示释放，在官方文档上都有详细的说明。这里补充本样例代码的变量释放部分：

**了解了以上几点，整个解码流程是真正搭建起来了。最后提`AVDictionary`，一个名称为`可选项`，但是实际上非常有用的结构。**

### **4. options - `AVDictionary`**

在第二章的时候频繁出现了`AVDictionary** options`参数，尽管这个参数可以被置为`nullptr`，但实际上这个参数的用处还是挺大的，比如**设置FFMPEG缓存区大小、探测码流格式的时间、最大延时、超时时间、以及支持的协议的白名单等**。关于该方法的源码，笔者推荐阅读[FFmpeg接口-AVDictionary](https://link.zhihu.com/?target=https%3A//www.jianshu.com/p/89f2da631e16)。