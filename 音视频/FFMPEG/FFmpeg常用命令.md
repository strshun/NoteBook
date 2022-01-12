

参考链接 [https://www.ostechnix.com/20-ffmpeg-commands-beginners/](https://www.ostechnix.com/20-ffmpeg-commands-beginners/)

## 获取音视频文件信息

要显示媒体文件的详细信息，请运行

```
$ ffmpeg -i video.mp4
```

如果您不想看到FFmpeg标语和其他详细信息，而只希望看到媒体文件信息，请使用-hide_banner标志，如下所示

```
$ ffmpeg -i video.mp4 -hide_banner
```

## 将视频文件转换为不同格式

FFmpeg是强大的音频和视频转换器，因此可以在不同格式之间转换媒体文件。例如，要将mp4文件转换为avi文件，请运行：

```
$ ffmpeg -i video.mp4 video.avi   //mp4 file to avi file

$ ffmpeg -i video.flv video.mpeg  //flv to mpeg
```

如果要保留源视频文件的质量，请使用“ -qscale 0”参数：

```
$ ffmpeg -i input.webm -qscale 0 output.mp4
```

要检查FFmpeg支持的格式列表，请运行

```
$ ffmpeg -formats
```

## 将视频文件转换为音频文件

要将视频文件转换为音频文件，只需将输出格式指定为.mp3或.ogg或任何其他音频格式。

```
$ ffmpeg -i input.mp4 -vn output.mp3  //input.mp4 video file to output.mp3 audio file
```

另外，您可以对输出文件使用各种音频转码选项，如下所示。

```
$ ffmpeg -i input.mp4 -vn -ar 44100 -ac 2 -ab 320 -f mp3 output.mp3
```

* **-vn** ---  表示我们已禁用输出文件中的视频录制。
* **-ar**  ---  设置输出文件的音频频率。使用的常用值为22050、44100、48000 Hz。
* **-ac**  --- 设置音频通道数。
* **-ab** --- 表示音频比特率
* **f** --- 输出文件格式。就我们而言，它是mp3格式。

## 更改视频文件的分辨率

如果要为视频文件设置特定的分辨率，可以使用以下命令：

```
$ ffmpeg -i input.mp4 -filter:v scale=1280:720 -c:a copy output.mp4
or
$ ffmpeg -i input.mp4 -s 1280x720 -c:a copy output.mp4
```

上面的命令会将给定视频文件的分辨率设置为1280×720。

此技巧将帮助您将视频文件缩放到较小的显示设备，例如平板电脑和手机。

## 压缩视频文件

最好将媒体文件的大小减小到较小的大小，以节省硬盘的空间

以下命令将压缩并减小输出文件的大小。

```
$ ffmpeg -i input.mp4 -vf scale=1280:-1 -c:v libx264 -preset veryslow -crf 24 output.mp4
```

请注意，如果尝试减小视频文件的大小，将会失去质量。如果24过于激进，则可以将该crf值降低至23或更低。

您还可以对音频进行一点点转码，并使其立体声，以通过包括以下选项来减小尺寸。

```
-ac 2 -c:a aac -strict -2 -b:a 128k
```

## 压缩音频文件

与压缩视频文件一样，您也可以使用-ab标志压缩音频文件，以节省一些磁盘空间。

假设您有一个320 kbps比特率的音频文件。您想通过将比特率更改为任何较低的值来压缩它，如下所示

```
$ ffmpeg -i input.mp3 -ab 128 output.mp3
```

各种可用音频比特率的列表是：

* 96kbps
* 12kbps
* 128kbps
* 160kbps
* 192kbps
* 256kbps
* 320kbps

## 从视频文件中删除音频流

如果您不想从视频文件中获取音频，请使用-an标志。

```
$ ffmpeg -i input.mp4 -an output.mp4
```

在这里，“ an”表示没有录音。

上面的命令将撤消所有与音频相关的标志。

## 从媒体文件中删除视频流

同样，如果您不想要视频流，则可以使用“ vn”标志轻松地将其从媒体文件中删除。 vn代表不进行视频录制。换句话说，此命令将给定的媒体文件转换为音频文件。

以下命令将从指定的媒体文件中删除视频。

```
$ ffmpeg -i input.mp4 -vn output.mp3
```

您还可以使用“ -ab”标志来提及输出文件的比特率，如下例所示。

```
$ ffmpeg -i input.mp4 -vn -ab 320 output.mp3
```

## 从视频中提取图像

FFmpeg的另一个有用功能是我们可以轻松地从视频文件中提取图像。如果要从视频文件创建相册，这可能非常有用。

要从视频文件中提取图像，请使用以下命令：

```
$ ffmpeg -i input.mp4 -r 1 -f image2 image-%2d.png
```

* **-r**     设置帧频。即每秒要提取到图像中的帧数。默认值25
* **-f **  表示输出格式，即本例中的图像格式。
* **image-%2d.png**  表示我们要如何命名提取的图像。在这种情况下，名称应以image-01.png，image-02.png，image-03.png等开头。如果使用％3d，则图像名称将以image-001.png，image-002.png等开头。

## 裁剪视频

FFMpeg允许在我们选择的任意维度上裁剪给定的媒体文件。

下面给出了裁剪视频的语法：

```
ffmpeg -i input.mp4 -filter:v "crop=w:h:x:y" output.mp4
```

- **input.mp4** – 源视频文件。
- **-filter:v** –指示视频过滤器
- **crop** – Indicates crop filter.
- **w** – **Width** of the rectangle that we want to crop from the source video. 我们要从源视频中裁剪的矩形的大小
- **h** – Height of the rectangle. 矩形的高度
- **x** – **x coordinate** of the rectangle that we want to crop from the source video. 我们要从源视频中裁剪的矩形的x坐标
- **y** – y coordinate of the rectangle. 矩形的y坐标。

假设您要从位置（200,150）开始播放一个宽度为640像素，高度为480像素的视频，命令如下：

```
$ ffmpeg -i input.mp4 -filter:v "crop=640:480:200:150" output.mp4
```

请注意，裁剪视频会影响质量。除非有必要，否则不要这样做。

## 转换视频的特定部分

有时，您可能只想将视频文件的特定部分（持续时间）转换为其他格式。例如，以下命令会将给定video.mp4文件的前10秒转换为video.avi格式。

```
$ ffmpeg -i input.mp4 -t 10 output.avi
```

在这里，我们以秒为单位指定时间。另外，可以以hh.mm.ss格式指定时间。

## 设置视频宽高比

您可以使用-aspect标志（如下所示）将宽高比设置为视频文件。

```
$ ffmpeg -i input.mp4 -aspect 16:9 output.mp4
```

常用的宽高比是：

- 16:9
- 4:3
- 16:10
- 5:4
- 2:21:1
- 2:35:1
- 2:39:1

## 将海报图像添加到音频文件

您可以将海报图像添加到文件中，以便在播放音频文件时显示图像。这对于在视频托管或共享网站中托管音频文件很有用。

```
$ ffmpeg -loop 1 -i inputimage.jpg -i inputaudio.mp3 -c:v libx264 -c:a aac -strict experimental -b:a 192k -shortest output.mp4
```

## 使用开始和停止时间修剪媒体文件

要使用开始和停止时间将视频修剪为较小的剪辑，我们可以使用以下命令。

```
$ ffmpeg -i input.mp4 -ss 00:00:50 -codec copy -t 50 output.mp4
```

- s – 表示视频剪辑的开始时间。在我们的示例中，开始时间是第50秒。
- -t – 表示总持续时间

当您想使用开始和结束时间从音频或视频文件中剪切部分时，这非常有用。

同样，我们可以像下面那样修剪音频文件

```
$ ffmpeg -i audio.mp3 -ss 00:01:54 -to 00:06:53 -c copy output.mp3
```

## 将视频文件分成多个部分

一些网站只允许您上传特定大小的视频。在这种情况下，您可以将大型视频文件拆分为多个较小的部分，如下所示。

```
$ ffmpeg -i input.mp4 -t 00:00:30 -c copy part1.mp4 -ss 00:00:30 -codec copy part2.mp4
```

在此，-t 00:00:30表示从视频开始到视频的30秒创建的部分。 -ss 00:00:30显示视频下一部分的开始时间戳。这意味着第二部分将从30秒开始，并将一直持续到原始视频文件的末尾。

## 将多个视频部分合并或合并为一个

FFmpeg还将加入多个视频部分，并创建一个视频文件。

创建join.txt文件，其中包含要加入的文件的确切路径。所有文件应为相同格式（相同编解码器）。所有文件的路径名都应像下面这样一一提及。

```
file /home/sk/myvideos/part1.mp4
file /home/sk/myvideos/part2.mp4
file /home/sk/myvideos/part3.mp4
file /home/sk/myvideos/part4.mp4
```

现在，使用命令加入所有文件：

```
$ ffmpeg -f concat -i join.txt -c copy output.mp4
```

如果出现错误，则如下所示；

```
[concat @ 0x555fed174cc0] Unsafe file name '/path/to/mp4'
join.txt: Operation not permitted
```

Add **“-safe 0”**:

```
$ ffmpeg -f concat -safe 0 -i join.txt -c copy output.mp4
```

上面的命令会将part1.mp4，part2.mp4，part3.mp4和part4.mp4文件连接到一个名为“ output.mp4”的文件中。

## 将字幕添加到视频文件

我们还可以使用FFmpeg将字幕添加到视频文件中。为视频下载正确的字幕，然后将其添加到视频中，如下所示。

```
$ fmpeg -i input.mp4 -i subtitle.srt -map 0 -map 1 -c copy -c:v libx264 -crf 23 -preset veryfast output.mp4
```

## 预览或测试视频或音频文件

您可能需要预览以验证或测试输出文件是否已正确转码。为此，您可以使用以下命令从终端中播放它：

```
ffplay video.mp4
```

同样，您可以如下所示测试音频文件。

```
$ ffplay audio.mp3
```

## 提高/降低视频播放速度

FFmpeg允许您调整视频播放速度

要提高视频播放速度，请运行：

```
$ ffmpeg -i input.mp4 -vf "setpts=0.5*PTS" output.mp4
```

该命令将使视频速度翻倍。 

要降低视频速度，您需要使用大于1的乘数。要降低播放速度，请运行：

```
$ ffmpeg -i input.mp4 -vf "setpts=4.0*PTS" output.mp4
```

## 创建动画GIF

```
我们出于各种目的在几乎所有社交和专业网络上使用GIF图片。使用FFmpeg，我们可以轻松快速地创建动画视频文件。以下指南说明了如何在类似Unix的系统中使用FFmpeg和ImageMagick创建动画GIF文件。
```

[**How To Create Animated GIF In Linux**](https://www.ostechnix.com/create-animated-gif-ubuntu-16-04/)

**Suggested read:**

[**Gifski – A Cross-platform High-quality GIF Encoder**](https://www.ostechnix.com/gifski-a-cross-platform-high-quality-gif-encoder/)



## 从PDF文件创建视频

[**How To Create A Video From PDF Files In Linux**](https://www.ostechnix.com/create-video-pdf-files-linux/)

## 旋转影片

[**How To Rotate Videos Using FFMpeg From Commandline**](https://www.ostechnix.com/how-to-rotate-videos-using-ffmpeg-from-commandline/)

## 将视频转换为WhatsApp视频格式

[**Convert Videos To WhatsApp Video Format With FFmpeg**](https://www.ostechnix.com/convert-videos-to-whatsapp-video-format-with-ffmpeg/)

