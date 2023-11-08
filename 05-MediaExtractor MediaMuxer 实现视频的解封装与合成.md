# 05-MediaExtractor MediaMuxer 实现视频的解封装与合成

 [腾讯课堂 零声教育](https://0voice.ke.qq.com) 整理，  版权归原作者所有，如有侵权请联系删除，原文链接：

## 5.0 目录

1. MediaExtractor MediaMuxer 能做什么
2. 视频解封装和合成的API以及流程介绍
3. 三个实践（视频解封装提取纯音轨和视频轨文件、再合成新视频、给视频换个背景音）
4. 遇到的问题
5. 收获

## 5.1 有什么实际应用

在我们日常使用短视频软件的时候，对视频的裁剪，拼凑，加入背景是很常用的操作，这些功能是如何实现的呐？其实是将视频多信道的分离出来，比如音轨和视频轨道分隔出来，可以做到二次合成。

今天我们通过对来MediaExtractor和MediaMuxer的学习分析和实践来实现 “把视频分离（提取&解封装）出纯音频和纯视频文件”、“替换背景音乐，合成新的视频文件”。

## 5.2 视频解封装和合成的API以及流程介绍

### 5. 2.1 MediaExtractor：视频轨道提取器（解封装）

**主要API介绍**

-  setDataSource(path)：path本地或者网络文件
- getTrackCount：获取轨道数
-  getTrackFormat(i):对应轨道的格式 MediaFormat
-  selectTrack(I）：切换到（选定）某个轨道
- readSampleData（ByteBuffer byteBuff, int offset）: 把指定轨道中的样本数据按偏移量读取到ByteBuffer字节缓冲区
- advance():  提取到下一帧数据  作用有点类似于cursor
- unselectTrack(i)
-  release()
- getSampleFlags: 获取数据的flag，数据为什么要用Sample来表示，因为音视频的数据是采样数据。
- getSampleTime：返回当前的时间戳

**数据提取(解封装)流程如下：**

```dart
//1. 构造MediaExtractor
MediaExtractor mediaExtractor = new MediaExtractor();
try {
    //2.设置数据源
    mediaExtractor.setDataSource(inputFile.getAbsolutePath());
    //3. 获取轨道数
    int trackCount = mediaExtractor.getTrackCount();
    Log.i(TAG, "demuxerMP4: trackCount=" + trackCount);
    //遍历轨道，查看音频轨或者视频轨道信息
    for (int i = 0; i < trackCount; i++) {
        //4. 获取某一轨道的媒体格式
        MediaFormat trackFormat = mediaExtractor.getTrackFormat(i);
        String keyMime = trackFormat.getString(MediaFormat.KEY_MIME);
        Log.i(TAG, "demuxerMp4: keyMime=" + keyMime);
        if (TextUtils.isEmpty(keyMime)) {
            continue;
        }
        //5.通过mime信息识别音轨或视频轨道，打印相关信息
        if (keyMime.startsWith("video/")) {
            //打印视频的宽高
            Log.i(TAG, "extractorAndMuxerMP4:                     videoWidth="+trackFormat.getInteger(MediaFormat.KEY_WIDTH)+" videoHeight="+trackFormat.getInteger(MediaFormat.KEY_HEIGHT));

        } else if (keyMime.startsWith("audio/")) {
            //打印音轨的通道数以及比特率
            Log.i(TAG, "extractorAndMuxerMP4: channelCount="+trackFormat.getInteger(MediaFormat.KEY_CHANNEL_COUNT)+" bitRate="+trackFormat.getInteger(MediaFormat.KEY_BIT_RATE));
        }
    }
} catch (IOException e) {
    e.printStackTrace();
} finally {
    mediaExtractor.release();
}
```

### 5.2.2  MediaMuxer：合成（封装）

把音轨和视频轨道合成封装为新的视频
 **主要API介绍**

- MediaMuxer（path，format）：path 输出文件的名称；foramt输出文件的格式，当前只支持mp4
- addTrack（trackFormat）：添加轨道，通常是使用MediaCodec.getOutputForma()或MediaExtractor.getTrackFormat(int index)来获取MediaFormat
- start()：开始封装合成
- writeSampleData (int trackIndex, ByteBuffer byteBuf, MediaCodec.BufferInfo bufferInfo): 把数据写入到
- stop()
- release()

**封装(合成)流程如下：**

```cpp
{
    MediaFormat trackFormat = mediaExtractor.getTrackFormat(i);
    MediaMuxer mediaMuxer;
    mediaExtractor.selectTrack(i);

    //1. 构造MediaMuxer
    mediaMuxer = new MediaMuxer(outputFile.getAbsolutePath(), MediaMuxer.OutputFormat.MUXER_OUTPUT_MPEG_4);
    //2. 添加轨道信息 参数为MediaFormat
    mediaMuxer.addTrack(trackFormat);
    //3. 开始合成
    mediaMuxer.start();

    //4. 设置buffer
    ByteBuffer buffer = ByteBuffer.allocate(500 * 1024);
    MediaCodec.BufferInfo bufferInfo = new MediaCodec.BufferInfo();

    //5.通过mediaExtractor.readSampleData读取数据流
    int sampleSize = 0;
    while ((sampleSize = mediaExtractor.readSampleData(buffer, 0)) > 0) {
        bufferInfo.flags = mediaExtractor.getSampleFlags();
        bufferInfo.offset = 0;
        bufferInfo.size = sampleSize;
        bufferInfo.presentationTimeUs = mediaExtractor.getSampleTime();
        int isEOS = bufferInfo.flags & MediaCodec.BUFFER_FLAG_END_OF_STREAM;
        Log.i(TAG, "demuxerMp4:  flags=" + bufferInfo.flags + " size=" + sampleSize + " time=" + bufferInfo.presentationTimeUs + " outputName" + outputName+" isEOS="+isEOS);
        //6. 把通过mediaExtractor解封装的数据通过writeSampleData写入到对应的轨道
        mediaMuxer.writeSampleData(0, buffer, bufferInfo);
        mediaExtractor.advance();
    }
    Log.i(TAG, "extractorAndMuxer: " + outputName + "提取封装完成");

    mediaExtractor.unselectTrack(i);
    //6.关闭
    mediaMuxer.stop();
    mediaMuxer.release();
}
```

## 5.3 实践（以及ffmpeg的实现）

**1. 提取视频分离出纯音频和纯视频文件**

```java
private void extractorAndMuxerMP4() {
    tvOut.setText("");
    File inputFile = new File(getExternalFilesDir(Environment.DIRECTORY_MUSIC), "forme.mp4");
    if (!inputFile.exists()) {
        Toast.makeText(this, "文件不存在", Toast.LENGTH_SHORT).show();
        return;
    }

    //数据提取(解封装)
    //1. 构造MediaExtractor
    MediaExtractor mediaExtractor = new MediaExtractor();
    try {
        //2.设置数据源
        mediaExtractor.setDataSource(inputFile.getAbsolutePath());
        //3. 获取轨道数
        int trackCount = mediaExtractor.getTrackCount();
        Log.i(TAG, "demuxerMP4: trackCount=" + trackCount);
        //遍历轨道，查看音频轨或者视频轨道信息
        for (int i = 0; i < trackCount; i++) {
            //4. 获取某一轨道的媒体格式
            MediaFormat trackFormat = mediaExtractor.getTrackFormat(i);
            String keyMime = trackFormat.getString(MediaFormat.KEY_MIME);
            Log.i(TAG, "demuxerMp4: keyMime=" + keyMime);
            if (TextUtils.isEmpty(keyMime)) {
                continue;
            }
            //5.通过mime信息识别音轨或视频轨道，打印相关信息
            if (keyMime.startsWith("video/")) {
                File outputFile = extractorAndMuxer(mediaExtractor, i, "/video.mp4");
                tvOut.setText("纯视频文件路径：" + outputFile.getAbsolutePath());
                Log.i(TAG, "extractorAndMuxerMP4: videoWidth="+trackFormat.getInteger(MediaFormat.KEY_WIDTH)+" videoHeight="+trackFormat.getInteger(MediaFormat.KEY_HEIGHT));

            } else if (keyMime.startsWith("audio/")) {
                File outputFile = extractorAndMuxer(mediaExtractor, i, "/audio.aac");

                Log.i(TAG, "extractorAndMuxerMP4: channelCount="+trackFormat.getInteger(MediaFormat.KEY_CHANNEL_COUNT)+" bitRate="+trackFormat.getInteger(MediaFormat.KEY_BIT_RATE));

                tvOut.setText(tvOut.getText().toString() + "\n纯音频路径：" + outputFile.getAbsolutePath());
                tvOut.setVisibility(View.VISIBLE);
            }
        }
    } catch (IOException e) {
        e.printStackTrace();
    } finally {
        mediaExtractor.release();
    }

}

private File extractorAndMuxer(MediaExtractor mediaExtractor, int i, String outputName) throws IOException {
    MediaFormat trackFormat = mediaExtractor.getTrackFormat(i);
    MediaMuxer mediaMuxer;
    mediaExtractor.selectTrack(i);

    File outputFile = new File(getExternalFilesDir(Environment.DIRECTORY_MUSIC).getAbsolutePath() + outputName);
    if (outputFile.exists()) {
        outputFile.delete();
    }
    //1. 构造MediaMuxer
    mediaMuxer = new MediaMuxer(outputFile.getAbsolutePath(), MediaMuxer.OutputFormat.MUXER_OUTPUT_MPEG_4);
    //2. 添加轨道信息 参数为MediaFormat
    mediaMuxer.addTrack(trackFormat);
    //3. 开始合成
    mediaMuxer.start();

    //4. 设置buffer
    ByteBuffer buffer = ByteBuffer.allocate(500 * 1024);
    MediaCodec.BufferInfo bufferInfo = new MediaCodec.BufferInfo();

    //5.通过mediaExtractor.readSampleData读取数据流
    int sampleSize = 0;
    while ((sampleSize = mediaExtractor.readSampleData(buffer, 0)) > 0) {
        bufferInfo.flags = mediaExtractor.getSampleFlags();
        bufferInfo.offset = 0;
        bufferInfo.size = sampleSize;
        bufferInfo.presentationTimeUs = mediaExtractor.getSampleTime();
        int isEOS = bufferInfo.flags & MediaCodec.BUFFER_FLAG_END_OF_STREAM;
        Log.i(TAG, "demuxerMp4:  flags=" + bufferInfo.flags + " size=" + sampleSize + " time=" + bufferInfo.presentationTimeUs + " outputName" + outputName+" isEOS="+isEOS);
        //6. 把通过mediaExtractor解封装的数据通过writeSampleData写入到对应的轨道
        mediaMuxer.writeSampleData(0, buffer, bufferInfo);
        mediaExtractor.advance();
    }
    Log.i(TAG, "extractorAndMuxer: " + outputName + "提取封装完成");

    mediaExtractor.unselectTrack(i);
    //6.关闭
    mediaMuxer.stop();
    mediaMuxer.release();
    return outputFile;
}
```

**2. 把纯音频文件和纯视频文件（封装）合成为视频文件**



```dart
/**
     * 把音轨和视频轨再合成新的视频
     */
private String muxerMp4(String inputAudio , String outPutVideo) {
    File videoFile = new File(getExternalFilesDir(Environment.DIRECTORY_MUSIC), "video.mp4");
    File audioFile = new File(getExternalFilesDir(Environment.DIRECTORY_MUSIC), inputAudio);
    File outputFile = new File(getExternalFilesDir(Environment.DIRECTORY_MUSIC), outPutVideo);

    if (outputFile.exists()) {
        outputFile.delete();
    }
    if (!videoFile.exists()) {
        Toast.makeText(this, "视频源文件不存在", Toast.LENGTH_SHORT).show();
        return "";
    }
    if (!audioFile.exists()) {
        Toast.makeText(this, "音频源文件不存在", Toast.LENGTH_SHORT).show();
        return "";
    }

    MediaExtractor videoExtractor = new MediaExtractor();
    MediaExtractor audioExtractor = new MediaExtractor();

    try {
        MediaMuxer mediaMuxer = new MediaMuxer(outputFile.getAbsolutePath(), MediaMuxer.OutputFormat.MUXER_OUTPUT_MPEG_4);
        int videoTrackIndex = 0;
        int audioTrackIndex = 0;

        //先添加视频轨道
        videoExtractor.setDataSource(videoFile.getAbsolutePath());
        int trackCount = videoExtractor.getTrackCount();
        Log.i(TAG, "muxerToMp4: trackVideoCount=" + trackCount);

        for (int i = 0; i < trackCount; i++) {
            MediaFormat trackFormat = videoExtractor.getTrackFormat(i);
            String mimeType = trackFormat.getString(MediaFormat.KEY_MIME);
            if (TextUtils.isEmpty(mimeType)) {
                continue;
            }
            if (mimeType.startsWith("video/")) {
                videoExtractor.selectTrack(i);

                videoTrackIndex = mediaMuxer.addTrack(trackFormat);
                Log.i(TAG, "muxerToMp4: videoTrackIndex=" + videoTrackIndex);
                break;
            }
        }

        //再添加音频轨道
        audioExtractor.setDataSource(audioFile.getAbsolutePath());
        int trackCountAduio = audioExtractor.getTrackCount();
        Log.i(TAG, "muxerToMp4: trackCountAduio=" + trackCountAduio);
        for (int i = 0; i < trackCountAduio; i++) {
            MediaFormat trackFormat = audioExtractor.getTrackFormat(i);
            String mimeType = trackFormat.getString(MediaFormat.KEY_MIME);
            if (TextUtils.isEmpty(mimeType)) {
                continue;
            }
            if (mimeType.startsWith("audio/")) {
                audioExtractor.selectTrack(i);
                audioTrackIndex = mediaMuxer.addTrack(trackFormat);
                Log.i(TAG, "muxerToMp4: audioTrackIndex=" + audioTrackIndex);
                break;
            }

        }



        //再进行合成
        mediaMuxer.start();

        ByteBuffer byteBuffer = ByteBuffer.allocate(500 * 1024);

        MediaCodec.BufferInfo bufferInfo = new MediaCodec.BufferInfo();
        int sampleSize = 0;

        while ((sampleSize = videoExtractor.readSampleData(byteBuffer, 0)) > 0) {

            bufferInfo.flags = videoExtractor.getSampleFlags();
            bufferInfo.offset = 0;
            bufferInfo.size = sampleSize;
            bufferInfo.presentationTimeUs = videoExtractor.getSampleTime();
            mediaMuxer.writeSampleData(videoTrackIndex, byteBuffer, bufferInfo);
            videoExtractor.advance();
        }

        int audioSampleSize = 0;

        MediaCodec.BufferInfo audioBufferInfo = new MediaCodec.BufferInfo();


        while ((audioSampleSize = audioExtractor.readSampleData(byteBuffer, 0)) > 0) {

            audioBufferInfo.flags = audioExtractor.getSampleFlags();
            audioBufferInfo.offset = 0;
            audioBufferInfo.size = audioSampleSize;
            audioBufferInfo.presentationTimeUs = audioExtractor.getSampleTime();
            mediaMuxer.writeSampleData(audioTrackIndex, byteBuffer, audioBufferInfo);
            audioExtractor.advance();
        }

        //最后释放资源
        videoExtractor.release();
        audioExtractor.release();
        mediaMuxer.stop();
        mediaMuxer.release();

    } catch (IOException e) {
        e.printStackTrace();
        return "";

    }
    return outputFile.getAbsolutePath();

}
```

**3. 替换背景音乐，合成新的视频文件**

其实和第二步一样了，通过传入不同的aac音频源即可，这里需要注意一点，mediamuxer 只支持 aac 格式的，不支持mp3，否则会报如下异常，所以需要先把mp3转为aac。可以采用ffmpeg如下命令截取和转换



```css
java.lang.IllegalStateException: Failed to add the track to the muxer
        at android.media.MediaMuxer.nativeAddTrack(Native Method)
        at android.media.MediaMuxer.addTrack(MediaMuxer.java:638)

—> 添加音轨不是aac格式，而是mp3格式时，在medimuter.addTrack(audioFromat)时会报上述错误，解决方案：把mp3转成aac

ffmpeg -i 输入.mp3 -acodec aac 输出.aac -y
```

## 5.4 遇到的问题

### **5.4.1  在合成写入数据时报 IllegalArgumentException: trackIndex is invalid**



```css
java.lang.IllegalArgumentException: trackIndex is invalid
        at android.media.MediaMuxer.writeSampleData(MediaMuxer.java:669)
        —>  mediaMuxer.writeSampleData(0,buffer,bufferInfo);
原因和方案：  解封装时候输出的trackIndex不对导致，因为不管是纯音轨还是纯视频轨道文件只有一个轨道。
```

### **5.4.2  解码出来的存视频文件的长度比原视频少了，而音频的长度一致。**



```css
和视频源有关系，有的原视频最后几秒只有音频播放画面不动，就是这种情况，刚开时不知道，还以为是什么bug，最后通过ffmpeg直接对原视频进行提取，得到的结果一样。

用ffmpeg命令提取纯视频 对比看下
ffmpeg -i 输入.mp4 -vcodec copy -an 输出.mp4 -y 查看生成的视频也是一样。
说明这个视频中视频流就是比音频流要短。

ffmpeg -i 输入.mp4 -acodec copy -vn 输出.aac -y 查看生成的音频流。和通过medieExtractor和mediamuxter提取的一致。
```

### **5.4.3  mediaExtractor.advance()时报IllegalArgumentException: bufferInfo must specify a valid buffer**

```css
通过查看bufferinfo的信息此时flags和presntationTimeUs都为-1，是advance调用时间不对引起

java.lang.IllegalArgumentException: bufferInfo must specify a valid buffer offset, size and presentation time
        at android.media.MediaMuxer.writeSampleData(MediaMuxer.java:682)
        
解决方案： 先调mediaMuxer.writeSampleData 后再mediaExtractor.advance();
```

### **5.4.4  合成时报如下错误，这个mediaMuxer.start之前没有添加轨道导致（流程不熟导致）**

```java
java.lang.IllegalStateException: Failed to start the muxer
        at android.media.MediaMuxer.nativeStart(Native Method)
        at android.media.MediaMuxer.start(MediaMuxer.java:452)

start之前只是构造了mediaMuxer但没有mediaMuxer.addTrack(trackFormat);
```

### **5.4.5  在把音轨和视频轨道合成新视频时，复用了MediaTractor导致异常**

```java
java.io.IOException: Failed to instantiate extractor.
com.av.mediajourney W/System.err:     at android.media.MediaExtractor.nativeSetDataSource(Native Method)
com.av.mediajourney W/System.err:     at android.media.MediaExtractor.setDataSource(MediaExtractor.java:203)

解决：在把视频轨道的源文件路径通过setDataSource设置到mediaextractor后，再把音轨的源文件setdataSource就报了这个错误
正确的做法是针对每一个源设置一个MediaExtractor，不同共用
```

### **5.4.6  把纯音轨和纯视频轨道合成新视频后，播放视频没有声音 时间是对的，但是没有声音**

猜测 会不会是因为轨道0是视频，轨道1是音频的原因？ 用ffmpeg对比查看了下原视频和合成的视频这点有些差异，尝试调下顺序看下。
—>调整后没有效果。。。继续通过两证ffmpeg -i输出信息定位，发现metaChange不同
—>折腾了半天也没结果，最后通过ffplay来播放合成的视频，一切正常，只能说播放器的原因吧，也可能在合成时设置不全导致部分播放器无法播放，暂时不得结果



### **5.4.7 还出现一种情况是合成后的时长变成了音频加视频的时长总和了**

原因是bufferInfo.presentationTimeUs的值不对导致。
异常实现如下：

```cpp
ByteBuffer byteBuffer = ByteBuffer.allocate(500 * 1024);
//先计算出视频帧间隔时间
long smapleTime = 0;
videoExtractor.readSampleData(byteBuffer, 0);
if (videoExtractor.getSampleFlags() == MediaExtractor.SAMPLE_FLAG_SYNC) {
    videoExtractor.advance();
}
videoExtractor.readSampleData(byteBuffer, 0);
long secondTime = videoExtractor.getSampleTime();
videoExtractor.advance();
long thirdtime = videoExtractor.getSampleTime();
smapleTime = Math.abs(thirdtime - secondTime);
Log.i(TAG, "muxerStart: smapleTime=" + smapleTime);

videoExtractor.unselectTrack(videoTrackIndex);
videoExtractor.selectTrack(videoTrackIndex);

MediaCodec.BufferInfo bufferInfo = new MediaCodec.BufferInfo();
int sampleSize = 0;

while ((sampleSize = videoExtractor.readSampleData(byteBuffer, 0)) > 0) {

    bufferInfo.flags = videoExtractor.getSampleFlags();
    bufferInfo.offset = 0;
    bufferInfo.size = sampleSize;
    bufferInfo.presentationTimeUs += smapleTime;
    //bufferInfo.presentationTimeUs = videoExtractor.getSampleTime();
    mediaMuxer.writeSampleData(videoTrackIndex, byteBuffer, bufferInfo);
    videoExtractor.advance();
}

int audioSampleSize = 0;
ByteBuffer audioByteBuffer = ByteBuffer.allocate(500 * 1024);

MediaCodec.BufferInfo audioBufferInfo = new MediaCodec.BufferInfo();


while ((audioSampleSize = audioExtractor.readSampleData(byteBuffer, 0)) > 0) {

    audioBufferInfo.flags = audioExtractor.getSampleFlags();
    audioBufferInfo.offset = 0;
    audioBufferInfo.size = audioSampleSize;
    audioBufferInfo.presentationTimeUs += smapleTime;

    // audioBufferInfo.presentationTimeUs = videoExtractor.getSampleTime();
    mediaMuxer.writeSampleData(audioTrackIndex, audioByteBuffer, audioBufferInfo);
    audioExtractor.advance();
}
```

这些遇到的问题一部分是对mediaExtractor和mediaMuxer的流程不熟悉导致。而有些需要借助ffpmpeg和ffplay进行协助分析排查。

## 5.5 参考

[Android 音视频学习：使用 MediaExtractor 和 MediaMuxer 解析和封装 mp4 文件]（[https://mp.weixin.qq.com/s/KsOdAnTCQ7B_V7agZ0OwXQ](https://links.jianshu.com/go?to=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FKsOdAnTCQ7B_V7agZ0OwXQ)）
 [Android 视频分离和合成(MediaMuxer和MediaExtractor)]（[https://blog.csdn.net/zhi184816/article/details/52514138](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fzhi184816%2Farticle%2Fdetails%2F52514138)）

## 5.6 收获

1. 了解MediaExtractor和Mediamuxer的作用
2. MediaExtractor和Mediamuxer熟悉API和使用流程
3. 提取音轨和视频轨然后进行再合成或者替换音轨实现换背景音乐
4. 遇到问题的分析解决以及复盘。



