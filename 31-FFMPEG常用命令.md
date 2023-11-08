# 31-FFMPEG常用命令

## 31.0 目录

1. 播放器ffplay常用命令
2. 多媒体分析器ffprobe常用命令
3. 编解码工具ffmpeg常用命令
4. 资料
5. 收获

FFMPEG是一个跨平台的音视频音视频处理的开源套件，我们的学习实践路线如下：
 首先使用PC上使用熟悉基本的常用命令；
 再交叉编译android平台上使用的ffmpeg；
 最后在代码层面学习ffmpeg的代码结构以及具体实现。

本篇，我们先来熟悉ffmpeg的常用命令，先从直观上了解ffmpeg能做什么。

使用FFMPEG之前，我们要先安装对应的应用程序，可以采用从ffmpeg官网上下载源码进行配置编译使用，也可以采用直接安装对应的应用程序，

Mac/Linux平台可以通过如下方式安装编译



```go
1. 在yasm官网http://yasm.tortall.net/Download.html 下载yasm
2. 配置yasm
3. make 编译yasm
4. make install 安装yasm
5. 在官网http://ffmpeg.org/ 下载ffmpeg
6. 配置ffmpeg
7. make编译
8. make install 安装ffmpeg
```

Mac下也可以通过brew安装ffmpeg



```dart
brew iinstall ffmpeg --with-ffplay
```

我们可以通过后者快速安装ffmpeg，先了解ffmpeg能做什么，再来编译或者交叉编译生成对应的ffmpeg命令。

## 31.1 播放器ffplay常用命令

ffplay是以FFMPEG框架为基础，外加SDL构建的多媒体播放器。 支持各种格式的音视频的播放，包括各种封装格式的音视频、以及裸音频pcm或者裸yuv数据，也可以设置音视频同步的方式（以音频为基准、以视频为基准、外部时钟）、播放时可以设置循环模式
 下面我们来具体实践

### 31.1.1 播放音频数据

ffplay music.mp3
 播放音频可以通过快捷键w切换显示模式

![img](https:////upload-images.jianshu.io/upload_images/1791669-aa14f9c310a8d776.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

![img](https:////upload-images.jianshu.io/upload_images/1791669-d667ed5e9103a2cf.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

通过快捷键q退出播放

### 31.1.2 播放视频数据

```css
ffplay video.mp4
```

如果想循环播放可以通过loop来指定循环次数

```css
ffplay video.mp4 -loop 3
```

### 31.1.3 播放yuv数据

使用ffplay播放yuv原始数据表示的视频图片，要告诉ffplay视频的格式、大小、类型，如下所示：



```css
ffplay -f rawvideo -s 640x480 -pix_fmt yuv420p yuv.yuv
```

从mp4中提取出对应的yuv数据，可以通过



```css
ffmpeg -i video.mp4 -s 640x480 -pix_fmt yuv420p yuv.yuv
```

### 31.1.4 设置音视频同步方式

音视频常用的方案有三种 以音频为基准（默认）、以视频为基准、以外部时钟为基准。



```css
ffplay video.mp4 -sync audio
ffplay video.mp4 -sync video
ffplay video.mp4 -sync ext
```

## 31.2 多媒体分析器ffprobe常用命令

ffprobe 的是FFMPEG提供的多媒体探测分析工具，可以分析格式音视频的信息



```bash
ffprobe music.mp3

-->
Duration: 00:04:28.62, start: 0.025057, bitrate: 320 kb/s
    Stream #0:0: Audio: mp3, 44100 Hz, stereo, fltp, 320 kb/s
    Metadata:
      encoder         : LAME3.99r
```

通过上面信息可以看到
 音乐时长信息：Duration: 00:04:28.62；
 开始时间：start: 0.025057
 比特率：320 kb/s
 流的类型：Stream #0:0: Audio: mp3
 采样率：44100 Hz
 声道：stereo

同样的我们可以通过ffprobe来查看视频的信息



```dart
ffprobe video.mp4

-->

Duration: 00:03:25.75, start: 0.000000, bitrate: 629 kb/s
    Stream #0:0(und): Video: h264 (High) (avc1 / 0x31637661), yuv420p, 640x480 [SAR 1:1 DAR 4:3], 496 kb/s, 24 fps, 24 tbr, 12288 tbn, 48 tbc (default)

    Stream #0:1(und): Audio: aac (LC) (mp4a / 0x6134706D), 48000 Hz, stereo, fltp, 128 kb/s (default)
```

我们可以看到视频有两个流：Stream #0:0(und): Video 视频流和Stream #0:1(und): Audio音频流。
 分辨率：640x480
 帧率：24 fps
 视频编码格式： h264 (High) (avc1 / 0x31637661),
 图像存储方式：yuv420p

## 31.3 编解码工具ffmpeg常用命令

ffmpeg命令可以转化各种格式的多媒体文件。
 按照功能分类可以分为如下常用几种类型

1. 信息查询部分
2. 通用参数
3. 视频参数
4. 音频参数

可以通过`ffmpeg --help`查看支持的上述命令
 查询参数的部分如下：



```dart
-formats            show available formats
-muxers             show available muxers
-demuxers           show available demuxers
-devices            show available devices
-codecs             show available codecs
-decoders           show available decoders
-encoders           show available encoders
-bsfs               show available bit stream filters
-protocols          show available protocols
-filters            show available filters
-pix_fmts           show available pixel formats
-layouts            show standard channel layouts
-sample_fmts        show available audio sample formats
-colors             show available color names
```

比如可以通过`ffmpeg -codecs`查看支持的编码类型

下面重点说下通用类型、音频参数和视频参数

### 31.3.1 通用参数



```undefined
-i filename  指定输入文件名  
-y           覆盖同名的输出文件
-f fmt       指定音/视频的格式
-t duration  指定输出音/视频的时长，单位秒
-to time_stop 指定输出音/视频结束点，单位秒
-fs limit_size  限定输出文件大小
-ss time_off    指定输出音/视频的开始时间点，单位秒，也支持hh:mm:ss的格式
```

### 31.3.2 音频参数



```go
-aq quality   指定输出音频的质量
-ar rate      指定音频采样率 (单位 Hz)
-ac channels  指定音频声道数量
-an           输出的文件不带音频
-acodec codec 指定输出的音频编码类型('copy' to copy stream)
-vol volume    指定音频的音量 (256=normal)
-af filter_graph    指定音效
-ab    指定输出音频的比特率
```

### 31.3.3 视频参数



```rust
-r rate   指定帧率 (单位Hz )
-s size   指定分辨率 (WxH)
-aspect aspect  指定宽高比(4:3, 16:9 or 1.3333, 1.7777)
-vn           指定输出文件不包含视频
-vcodec codec 指定输出视频的编码格式 ('copy' to copy stream)

-vf filter_graph 指定视频滤镜
-ab bitrate      指定音频比特率 (please use -b:a)
-b bitrate   指定比特率，若指定该值为平均比特率 (please use -b:v)
-vb 指定视频比特率
```

下面来看下一些比较常用的命令事例

**1. flac格式 --》转成mp3**



```css
ffmpeg -i in.flac -acodec libmp3lame -ar 44100 -ab 320k -ac 2 out.mp3
```

输入的音频文件为in.flac，指定编码格式为libmp3lame（即mp3的编码库），音频采样率44100，比特率为320kb/s，声道数量为2，输出文件为out.mp3

**2. 转换视频格式**



```css
ffmpeg -i in.mov -s 1920x1080 -pix_fmt yuv420p -vcodec libx264 -preset medium -profile:v high -level:v 4.1 -crf 23 -acodec aac -ar 44100 -ac 2 -b:a 128k out.mp4
```

码率控制模式:  -qp  -crf -b

-qp(Constant Quantizer)恒定量化器模式
 -crf(Constant Rate Factor) 恒定速率因子模式
 -b （bitrate） 固定目标码率模式

VBR（Variable Bit Rate）动态比特率 简单的给少些码率、负责的多放些码率

CBR（Constant Bit Rate）恒定比特率  在vbr的基础上改进 码率固定在一个值上

**3. 提取视频**



```css
ffmpeg -i in.mp4 -vcodec copy -an v.mp4
```

还可以从视频文件中直接提取出裸h264数据
 ffmpeg -i in.mp4 -an -vcodec copy -bsf:v h264_mp4toannexb out.h264
 其中mp4toannexb是一种bitmapfilter类型

**4. 提取音频**



```css
ffmpeg -i in.mp4 -vn -acodec copy a.m4a
```

如果多个音频流 通过 -map 0:3来分别提取

eg
 Stream #0:2[0x81]:Audio:ac3,48khz,5.1,s16,384kb/s
 Stream #0:3[0x82]:Audio:ac3,48khz,5.1,s16,384kb/s
 Stream #0:4[0x80]:Audio:ac3,48khz,5.1,s16,384kb/s

再合并回去



```css
ffmpeg -i a.m4a -i v.mp4 -c copy out.mp4
```

**5. 截取**



```css
ffmpeg -i in.mp3 -ss 00:01:00 -to 00:10:00 -acodec copy out.mp3
或者
ffmpeg -i in.mp3 -ss 00:01:00 -t 10 -acodec copy out.mp3
```

把-ss放在-i之前，ffmpeg会启用关键帧技术，加速它的操作，但这样提取出来的时期，在播放时显示的
 起止不一定准确，可以通过 -copyts 拷贝时间戳，保证时间的正确性



```css
ffmpeg -i in.mp3 -ss 00:01:00 -to 00:10:00 -acodec copy out.mp3
ffmpeg  -ss 00:01:00 -i in.mp3 -to 00:10:00 -acodec copy out.mp3
ffmpeg  -ss 00:01:00 -i in.mp3 -to 00:10:00 -acodec copy-copyts out.mp3
```

还可以截取多个文件



```css
ffmpeg -i video.mp4 -t 00:00:50 -c copy small-1.mp4 -ss 00:00:50 -t 10 -codec copy small-2.mp4 -y
```

**6. 把多个视频连接成一个视频**
 多个视频的宽高、码率一致



```objectivec
ffmpeg -i "concat:01.mp4|02.mp4|03.mp4" -c copy out.mp4
```

如果参数不一致推荐使用Avidemux这款软件处理

**7. 截图**



```css
ffmpeg -i in.mp4 -ss 5 -vframes 1 img.jpg
```

上述指令是指 在第五秒截图

还可以采用更加方便的方式截一系列的图



```css
ffmpeg -i in.mp4 -r 0.25  frames_%04d.png
```

上述命令指定每4秒截取一帧画面生成一张图片，

**8. 水印**



```csharp
ffmpeg -i in.mp4 -i logo.png -filter_complex "overlay=20:20" out.mp4
```

通过两个-i指定两个输入源，一个是视频，一个是水印，通过filter_complex指定水印的位置_

**9. 动图**



```css
ffmpeg -i in.mp4 -ss 7.5 -to 8.5 -s 640x320 -r 15 out.gif
```

指定分辨率为640x320，帧率为15

或者可以通过如下命令指定宽高



```csharp
ffmpeg -i in.mp4 -vf scale=100:-1 -t 5 -r 10 out.gif
```

宽高比不变，宽度指定为100

还可以把一组图片生成一张动图



```css
ffmpeg -i frames_%04d.png -r 5 out.gif
```

**10.淡入淡出效果**
 给一个音频做一个淡入的效果,可以通过如下命令



```ruby
ffmpeg -i music.mp3 -filter_complex afade=t=in:ss=0:d=5 music11.mp3
```

前5秒做个淡入效果

淡出效果可以采用如下命令



```csharp
ffmpeg -i music.mp3 -filter_complex afade=t=out:st=200:d=5 music22.mp3
```

从200秒开始做5秒的淡出效果

**11. 倍速**
 两倍速处理音频



```undefined
ffmpeg -i music.mp3 -filter_complex atempo=2 out1.mp3 -y
```

0.5倍速处理音频



```undefined
ffmpeg -i music.mp3 -filter_complex atempo=0.5 out1.mp3 -y
```

**12.混音**
 将两个声音进行合并



```ruby
ffmpeg -i music.mp3 -i bg.mp3 -filter_complex amix=inputs=2:duration=shortest out.mp3
```

## 31.4 资料

《FFmpeg从入门到精通》
 《音视频开发进阶指南》
 [【FFmpeg 分P教学】转码、压制、录屏、裁切、合并、提取 … 统统不是问题。](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV1Ft411s7Xa%3Fp%3D9)
 [基于FFmpeg+SDL的视频播放器的制作——雷霄骅](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.bilibili.com%2Fvideo%2FBV14x411D7FD%3Ffrom%3Dsearch%26seid%3D12150037577862710226)

## 31.5 收获

1. 了解ffplay常用命令的使用
2. 了解ffprobe的使用
3. 了解ffmpeg常用命令
4. 学习实践常用命令的使用场景



 