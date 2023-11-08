# 35-FFmpeg + AudioTrack 实现音频解码和播放

## 35.0 目录

1. 音频解码流程
2. 解码音频为pcm
3. 使用AudioTrack播放音频
4. 资料
5. 收获

上一篇我们了解了FFmpeg解码流程、关键函数和结构体，实现了视频解码器。这篇我们来实现下音频的解码器。解码流程和视频的基本一致。FFmpeg解码的音频裸数据是PCM格式，android上播放PCM音频数据可以通过AudioTrack和OpenSL ES来实现。

下面我们下来看下解码的流程

## 35.1 音频解码流程

![img](https:////upload-images.jianshu.io/upload_images/1791669-a2e1602e85c12c20.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

和上一篇的视频解码流程基本一致。需要注意的是音频对音频的重采样，以及不同样本格式的数据的排列方式

### 35.1.1 音频解码流程

1. avformat_open_input 打开媒体文件
2. avformat_find_stream_info 初始化AVFormatContext_
3. 匹配到音频流的index
4. avcodec_find_decoder 根据音频流信息的codec_id找到对应的解码器_
5. avcodec_open2  使用给定的AVCodec初始化AVCodecContext_
6. 初始化输出文件、解码AVPacket和AVFrame结构体
7. 申请重采样SwrContext上下文并进行重采样初始化
8. av_read_frame 开始一帧一帧读取
9. avcodec_send_packet
10. avcodec_receive_frame
11. swr_convert重采样
12. 写入到PCM文件或者使用AudioTrack、OpenSL ES进行播放
13. 释放资源

### 35.1.2 补充知识

音频采样格式



```objectivec
enum AVSampleFormat {
    AV_SAMPLE_FMT_NONE = -1,
    AV_SAMPLE_FMT_U8,          ///< unsigned 8 bits
    AV_SAMPLE_FMT_S16,         ///< signed 16 bits
    AV_SAMPLE_FMT_S32,         ///< signed 32 bits
    AV_SAMPLE_FMT_FLT,         ///< float
    AV_SAMPLE_FMT_DBL,         ///< double

    AV_SAMPLE_FMT_U8P,         ///< unsigned 8 bits, planar
    AV_SAMPLE_FMT_S16P,        ///< signed 16 bits, planar
    AV_SAMPLE_FMT_S32P,        ///< signed 32 bits, planar
    AV_SAMPLE_FMT_FLTP,        ///< float, planar
    AV_SAMPLE_FMT_DBLP,        ///< double, planar
    AV_SAMPLE_FMT_S64,         ///< signed 64 bits
    AV_SAMPLE_FMT_S64P,        ///< signed 64 bits, planar

    AV_SAMPLE_FMT_NB           ///< Number of sample formats. DO NOT USE if linking dynamically
};
```

> 带P和不带P，关系到了AVFrame中的data的数据排列，不带P，则是LRLRLRLRLR排列，带P则是LLLLLRRRRR排列，若是双通道则带P则意味着data[0]全是L，data[1]全是R（注意：这是采样点不是字节），PCM播放器播放的文件需要的是LRLRLRLR的。

## 35.2 解码pcm代码实现

具体实现见代码和详细注释



```cpp
#include <jni.h>
#include <string>
#include <unistd.h>


extern "C" {
    #include "include/libavcodec/avcodec.h"
    #include "include/libavformat/avformat.h"
    #include "include/log.h"
    #include <libswscale/swscale.h>
    #include <libavutil/imgutils.h>
    #include <libswresample/swresample.h>

}



extern "C"
    JNIEXPORT jint JNICALL
    Java_android_spport_mylibrary2_Demo_decodeAudio(JNIEnv *env, jobject thiz, jstring video_path,
                                                    jstring pcm_path) {

    //申请avFormatContext空间，记得要释放
    AVFormatContext *pFormatContext = avformat_alloc_context();

    const char *url = env->GetStringUTFChars(video_path, 0);

    //1. 打开媒体文件
    int result = avformat_open_input(&pFormatContext, url, NULL, NULL);
    if (result != 0) {
        LOGE("open input error url =%s,result=%d", url, result);
        return -1;
    }
    //2.读取媒体文件信息，给avFormatContext赋值
    result = avformat_find_stream_info(pFormatContext, NULL);
    if (result < 0) {
        LOGE("open input avformat_find_stream_info,result=%d", result);
        return -1;
    }
    ////3. 匹配到音频流的index
    int audioIndex = -1;
    for (int i = 0; i < pFormatContext->nb_streams; ++i) {
        AVMediaType codecType = pFormatContext->streams[i]->codecpar->codec_type;
        if (AVMEDIA_TYPE_AUDIO == codecType) {
            audioIndex = i;
            break;
        }
    }
    if (audioIndex == -1) {
        LOGE("not find a audio stream");
        return -1;
    }

    AVCodecParameters *pCodecParameters = pFormatContext->streams[audioIndex]->codecpar;

    //4. 根据流信息的codec_id找到对应的解码器
    AVCodec *pCodec = avcodec_find_decoder(pCodecParameters->codec_id);

    if (pCodec == NULL) {
        LOGE("Couldn`t find Codec");
        return -1;
    }

    AVCodecContext *pCodecContext = pFormatContext->streams[audioIndex]->codec;

    //5.使用给定的AVCodec初始化AVCodecContext
    int openResult = avcodec_open2(pCodecContext, pCodec, NULL);
    if (openResult < 0) {
        LOGE("avcodec open2 result %d", openResult);
        return -1;
    }

    const char *pcmPathStr = env->GetStringUTFChars(pcm_path, NULL);

    //新建一个二进制文件，已存在的文件将内容清空，允许读写
    FILE *pcmFile = fopen(pcmPathStr, "wb+");
    if (pcmFile == NULL) {
        LOGE(" fopen outPut file error");
        return -1;
    }

    //6. 初始化输出文件、解码AVPacket和AVFrame结构体
    auto *packet = (AVPacket *) av_malloc(sizeof(AVPacket));

    AVFrame *pFrame = av_frame_alloc();

    //7. 申请重采样SwrContext上下文
    SwrContext *swrContext = swr_alloc();

    int numBytes = 0;
    uint8_t *outData[2] = {0};
    int dstNbSamples = 0;                           // 解码目标的采样率

    int outChannel = 2;                             // 重采样后输出的通道
    //带P和不带P，关系到了AVFrame中的data的数据排列，不带P，则是LRLRLRLRLR排列，带P则是LLLLLRRRRR排列，
    // 若是双通道则带P则意味着data[0]全是L，data[1]全是R（注意：这是采样点不是字节），PCM播放器播放的文件需要的是LRLRLRLR的。
    //P表示Planar（平面），其数据格式排列方式为 (特别记住，该处是以点nb_samples采样点来交错，不是以字节交错）:
    //                    LLLLLLRRRRRRLLLLLLRRRRRRLLLLLLRRRRRRL...（每个LLLLLLRRRRRR为一个音频帧）
    //                    而不带P的数据格式（即交错排列）排列方式为：
    //                    LRLRLRLRLRLRLRLRLRLRLRLRLRLRLRLRLRLRL...（每个LR为一个音频样本）

    AVSampleFormat outFormat = AV_SAMPLE_FMT_S16P;  // 重采样后输出的格式
    int outSampleRate = 44100;                          // 重采样后输出的采样率

    // 通道布局与通道数据的枚举值是不同的，需要av_get_default_channel_layout转换
    swrContext = swr_alloc_set_opts(0,                                 // 输入为空，则会分配
                                    av_get_default_channel_layout(outChannel),
                                    outFormat,                         // 输出的采样频率
                                    outSampleRate,                     // 输出的格式
                                    av_get_default_channel_layout(pCodecContext->channels),
                                    pCodecContext->sample_fmt,       // 输入的格式
                                    pCodecContext->sample_rate,      // 输入的采样率
                                    0,
                                    0);

    //重采样初始化
    int swrInit = swr_init(swrContext);
    if (swrInit < 0) {
        LOGE("swr init error swrInit=%d", swrInit);
        return -1;
    }

    auto *outPcmBuffer = (uint8_t *) av_malloc(AVCODEC_MAX_AUDIO_FRAME_SIZE);

    int frame_cnt = 0;

    outData[0] = (uint8_t *) av_malloc(1152 * 8);
    outData[1] = (uint8_t *) av_malloc(1152 * 8);

    //8. 开始一帧一帧读取
    while (av_read_frame(pFormatContext, packet) >= 0) {
        if (packet->stream_index == audioIndex) {
            //9。将封装包发往解码器
            int ret = avcodec_send_packet(pCodecContext, packet);
            if (ret) {
                LOGE("Failed to avcodec_send_packet(pAVCodecContext, pAVPacket) ,ret =%d", ret);
                break;
            }
            //            LOGI("av_read_frame");
            // 10. 从解码器循环拿取数据帧
            while (!avcodec_receive_frame(pCodecContext, pFrame)) {
                // nb_samples并不是每个包都相同，遇见过第一个包为47，第二个包开始为1152的

                // 获取每个采样点的字节大小
                numBytes = av_get_bytes_per_sample(outFormat);
                //修改采样率参数后，需要重新获取采样点的样本个数
                dstNbSamples = av_rescale_rnd(pFrame->nb_samples,
                                              outSampleRate,
                                              pCodecContext->sample_rate,
                                              AV_ROUND_ZERO);
                // 重采样
                swr_convert(swrContext,
                            outData,
                            dstNbSamples,
                            (const uint8_t **) pFrame->data,
                            pFrame->nb_samples);
                LOGI("avcodec_receive_frame");
                // 第一次显示
                static bool show = true;
                if (show) {
                    LOGE("numBytes pFrame->nb_samples=%d dstNbSamples=%d,numBytes=%d,pCodecContext->sample_rate=%d,outSampleRate=%d", pFrame->nb_samples,
                         dstNbSamples,numBytes,pCodecContext->sample_rate,outSampleRate);
                    show = false;
                }
                // 使用LRLRLRLRLRL（采样点为单位，采样点有几个字节，交替存储到文件，可使用pcm播放器播放）
                for (int index = 0; index < dstNbSamples; index++) {
                    // // 交错的方式写入, 大部分float的格式输出 符合LRLRLRLR点交错模式
                    for (int channel = 0;channel < pCodecContext->channels; channel++)
                    {
                        fwrite((char *) outData[channel] + numBytes * index, 1, numBytes, pcmFile);
                    }
                }
                av_packet_unref(packet);
            }
            frame_cnt++;
        }
    }

    LOGI("frame count is %d", frame_cnt);

    swr_free(&swrContext);
    av_free(outPcmBuffer);
    avcodec_close(pCodecContext);
    avformat_close_input(&pFormatContext);

    env->ReleaseStringUTFChars(video_path, url);

    env->ReleaseStringUTFChars(pcm_path, pcmPathStr);

    return 0;
}
```

## 35.3 使用AudioTrack播放PCM音频

这一小节我们再上一小节解码输出PCM音频数据的基础上，再Native层调用Java层的AudioTrack进行完成音频的播放。

在[音视频开发之旅（三）AudioTrack播放PCM音频](https://links.jianshu.com/go?to=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FYaw3QJP5XZ2Uwc8N3-UXxQ)我们已经学习实践过，我们简单回顾下。



```objectivec
public AudioTrack(int streamType, int sampleRateInHz, int channelConfig, int audioFormat,
                  int bufferSizeInBytes, int mode)
    其中采样率sampleRateInHz、声道数channelConfig、音频格式audioFormat以及音频缓冲区大小bufferSizeInBytes 

    来看参数streamType以及mode

    streamType音频流的类型，有如下几种
    AudioManager#STREAM_VOICE_CALL：电话声音AudioManager#STREAM_SYSTEM：系统声音
    AudioManager#STREAM_RING：铃声
    AudioManager#STREAM_MUSIC：音乐声
    AudioManager#STREAM_ALARM：闹铃声
    AudioManager#STREAM_NOTIFICATION：通知声

    这里我们使用的是AudioManager#STREAM_MUSIC。

    下面我们重点看下mode
@param mode streaming or static buffer.
    MODE_STATIC and MODE_STREAM

    STATIC模式：一次性将所有的数据放到一个固定的buffer，然后直接传送给AudioTrack，简单有效，通常应用于播放铃声或者系统提示音等，占用内存较少的音频数据

    STREAM模式：一次一次的将音频数据流写入到AudioTrack对象中，并持续处于阻塞状态，当数据从Java层到Native层执行播放完毕后才返回，这种方式可以避免由于音频过大导致内存占用过多。当然对应的不足就是总是在java和native层进行交互，并且阻塞直到播放完毕，效率损失较大。
```

我们这里使用STREAM模式相关的方法类如下



```java
package android.spport.mylibrary2;

import android.media.AudioFormat;
import android.media.AudioManager;
import android.media.AudioTrack;
import android.util.Log;

import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;

public class AudioTrackStreamHelper {

    private static final String TAG = "AudioTrackStreamHelper";
    private AudioTrack audioTrack;
    private int sampleRateInHz;
    private int channels;
    private int audioFormat;
    private int bufferSize;
    private int mode = -1;

    private boolean hasPcmFile = false;
    private File pcmFile;
    private Thread audioTrackThread;


    public void initAudioTrackParams(String path) {
        sampleRateInHz = 44100;
        channels = AudioFormat.CHANNEL_OUT_STEREO;
        audioFormat = AudioFormat.ENCODING_PCM_16BIT;
        bufferSize = AudioTrack.getMinBufferSize(sampleRateInHz, channels, audioFormat);

        pcmFile = new File(path);//"raw.pcm"
        if (pcmFile.exists()) {
            hasPcmFile = true;
        }
    }

    private int initAudioTrackWithMode(int mode, int bufferSize) {
        if (audioTrack != null) {
            audioTrack.release();
            audioTrack.setPlaybackPositionUpdateListener(null);
            audioTrack = null;
        }

        audioTrack = new AudioTrack(AudioManager.STREAM_MUSIC, sampleRateInHz, channels, audioFormat, bufferSize, mode);
        if (audioTrack != null) {
            Log.i(TAG, "initAudioTrackWithMode: state="+audioTrack.getState()+" playState="+audioTrack.getPlayState());
            return audioTrack.getState();
        }
        return AudioTrack.STATE_UNINITIALIZED;
    }

    public boolean isHasPcmFile() {
        return hasPcmFile;
    }

    public void play() {
        releaseAudioTrack();

        int state = initAudioTrackWithMode(AudioTrack.MODE_STREAM, bufferSize);
        if (state == AudioTrack.STATE_UNINITIALIZED) {
            Log.e(TAG, "run: state is uninit");
            return;
        }

        audioTrackThread = new Thread(new Runnable() {
            @Override
            public void run() {
                FileInputStream fileInputStream = null;
                try {
                    fileInputStream = new FileInputStream(pcmFile);
                    byte[] buffer = new byte[bufferSize / 2];
                    int readCount;
                    Log.d(TAG, "run: ThreadId=" + Thread.currentThread() + " playState=" + audioTrack.getPlayState());
                    //stream模式，可以先调用play
                    audioTrack.play();
                    while (fileInputStream.available() > 0) {
                        readCount = fileInputStream.read(buffer);
                        if (readCount == AudioTrack.ERROR_BAD_VALUE || readCount == AudioTrack.ERROR_INVALID_OPERATION) {
                            continue;
                        }
                        if (audioTrack == null) {
                            return;
                        } else {
                            Log.i(TAG, "run: audioTrack.getState()" + audioTrack.getState() + " audioTrack.getPlayState()=" + audioTrack.getPlayState());
                        }
//                        audioTrack.getPlayState()
                        //一次一次的写入pcm数据到audioTrack.由于是在子线程中进行write，快速连续点击可能主线程触发了stop或者release，导致子线程write异常：IllegalStateException: Unable to retrieve AudioTrack pointer for write()
                        //所以加playstate的判断
                        if (readCount > 0 && audioTrack != null && audioTrack.getPlayState() == AudioTrack.PLAYSTATE_PLAYING && audioTrack.getState() == AudioTrack.STATE_INITIALIZED) {
                            audioTrack.write(buffer, 0, readCount);
                        }
                    }

                } catch (IOException | IllegalStateException e) {
                    e.printStackTrace();
                    Log.e(TAG, "play: " + e.getMessage());
                } finally {
                    if (fileInputStream != null) {
                        try {
                            fileInputStream.close();
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                    }
                    Log.d(TAG, "playWithStreamMode: end  ThreadID=" + Thread.currentThread());
                }
            }
        });
        audioTrackThread.start();
    }


    public void pausePlay() {
        if (audioTrack != null) {
            if (audioTrack.getState() > AudioTrack.STATE_UNINITIALIZED) {
                audioTrack.pause();
            }
            Log.d(TAG, "pausePlay: isPlaying false getPlayState= " + audioTrack.getPlayState());
        }
        if (audioTrackThread != null) {
            audioTrackThread.interrupt();
        }
    }

    private void releaseAudioTrack() {
        if (audioTrack != null && audioTrack.getState() == AudioTrack.STATE_INITIALIZED) {
            audioTrack.stop();
            audioTrack.release();
            Log.d(TAG, "pausePlay: isPlaying false");
        }
        if (audioTrackThread != null) {
            audioTrackThread.interrupt();
        }
    }

    public void destroy() {
        if (audioTrack != null) {
            audioTrack.release();
            audioTrack = null;
        }
        if (audioTrackThread != null) {
            audioTrackThread.interrupt();
            audioTrackThread = null;
        }
    }
}
```

由于是Java代码，可以在java层在直接调用，省去了JNI的消耗。



```java
public class MainActivity extends AppCompatActivity {


    private Demo demo;
    AudioTrackStaticModeHelper audioTrackHelper;
    AudioTrackStreamHelper audioTrackStreamHelper;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // Example of a call to a native method
        TextView tv = findViewById(R.id.sample_text);

        checkPermission();

        demo = new Demo();
        tv.setText(demo.stringFromJNI());
        String folderurl= Environment.getExternalStorageDirectory().getPath();
        File externalFilesDir = getExternalFilesDir(null);
        Log.i("MainActivity", "externalFilesDir: "+externalFilesDir);

//        demo.decodeVideo(folderurl+"/input.mp4", externalFilesDir+"/output7.yuv");

        demo.decodeAudio(folderurl+"/input.mp4", externalFilesDir+"/audio.pcm");

        initAudioTrackStreamMode(externalFilesDir);

    }

    private void initAudioTrackStreamMode(File externalFilesDir) {
        audioTrackStreamHelper = new AudioTrackStreamHelper();

        audioTrackStreamHelper.initAudioTrackParams(externalFilesDir+"/audio.pcm");
        audioTrackStreamHelper.play();
    }
}
```

由于我们FFmpeg解码时同步的，所以可以采用这种方式，但是解码本事是耗时操作，应该创建解码线程，然后播放PCM时也可以直接送给AudioTrack进行播放，而不用先写入到PCM文件再设置播放。这些都是可优化点。我们在后续音视频同步时再进行优化。

代码已上传至[github](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fayyb1988%2Fffmpegvideodecodedemo) [https://github.com/ayyb1988/ffmpegvideodecodedemo](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fayyb1988%2Fffmpegvideodecodedemo)
 欢迎交流，一起学习成长。

## 35.4 资料

1. 《音视频开发进阶》
2. [ffmpeg主体架构分析](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.cnblogs.com%2Ftocy%2Fp%2Fffmpeg-framework-analysis.html)
3. [FFmpeg开发笔记（七）：ffmpeg解码音频保存为PCM并使用软件播放](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fqq21497936%2Farticle%2Fdetails%2F108799279)
4. [Android NDK开发之旅35--FFmpeg+AudioTrack音频播放](https://www.jianshu.com/p/b7b6f755536c)
5. [音视频开发之旅（三）AudioTrack播放PCM音频](https://links.jianshu.com/go?to=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FYaw3QJP5XZ2Uwc8N3-UXxQ)

## 35.5 收获

1. 了解音频解码流程
2. 实现音频解码
3. 解决由于没有重采样以及采样输出格式不对导致音频播放声音异常问题
4. 使用AudioTrack的STRAM模式对解码后的PCM进行播放