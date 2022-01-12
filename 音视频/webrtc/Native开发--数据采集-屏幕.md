## 1. 实时音视频开发主要步骤

![image](https://github.com/243286065/pictures_markdown/blob/master/webrtc/47d54ff6638beef0c09cd7d85b85441f.png?raw=true)

## 2. 屏幕采集

在上一篇文章中我们讲述了如何采集摄像头的数据，这篇文章就主要讲述如何采集屏幕的数据。

摄像头采集使用的模块主要是`webrtc::VideoCaptureModule`，代码位于`modules/video_capture`下；
屏幕采集主使用的模块主要是`webrtc::DesktopCapturer`,位于`modules/desktop_capture`。

从`src/modules/desktop_capture/desktop_capturer.h`的接口定义中我们能够能方便地实现屏幕抓取：

```C++
  // Creates a DesktopCapturer instance which targets to capture windows.
  static std::unique_ptr<DesktopCapturer> CreateWindowCapturer(
      const DesktopCaptureOptions& options);

  // Creates a DesktopCapturer instance which targets to capture screens.
  static std::unique_ptr<DesktopCapturer> CreateScreenCapturer(
      const DesktopCaptureOptions& options);
```

- `DesktopCapturer::CreateWindowCapturer`主要是抓取应用窗口；

- `DesktopCapturer::CreateScreenCapturer`主要就是抓取屏幕。

- 创建完`DesktopCapturer`实例后，可以通过`DesktopCapturer::GetSourceList`来获取可选的窗口或者屏幕信息；

- 通过`DesktopCapturer::SelectSource`可以选定要抓取的窗口或者屏幕；

- 通过`DesktopCapturer::Start`来注册一个`DesktopCapturer::Callback`回调，用于接收抓取的`DesktopFrame`一帧数据。

- 需要调用`DesktopCapturer::CaptureFrame`,才会由`DesktopCapturer::Start`注册的回调返回一帧数据，因此抓屏和摄像头不同，是需要自己手动调用`DesktopCapturer::CaptureFrame`来控制帧率。

- 注意

  ```
  DesktopCapturer::Callback
  ```

  回调回来的是一个

  ```
  webrtc::DesktopFrame
  ```

  ,它是个

  ```
  RGBA
  ```

  数据，而我们之前在采集摄像头时使用的

  ```
  webrtc::VideoFrame
  ```

  是个

  ```
  I420
  ```

  数据，我们需要使用

  ```
  libyuv
  ```

  将其进行一个转换:

  ```C++
  rtc::scoped_refptr<webrtc::I420Buffer> i420_buffer_;
  
  if (!i420_buffer_.get() ||
    i420_buffer_->width() * i420_buffer_->height() < width * height) {
  i420_buffer_ = webrtc::I420Buffer::Create(width, height);
  }
  
  libyuv::ConvertToI420(frame->data(), 0, i420_buffer_->MutableDataY(),
                      i420_buffer_->StrideY(), i420_buffer_->MutableDataU(),
                      i420_buffer_->StrideU(), i420_buffer_->MutableDataV(),
                      i420_buffer_->StrideV(), 0, 0, width, height, width,
                      height, libyuv::kRotate0, libyuv::FOURCC_ARGB);
  
  webrtc::VideoFrame video_frame(i420_buffer_, 0, 0, webrtc::kVideoRotation_0);
  ...
  ```

- 然后，和前面采集摄像头数据一样，我们要将捕获到的数据提供出去，供渲染或者传输，那么就是个`Source`，那么我们还需要实现一个`Source`接口。

## 3. 实现

首先是顶一个`Source`接口：

```C++
// desktop_capture_source.h

#ifndef EXAMPLES_DESKTOP_CAPTURE_DESKTOP_CAPTURER_SOURCE_TEST_H_
#define EXAMPLES_DESKTOP_CAPTURE_DESKTOP_CAPTURER_SOURCE_TEST_H_

#include "api/video/video_frame.h"
#include "api/video/video_source_interface.h"
#include "media/base/video_adapter.h"
#include "media/base/video_broadcaster.h"

namespace webrtc_demo {

class DesktopCaptureSource
    : public rtc::VideoSourceInterface<webrtc::VideoFrame> {
 public:
  DesktopCaptureSource() {}
  ~DesktopCaptureSource() override {}

  void AddOrUpdateSink(rtc::VideoSinkInterface<webrtc::VideoFrame>* sink,
                       const rtc::VideoSinkWants& wants) override;

  void RemoveSink(rtc::VideoSinkInterface<webrtc::VideoFrame>* sink) override;

 protected:
  // Notify sinkes
  void OnFrame(const webrtc::VideoFrame& frame);

 private:
  void UpdateVideoAdapter();

  rtc::VideoBroadcaster broadcaster_;
  cricket::VideoAdapter video_adapter_;
};

}  // namespace webrtc_demo

#endif  // EXAMPLES_DESKTOP_CAPTURE_DESKTOP_CAPTURER_SOURCE_TEST_H_
// desktop_capture_source.cc

#include "examples/desktop_capture/desktop_capture_source.h"

#include "api/video/i420_buffer.h"
#include "api/video/video_rotation.h"
#include "rtc_base/logging.h"

namespace webrtc_demo {

void DesktopCaptureSource::AddOrUpdateSink(
    rtc::VideoSinkInterface<webrtc::VideoFrame>* sink,
    const rtc::VideoSinkWants& wants) {
  broadcaster_.AddOrUpdateSink(sink, wants);
  UpdateVideoAdapter();
}

void DesktopCaptureSource::RemoveSink(
    rtc::VideoSinkInterface<webrtc::VideoFrame>* sink) {
  broadcaster_.RemoveSink(sink);
  UpdateVideoAdapter();
}

void DesktopCaptureSource::UpdateVideoAdapter() {
  video_adapter_.OnSinkWants(broadcaster_.wants());
}

void DesktopCaptureSource::OnFrame(const webrtc::VideoFrame& frame) {
  int cropped_width = 0;
  int cropped_height = 0;
  int out_width = 0;
  int out_height = 0;

  if (!video_adapter_.AdaptFrameResolution(
          frame.width(), frame.height(), frame.timestamp_us() * 1000,
          &cropped_width, &cropped_height, &out_width, &out_height)) {
    // Drop frame in order to respect frame rate constraint.
    return;
  }

  if (out_height != frame.height() || out_width != frame.width()) {
    // Video adapter has requested a down-scale. Allocate a new buffer and
    // return scaled version.
    // For simplicity, only scale here without cropping.
    rtc::scoped_refptr<webrtc::I420Buffer> scaled_buffer =
        webrtc::I420Buffer::Create(out_width, out_height);
    scaled_buffer->ScaleFrom(*frame.video_frame_buffer()->ToI420());
    webrtc::VideoFrame::Builder new_frame_builder =
        webrtc::VideoFrame::Builder()
            .set_video_frame_buffer(scaled_buffer)
            .set_rotation(webrtc::kVideoRotation_0)
            .set_timestamp_us(frame.timestamp_us())
            .set_id(frame.id());
    ;
    if (frame.has_update_rect()) {
      webrtc::VideoFrame::UpdateRect new_rect =
          frame.update_rect().ScaleWithFrame(frame.width(), frame.height(), 0,
                                             0, frame.width(), frame.height(),
                                             out_width, out_height);
      new_frame_builder.set_update_rect(new_rect);
    }
    broadcaster_.OnFrame(new_frame_builder.build());
  } else {
    // No adaptations needed, just return the frame as is.
    broadcaster_.OnFrame(frame);
  }
}

}  // namespace webrtc_demo
```

然后实现`Sink`，并适配`Source`接口，其实是可以写在一起的：

```C++
//desktop_capture.h

#ifndef EXAMPLES_DESKTOP_CAPTURE_DESKTOP_CAPTURER_TEST_H_
#define EXAMPLES_DESKTOP_CAPTURE_DESKTOP_CAPTURER_TEST_H_

#include "api/video/video_frame.h"
#include "api/video/video_sink_interface.h"
#include "examples/desktop_capture/desktop_capture_source.h"
#include "modules/desktop_capture/desktop_capturer.h"
#include "modules/desktop_capture/desktop_frame.h"
#include "api/video/i420_buffer.h"


#include <thread>
#include <atomic>

namespace webrtc_demo {

class DesktopCapture : public DesktopCaptureSource,
                       public webrtc::DesktopCapturer::Callback,
                       public rtc::VideoSinkInterface<webrtc::VideoFrame> {
 public:
  static DesktopCapture* Create(size_t target_fps, size_t capture_screen_index);

  ~DesktopCapture() override;

  std::string GetWindowTitle() const { return window_title_; }

  void StartCapture();
  void StopCapture();

 private:
  DesktopCapture();

  void Destory();

  void OnFrame(const webrtc::VideoFrame& frame) override {}

  bool Init(size_t target_fps, size_t capture_screen_index);

  void OnCaptureResult(webrtc::DesktopCapturer::Result result,
                       std::unique_ptr<webrtc::DesktopFrame> frame) override;

  std::unique_ptr<webrtc::DesktopCapturer> dc_;

  size_t fps_;
  std::string window_title_;

  std::unique_ptr<std::thread> capture_thread_;
  std::atomic_bool start_flag_;

  rtc::scoped_refptr<webrtc::I420Buffer> i420_buffer_;
};
}  // namespace webrtc_demo

#endif  // EXAMPLES_DESKTOP_CAPTURE_DESKTOP_CAPTURER_TEST_H_
// desktop_capture.cc

#include "examples/desktop_capture/desktop_capture.h"

#include "modules/desktop_capture/desktop_capture_options.h"
#include "rtc_base/logging.h"
#include "third_party/libyuv/include/libyuv.h"

namespace webrtc_demo {

DesktopCapture::DesktopCapture() : dc_(nullptr), start_flag_(false) {}

DesktopCapture::~DesktopCapture() {
  Destory();
}

void DesktopCapture::Destory() {
  StopCapture();

  if (!dc_)
    return;

  dc_.reset(nullptr);
}

DesktopCapture* DesktopCapture::Create(size_t target_fps,
                                       size_t capture_screen_index) {
  std::unique_ptr<DesktopCapture> dc(new DesktopCapture());
  if (!dc->Init(target_fps, capture_screen_index)) {
    RTC_LOG(LS_WARNING) << "Failed to create DesktopCapture(fps = "
                        << target_fps << ")";
    return nullptr;
  }
  return dc.release();
}

bool DesktopCapture::Init(size_t target_fps, size_t capture_screen_index) {
  dc_ = webrtc::DesktopCapturer::CreateScreenCapturer(
      webrtc::DesktopCaptureOptions::CreateDefault());

  if (!dc_)
    return false;

  webrtc::DesktopCapturer::SourceList sources;
  dc_->GetSourceList(&sources);
  if (capture_screen_index > sources.size()) {
    RTC_LOG(LS_WARNING) << "The total sources of screen is " << sources.size()
                        << ", but require source of index at "
                        << capture_screen_index;
    return false;
  }

  RTC_CHECK(dc_->SelectSource(sources[capture_screen_index].id));
  window_title_ = sources[capture_screen_index].title;
  fps_ = target_fps;

  RTC_LOG(LS_INFO) << "Init DesktopCapture finish";
  // Start new thread to capture
  return true;
}

void DesktopCapture::OnCaptureResult(
    webrtc::DesktopCapturer::Result result,
    std::unique_ptr<webrtc::DesktopFrame> frame) {
  RTC_LOG(LS_INFO) << "new Frame";

  static auto timestamp =
      std::chrono::duration_cast<std::chrono::milliseconds>(
          std::chrono::system_clock::now().time_since_epoch())
          .count();
  static size_t cnt = 0;

  cnt++;
  auto timestamp_curr = std::chrono::duration_cast<std::chrono::milliseconds>(
                            std::chrono::system_clock::now().time_since_epoch())
                            .count();
  if (timestamp_curr - timestamp > 1000) {
    RTC_LOG(LS_INFO) << "FPS: " << cnt;
    cnt = 0;
    timestamp = timestamp_curr;
  }

  // Convert DesktopFrame to VideoFrame
  if (result != webrtc::DesktopCapturer::Result::SUCCESS) {
    RTC_LOG(LS_ERROR) << "Capture frame faiiled, result: " << result;
  }
  int width = frame->size().width();
  int height = frame->size().height();
  // int half_width = (width + 1) / 2;

  if (!i420_buffer_.get() ||
      i420_buffer_->width() * i420_buffer_->height() < width * height) {
    i420_buffer_ = webrtc::I420Buffer::Create(width, height);
  }

  libyuv::ConvertToI420(frame->data(), 0, i420_buffer_->MutableDataY(),
                        i420_buffer_->StrideY(), i420_buffer_->MutableDataU(),
                        i420_buffer_->StrideU(), i420_buffer_->MutableDataV(),
                        i420_buffer_->StrideV(), 0, 0, width, height, width,
                        height, libyuv::kRotate0, libyuv::FOURCC_ARGB);

  // Notify sinks
  DesktopCaptureSource::OnFrame(
      webrtc::VideoFrame(i420_buffer_, 0, 0, webrtc::kVideoRotation_0));
}

void DesktopCapture::StartCapture() {
  if (start_flag_) {
    RTC_LOG(WARNING) << "Capture already been running...";
    return;
  }

  start_flag_ = true;

  // Start new thread to capture
  capture_thread_.reset(new std::thread([this]() {
    dc_->Start(this);

    while (start_flag_) {
      dc_->CaptureFrame();
      std::this_thread::sleep_for(std::chrono::milliseconds(1000 / fps_));
    }
  }));
}

void DesktopCapture::StopCapture() {
  start_flag_ = false;

  if (capture_thread_ && capture_thread_->joinable()) {
    capture_thread_->join();
  }
}

}  // namespace webrtc_demo
```

这里就是使用了`webrtc::DesktopCapturer::CreateScreenCapturer`来创建一个`DesktopCapturer`实例抓取屏幕。

最后，在`main`函数中将一个`webrtc::test::VideoRenderer`作为`Sink`提供给我们的`DesktopCapturer`.

```C++
// main.cc

#include "examples/desktop_capture/desktop_capture.h"
#include "test/video_renderer.h"
#include "rtc_base/logging.h"

#include <thread>

int main() {
  std::unique_ptr<webrtc_demo::DesktopCapture> capturer(webrtc_demo::DesktopCapture::Create(15,0));

  capturer->StartCapture();

  std::unique_ptr<webrtc::test::VideoRenderer> renderer(webrtc::test::VideoRenderer::Create(capturer->GetWindowTitle().c_str(), 720, 480));
  capturer->AddOrUpdateSink(renderer.get(), rtc::VideoSinkWants());

  std::this_thread::sleep_for(std::chrono::seconds(30));
  capturer->RemoveSink(renderer.get());

  RTC_LOG(WARNING) << "Demo exit";
  return 0;
}
```

从采集摄像头的例子到这里采集屏幕的例子，分别是使用了`video_capture`和`desktop_capture`，整体看，webrtc的模块划分相当的清晰明了。

采集屏幕的例子代码分支：https://github.com/243286065/webrtc-cpp-demo/tree/854a8229dbeb20ab3bdae208b21abf9ae86e9623

提交diff: https://github.com/243286065/webrtc-cpp-demo/commit/854a8229dbeb20ab3bdae208b21abf9ae86e9623