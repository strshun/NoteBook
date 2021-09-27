## 保存webrtc::VideoFrame到YUV文件

~~~c++
void  SaveVideoFrameToFile(const webrtc::VideoFrame& frame, std::string file)	
{	
    rtc::scoped_refptr<webrtc::VideoFrameBuffer> vfb = frame.video_frame_buffer();
    static FILE *fp = fopen(file, "wb+");
    if (fp != NULL)	{
        fwrite(vfb.get()->GetI420()->DataY(), 1, frame.height() * frame.width(), fp);
        fwrite(vfb.get()->GetI420()->DataU(), 1, frame.height() * frame.width() / 4, fp); 
        fwrite(vfb.get()->GetI420()->DataV(), 1, frame.height() * frame.width() / 4, fp);
        fflush(fp);
    }
}
~~~

反复调用该函数可以将一系列的帧存入到文件中，然后使用YUVViewer等工具打开视频文件查看。