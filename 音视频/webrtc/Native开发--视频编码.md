## 1. 实时音视频开发主要步骤

![image](https://github.com/243286065/pictures_markdown/blob/master/webrtc/47d54ff6638beef0c09cd7d85b85441f.png?raw=true)

## 2. 概述

前面我们通过两篇文章分别介绍了视频采集的两种方式：采集摄像头和采集屏幕。获取数据之后，如果是要本地显示，那么就像我们之前做的那样，直接渲染出来就行；方式如果是进行存储或者进行传输，往往需要对数据进行编码压缩。

webrtc中的视频编解码部分的主要实现是位于`modules/video_coding`. 我们初步能看到的编码器的接口是位于`api/video_codecs/video_encoder.h`,这里定义了`webrtc::VideoEncoder`，不过这是个虚类，只是用于进行接口定义和部分接口的实现。

## 3. 创建编码器

![image](https://github.com/243286065/pictures_markdown/blob/master/webrtc/9ac3d9e47ff6baff4461882f821b9d4d.png?raw=true)

在`media/engine/internal_encoder_factory.h`中定义了创建编码器的通用接口：

```C++
// media/engine/internal_encoder_factory.h

namespace webrtc {

class RTC_EXPORT InternalEncoderFactory : public VideoEncoderFactory {
 public:
  static std::vector<SdpVideoFormat> SupportedFormats();
  std::vector<SdpVideoFormat> GetSupportedFormats() const override;

  CodecInfo QueryVideoEncoder(const SdpVideoFormat& format) const override;

  std::unique_ptr<VideoEncoder> CreateVideoEncoder(
      const SdpVideoFormat& format) override;
};

}  // namespace webrtc
```

- 可以使用`InternalEncoderFactory::SupportedFormats`接口得到所支持的编码器类型；

- 可以使用

  ```
  InternalEncoderFactory::QueryVideoEncoder
  ```

  接口查询编码器的状态

  ```C++
  // media/engine/internal_encoder_factory.cc
  VideoEncoderFactory::CodecInfo InternalEncoderFactory::QueryVideoEncoder(
  const SdpVideoFormat& format) const {
    CodecInfo info;
    info.is_hardware_accelerated = false;
    info.has_internal_source = false;
    return info;
  }
  ```

- 可以使用

  ```
  InternalEncoderFactory::CreateVideoEncoder
  ```

  接口来创建编码器

  ```
  webrtc::VideoEncoder
  ```

  实例

  ```C++
  // media/engine/internal_encoder_factory.cc
  
  std::unique_ptr<VideoEncoder> InternalEncoderFactory::CreateVideoEncoder(
      const SdpVideoFormat& format) {
    if (absl::EqualsIgnoreCase(format.name, cricket::kVp8CodecName))
      return VP8Encoder::Create();
    if (absl::EqualsIgnoreCase(format.name, cricket::kVp9CodecName))
      return VP9Encoder::Create(cricket::VideoCodec(format));
    if (absl::EqualsIgnoreCase(format.name, cricket::kH264CodecName))
      return H264Encoder::Create(cricket::VideoCodec(format));
    if (kIsLibaomAv1EncoderSupported &&
        absl::EqualsIgnoreCase(format.name, cricket::kAv1CodecName))
      return CreateLibaomAv1Encoder();
    RTC_LOG(LS_ERROR) << "Trying to created encoder of unsupported format "
                      << format.name;
    return nullptr;
  }
  ```

从`InternalEncoderFactory::CreateVideoEncoder`的实现里我们可以看到，webrtc支持四种类型的视频编码器：

- VP8Encoder
- VP9Encoder
- H264Encoder
- LibaomAv1Encoder

> ```
> H264`默认是不支持的，需要在编译的时候加上特殊的编译参数`rtc_use_h264=true
> ```

## 4. 编码器的接口

视频编码器的定义主要就是`webrtc::VideoEncoder`结构体，位于`api/video_codecs/video_encoder.h`:
![image](https://github.com/243286065/pictures_markdown/blob/master/webrtc/5a9ab7699143e8f4d01c191136a062fd.png?raw=true)

- 可以通过

  ```
  VideoEncoder::RegisterEncodeCompleteCallback
  ```

  注册一个

  ```
  EncodedImageCallback
  ```

  对象来接收编码后的数据

  ```C++
  // api/video_codecs/video_encoder.h
  
  class EncodedImageCallback {
   public:
    virtual ~EncodedImageCallback() {}
  
    struct Result {
      enum Error {
        OK,
  
        // Failed to send the packet.
        ERROR_SEND_FAILED,
      };
  
      explicit Result(Error error) : error(error) {}
      Result(Error error, uint32_t frame_id) : error(error), frame_id(frame_id) {}
  
      Error error;
  
      // Frame ID assigned to the frame. The frame ID should be the same as the ID
      // seen by the receiver for this frame. RTP timestamp of the frame is used
      // as frame ID when RTP is used to send video. Must be used only when
      // error=OK.
      uint32_t frame_id = 0;
  
      // Tells the encoder that the next frame is should be dropped.
      bool drop_next_frame = false;
    };
  
    // Used to signal the encoder about reason a frame is dropped.
    // kDroppedByMediaOptimizations - dropped by MediaOptimizations (for rate
    // limiting purposes).
    // kDroppedByEncoder - dropped by encoder's internal rate limiter.
    enum class DropReason : uint8_t {
      kDroppedByMediaOptimizations,
      kDroppedByEncoder
    };
  
    // Callback function which is called when an image has been encoded.
    virtual Result OnEncodedImage(
        const EncodedImage& encoded_image,
        const CodecSpecificInfo* codec_specific_info,
        const RTPFragmentationHeader* fragmentation) = 0;
  
    virtual void OnDroppedFrame(DropReason reason) {}
  };
  ```

- 使用`VideoEncoder::Encode`接口来编码一个`VideoFrame`;

- 还有其他接口可以传递参数给编码器。

## 5. 实战

从上面的分析看，我们在开发中如果要自己创建视频编码器的话，最简单直接的办法是直接使用`InternalEncoderFactory::CreateVideoEncoder`接口来创建视频编码器。虽然`InternalEncoderFactory`的命名看起来是不对外开放的接口，但是其实在官方的`examples/unityplugin/simple_peer_connection.cc`下就有直接使用这个接口：

```C++
bool SimplePeerConnection::InitializePeerConnection(const char** turn_urls,
                                                    const int no_of_urls,
                                                    const char* username,
                                                    const char* credential,
                                                    bool is_receiver) {
bool SimplePeerConnection::InitializePeerConnection(const char** turn_urls,
                                                    const int no_of_urls,
                                                    const char* username,
                                                    const char* credential,
                                                    bool is_receiver) {
  RTC_DCHECK(peer_connection_.get() == nullptr);

  if (g_peer_connection_factory == nullptr) {
    g_worker_thread = rtc::Thread::Create();
    g_worker_thread->Start();
    g_signaling_thread = rtc::Thread::Create();
    g_signaling_thread->Start();

    g_peer_connection_factory = webrtc::CreatePeerConnectionFactory(
        g_worker_thread.get(), g_worker_thread.get(), g_signaling_thread.get(),
        nullptr, webrtc::CreateBuiltinAudioEncoderFactory(),
        webrtc::CreateBuiltinAudioDecoderFactory(),
        std::unique_ptr<webrtc::VideoEncoderFactory>(
            new webrtc::MultiplexEncoderFactory(
                std::make_unique<webrtc::InternalEncoderFactory>())),
        std::unique_ptr<webrtc::VideoDecoderFactory>(
            new webrtc::MultiplexDecoderFactory(
                std::make_unique<webrtc::InternalDecoderFactory>())),
        nullptr, nullptr);
  }
  if (!g_peer_connection_factory.get()) {
    DeletePeerConnection();
    return false;
  }

  g_peer_count++;
  if (!CreatePeerConnection(turn_urls, no_of_urls, username, credential)) {
    DeletePeerConnection();
    return false;
  }

  mandatory_receive_ = is_receiver;
  return peer_connection_.get() != nullptr;
}  
```

因此我们自己在开发时，也可以直接使用.

这里我们在之前采集屏幕的例子上继续修改，添加一个`video_encode_handler.h`类用于进行编码：

```C++
// video_encode_handler.h

#ifndef EXAMPLES_DESKTOP_CAPTURE_VIDEO_ENCODE_HANDLER_H_
#define EXAMPLES_DESKTOP_CAPTURE_VIDEO_ENCODE_HANDLER_H_

#include "api/video/video_frame.h"
#include "api/video/video_sink_interface.h"
#include "api/video_codecs/video_encoder.h"
#include "api/video/encoded_image.h"
#include "modules/video_coding/include/video_codec_interface.h"
#include "modules/include/module_common_types.h"

namespace webrtc_demo {

class VideoEncodeHandler : public rtc::VideoSinkInterface<webrtc::VideoFrame>,
                           public webrtc::EncodedImageCallback {
 public:
  enum class VideoEncodeType {
    VP8,
    VP9,
    H264,
    UNSUPPORT_TYPE,
  };

  ~VideoEncodeHandler();

  static std::unique_ptr<VideoEncodeHandler> Create(const VideoEncodeType type);

 private:
  VideoEncodeHandler(const VideoEncodeType type);

  // rtc::VideoSinkInterface override
  void OnFrame(const webrtc::VideoFrame& frame) override;

  // webrtc::EncodedImageCallback override
  webrtc::EncodedImageCallback::Result OnEncodedImage(
      const webrtc::EncodedImage& encoded_image,
      const webrtc::CodecSpecificInfo* codec_specific_info,
      const webrtc::RTPFragmentationHeader* fragmentation) override;
  void OnDroppedFrame(webrtc::EncodedImageCallback::DropReason reason) override;

  webrtc::VideoCodec DefaultCodecSettings(size_t width, size_t height, size_t keyFrameInterval);

  void ReInitEncoder();

  std::unique_ptr<webrtc::VideoEncoder> video_encoder_;
  std::string encode_type_name_;
  size_t frame_width_ = 0;
  size_t frame_height_ = 0;
};

}  // namespace webrtc_demo

#endif  // EXAMPLES_DESKTOP_CAPTURE_VIDEO_ENCODE_HANDLER_H_
```

首先，作为一个`Sink`可以注册给抓屏对象，使得能够获取到抓屏得到的frame；然后将自己作为`EncodedImageCallback`实例注册给`VideoEncoder`:

```C++
// video_encode_handler.cc

#include "examples/desktop_capture/video_encode_handler.h"

#include "api/video/video_codec_type.h"
#include "api/video_codecs/sdp_video_format.h"
#include "api/video_codecs/video_codec.h"
#include "media/engine/internal_encoder_factory.h"
#include "rtc_base/logging.h"
#include "test/video_codec_settings.h"

namespace webrtc_demo {

constexpr size_t kWidth = 1920;
constexpr size_t kHeight = 1080;
constexpr size_t kBaseKeyFrameInterval = 30;

const webrtc::VideoEncoder::Capabilities kCapabilities(false);
const webrtc::VideoEncoder::Settings kSettings(kCapabilities,
                                               /*number_of_cores=*/1,
                                               /*max_payload_size=*/0);

VideoEncodeHandler::VideoEncodeHandler(const VideoEncodeType type)
    : video_encoder_(nullptr), frame_width_(kWidth), frame_height_(kHeight) {
  switch (type) {
    case VideoEncodeType::VP8:
      encode_type_name_ = "VP8";
      break;
    case VideoEncodeType::VP9:
      encode_type_name_ = "VP9";
      break;
    case VideoEncodeType::H264:
      encode_type_name_ = "H264";
      break;
    default:
      break;
  }

  auto support_formats = webrtc::InternalEncoderFactory::SupportedFormats();
  for (auto& format : support_formats) {
    RTC_LOG(LS_INFO) << "Support encode: " << format.ToString();
    if (!video_encoder_ && format.name == encode_type_name_) {
      RTC_LOG(LS_INFO) << "Find encode: " << format.name;
      std::unique_ptr<webrtc::InternalEncoderFactory> encode_factory =
          std::make_unique<webrtc::InternalEncoderFactory>();
      video_encoder_ = encode_factory->CreateVideoEncoder(format);
      video_encoder_->RegisterEncodeCompleteCallback(this);

      ReInitEncoder();
    }
  }
}

void VideoEncodeHandler::ReInitEncoder() {
  webrtc::VideoCodec codec_settings =
      DefaultCodecSettings(frame_width_, frame_height_, kBaseKeyFrameInterval);
  video_encoder_->InitEncode(&codec_settings, kSettings);
}

VideoEncodeHandler::~VideoEncodeHandler() {}

std::unique_ptr<VideoEncodeHandler> VideoEncodeHandler::Create(
    const VideoEncodeType type) {
  if (type >= VideoEncodeType::UNSUPPORT_TYPE) {
    RTC_LOG(LS_WARNING) << "Not support encode type: " << type;
    return nullptr;
  }

  return std::unique_ptr<VideoEncodeHandler>(new VideoEncodeHandler(type));
}

void VideoEncodeHandler::OnFrame(const webrtc::VideoFrame& frame) {
  // RTC_LOG(LS_INFO) << "-----VideoEncodeHandler::OnFrame-----";
  if (!video_encoder_) {
    RTC_LOG(LS_ERROR) << "Encoder not valid";
    return;
  }

  if (frame.width() != (int)frame_width_ || frame.height() != (int)frame_height_) {
    frame_width_ = frame.width();
    frame_height_ = frame.height();
    ReInitEncoder();
  }

  if (video_encoder_) {
    video_encoder_->Encode(frame, nullptr);
  }
}

webrtc::EncodedImageCallback::Result VideoEncodeHandler::OnEncodedImage(
    const webrtc::EncodedImage& encoded_image,
    const webrtc::CodecSpecificInfo* codec_specific_info,
    const webrtc::RTPFragmentationHeader* fragmentation) {
  RTC_LOG(LS_INFO) << "-----VideoEncodeHandler::OnEncodedImage-----" << encoded_image.size() << ", " << encoded_image.Timestamp() << "--" << encoded_image._completeFrame;

  return webrtc::EncodedImageCallback::Result(
      webrtc::EncodedImageCallback::Result::OK);
}

void VideoEncodeHandler::OnDroppedFrame(
    webrtc::EncodedImageCallback::DropReason reason) {
  RTC_LOG(LS_INFO) << "-----VideoEncodeHandler::OnDroppedFrame-----";
}

webrtc::VideoCodec VideoEncodeHandler::DefaultCodecSettings(
    size_t width,
    size_t height,
    size_t key_frame_interval) {
  webrtc::VideoCodec codec_settings;
  webrtc::VideoCodecType codec_type =
      webrtc::PayloadStringToCodecType(encode_type_name_);
  webrtc::test::CodecSettings(codec_type, &codec_settings);

  codec_settings.width = static_cast<uint16_t>(width);
  codec_settings.height = static_cast<uint16_t>(height);

  switch (codec_settings.codecType) {
    case webrtc::kVideoCodecVP8:
      codec_settings.VP8()->keyFrameInterval = key_frame_interval;
      codec_settings.VP8()->frameDroppingOn = true;
      codec_settings.VP8()->numberOfTemporalLayers = 1;
      break;
    case webrtc::kVideoCodecVP9:
      codec_settings.VP9()->keyFrameInterval = key_frame_interval;
      codec_settings.VP9()->frameDroppingOn = true;
      codec_settings.VP9()->numberOfTemporalLayers = 1;
      break;
    case webrtc::kVideoCodecAV1:
      codec_settings.qpMax = 63;
      break;
    case webrtc::kVideoCodecH264:
      codec_settings.H264()->keyFrameInterval = key_frame_interval;
      break;
    default:
      break;
  }

  return codec_settings;
}

}  // namespace webrtc_demo
```

- ```
  webrtc::InternalEncoderFactory::SupportedFormats
  ```

  先确认我们支持的编码格式，然后创建encoder:

  ```
  std::unique_ptr<webrtc::InternalEncoderFactory> encode_factory =
        std::make_unique<webrtc::InternalEncoderFactory>();
  video_encoder_ = encode_factory->CreateVideoEncoder(format);
  video_encoder_->RegisterEncodeCompleteCallback(this);
  ```

- 在进行

  ```
  Encode
  ```

  编码之前，我们需要想初始化我们的编码器。

  ```
  void VideoEncodeHandler::ReInitEncoder() {
      webrtc::VideoCodec codec_settings =
            DefaultCodecSettings(frame_width_, frame_height_, kBaseKeyFrameInterval);
      video_encoder_->InitEncode(&codec_settings, kSettings);
  }
  ```

- 获取默认的编码器配置是调用了`test/video_codec_settings.h`里的`webrtc::test::CodecSettings`接口进行初步的初始化，实际上完全可以自己仿照着写一个。

- 进行编码`VideoEncoder::Encode`

- 获取编码结果只要通过重载的`EncodedImageCallback::OnEncodedImage`函数就可以实现。

最后在`main`函数中添加：

```C++
#include "examples/desktop_capture/desktop_capture.h"
#include "test/video_renderer.h"
#include "rtc_base/logging.h"
#include "examples/desktop_capture/video_encode_handler.h"

#include <thread>

int main() {
  std::unique_ptr<webrtc_demo::DesktopCapture> capturer(webrtc_demo::DesktopCapture::Create(30,0));

  capturer->StartCapture();

  std::unique_ptr<webrtc::test::VideoRenderer> renderer(webrtc::test::VideoRenderer::Create(capturer->GetWindowTitle().c_str(), 720, 480));
  capturer->AddOrUpdateSink(renderer.get(), rtc::VideoSinkWants());

  std::unique_ptr<webrtc_demo::VideoEncodeHandler> video_encode(webrtc_demo::VideoEncodeHandler::Create(webrtc_demo::VideoEncodeHandler::VideoEncodeType::VP8));
  capturer->AddOrUpdateSink(video_encode.get(), rtc::VideoSinkWants());

  std::this_thread::sleep_for(std::chrono::seconds(30));
  capturer->RemoveSink(renderer.get());
  capturer->RemoveSink(video_encode.get());

  RTC_LOG(WARNING) << "Demo exit";
  return 0;
}
```

视频编码的代码分支： https://github.com/243286065/webrtc-cpp-demo/tree/76e12021e0469d5108930ed4f28df308d6916791
提交commit: https://github.com/243286065/webrtc-cpp-demo/commit/76e12021e0469d5108930ed4f28df308d6916791

> webrtc中其实主要使用`VideoStreamEncoder`对象来创建`VideoEncoder`实例，不过基本原理差不多，只是它做了更多的封装，就编码器来说,和我们的`VideoEncodeHandler`大致实现原理是一样的。