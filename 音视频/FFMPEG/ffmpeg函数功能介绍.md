1，注册所有组件

```text
av_register_all();
```

------

2，初始化网络

```text
avformat_network_init()
```

------

3，对AVFormatContext做初始化

```text
avformat_alloc_context()
```

附：使用示列

```text
pFormatCtx = avformat_alloc_context();
```

------

4，打开视频流

```text
avformat_open_input()
```

附：使用示列

```text
if(avformat_open_input(&pFormatCtx,filepath,NULL,NULL)!=0){
    printf("Couldn't open input stream.\n");
    return -1;
}
```

说明：

1. 第一个参数是二级指针
2. 返回值为0，打开成功；否则打开失败

------

5，获取流信息

```text
avformat_find_stream_info()
```

附：使用示列

```text
if(avformat_find_stream_info(pFormatCtx,NULL)<0){
	printf("Couldn't find stream information.\n");
	return -1;
}
```

------

6，找到解码器

```text
avcodec_find_decoder()
```

附：使用示列

```text
pCodec = avcodec_find_decoder(pCodecCtx->codec_id);
```

说明：

```text
AVCodec         *pCodec     = NULL;
pCodecCtx = pFormatCtx->streams[videoindex]->codec;
```

------

7，打开解码器

```text
avcodec_open2()
```

附：使用示列

```text
if(avcodec_open2(pCodecCtx, pCodec,NULL)<0){
	printf("Could not open codec.\n");
	return -1;
}
```

------

8，解码帧初始化

```text
av_frame_alloc()
```

附：使用示列

```text
AVFrame* pFrame
pFrame = av_frame_alloc();
```

------

9，内存分配

```text
av_malloc()
```

说明：以字节为单位分配内存，并做了一些错误检查工作。

附：使用示列

```text
av_malloc(1024);
```

------

10，得到图片对应大小

```text
avpicture_get_size()
```

附：使用示列

```text
int a = avpicture_get_size(PIX_FMT_YUV420P, pCodecCtx->width, pCodecCtx->height)
```

------

11，用out_buffer对应数据填充pFrameYUV的data

```text
avpicture_fill()
```

附：使用示列

```text
avpicture_fill((AVPicture *)pFrameYUV, out_buffer, PIX_FMT_YUV420P, pCodecCtx->width, pCodecCtx->height);
```

------

12，打印信息

```text
av_dump_format()
```

附：使用示列

```text
av_dump_format(pFormatCtx,0,filepath,0);
```

------

13，读取原数据

```text
av_read_frame()
```

附：使用示列

```text
while(av_read_frame(pFormatCtx, packet)>=0){
      代码块;
}
```

------

14，解码数据为一帧

```text
avcodec_decode_video2()
```

附：使用示列

```text
int got_picture;
ret = avcodec_decode_video2(pCodecCtx, pFrame, &got_picture, packet);
```

------

15，去除黑边框

```text
sws_scale()
```

附：使用示列

```text
struct SwsContext *img_convert_ctx;

img_convert_ctx = sws_getContext(pCodecCtx->width, pCodecCtx->height, pCodecCtx->pix_fmt, 
		pCodecCtx->width, pCodecCtx->height, PIX_FMT_YUV420P, SWS_BICUBIC, NULL, NULL, NULL);
 
sws_scale(img_convert_ctx, (const uint8_t * const*)pFrame->data, pFrame->linesize, 0, pCodecCtx->height,
         pFrameYUV->data, pFrameYUV->linesize);
```

------

16，释放数据

```text
av_free_packet(packet);
sws_freeContext(img_convert_ctx);
av_frame_free(&pFrameYUV);
av_frame_free(&pFrame);
```

------

17，关闭数据

```text
avcodec_close(pCodecCtx);
avformat_close_input(&pFormatCtx);
```