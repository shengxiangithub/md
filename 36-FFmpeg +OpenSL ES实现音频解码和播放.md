# 36-FFmpeg +OpenSL ES实现音频解码和播放

## 36.0 目录

1. OpenSL ES基本介绍
2. OpenSL ES播放音频流程
3. 代码实现
4. 遇到的问题
5. 资料
6. 收获

上一篇我们通过AudioTrack实现了FFmpeg解码后的PCM音频数据的播放，在Android上还有一种播放音频的方式即OpenSL ES, 什么是OpenSL ES，这个我们平时接触的很少，原因是平时业务中大部分播放可以通过Java层的MediaPlayer或者AudioTrack实现音频播放。如果遇到一些特殊的需求，比如添加音效等这是不容易实现。而OpenSL可以很好的解决此类问题，并且还有很多丰富的功能。下面我们一起来学习实践吧。

## 36.1 OpenSL ES基本介绍

### 36.1.1 OPenSL ES 是什么？

> OpenSL ES (Open Sound Library for Embedded System) ,即嵌入式音频加速标准与 Android Java 框架中的 MediaPlayer 和 MediaRecorderAPI 提供类似的音频功能。OpenSL ES 提供 C 语言接口和 CPP 绑定，让您可以从使用任意一种语言编写的代码中调用 API。
>  相对MediaPlayer 和 MediaRecorderAPI 等java层API来说，OpenSL ES 则是比价低层级的 API, 属于 C 语言 API 。在开发中，一般会直接使用高级 API , 除非遇到性能瓶颈，如语音实时聊天、3D Audio 、某些 Effects 等，开发者可以直接通过 C/CPP开发基于 OpenSL ES 音频的应用, 提升应用的音频性能。

### 36.1.2 OpenSL ES有哪些能力呐？

我们通过下图的OpenSL ES使用指南中可以看到支持，音频的播放、混音、音效、以及录制等功能。



![img](https:////upload-images.jianshu.io/upload_images/1791669-3e527c11f82e6cf1.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

![img](https:////upload-images.jianshu.io/upload_images/1791669-529a6086c2e95f4c.png?imageMogr2/auto-orient/strip|imageView2/2/w/548/format/webp)

上述两种图片来自：[官方指南：OpenSL ES](https://links.jianshu.com/go?to=https%3A%2F%2Fdeveloper.android.google.cn%2Fndk%2Fguides%2Faudio%2Fopensl)

### 36.1.3 如何引入？

[OpenSL ES 编程说明](https://links.jianshu.com/go?to=https%3A%2F%2Fdeveloper.android.google.cn%2Fndk%2Fguides%2Faudio%2Fopensl%2Fopensl-prog-notes)

OpenSL ES的库我们可以在NDK 软件包中找到



```bash
eg： $NDK_PATH_/platforms/android-30/arch-arm/usr/lib/libOpenSLES.so
```

引入方式只需要在CmakeList.txt的target_link_libraries中加入**OpenSLES**即可



```bash
target_link_libraries( 
        native-lib
        avformat
        avcodec
        avfilter
        avutil
        swresample
        swscale
        OpenSLES

        ${log-lib})
```

### 36.1.4  对象与接口

OpenES SL虽然是面向过程的C语言编写的，但是以面向对象的思想提供了对象和接口，方便开发的在项目中使用。

> OpenSL ES 对象类似于 Java 和 CPP 等编程语言中的对象概念，不过 OpenSL ES 对象仅能通过其关联接口进行访问。其中包括所有对象的初始接口，称为 SLObjectItf。对象本身没有句柄，只有一个连接到对象的 SLObjectItf 接口的句柄。
>  需要注意的是 OpenSL ES 对象不能直接使用，必须通过其 GetInterface 函数用ID号拿到指定接口（如播放器的播放接口），然后通过该接口来访问功能函数

OpenSL ES 对象是先创建的，它会返回 SLObjectItf，然后再实现 (realize),然后使用 GetInterface，为其需要的每种功能获取接口
 音频播放会用到 引擎、混音器以及播放器对象和接口，下一小节我们来看下具体流程。

## 36.2 OpenSL ES播放音频流程

![img](https:////upload-images.jianshu.io/upload_images/1791669-c1b0bcd0b433aabc.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

图片来源：[ OpenSL-ES 官方文档](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.khronos.org%2Fregistry%2FOpenSL-ES%2F)

在CmakeList引入OpenSL库，然后在对应的CPP文件中导入相应的头文件即可使用OpenSL ES，具体流程如下

1. 创建引擎对象`SLObjectItf engineObj`
    初始化引擎 `Realize`
    获取引擎接口 `GetInterface SLEngineItf`
2. 创建混音器对象`SLObjectItf outputMixObj`
    初始化混音器 `Realize`
3. 设置输入输出数据参数
4. 创建播放器对象 `SLPlayItf playerObj`
    初始化播放器`Realize`
    获取播放器接口 `GetInterface`
5. 获取播放回调接口(即缓冲队列)`SLAndroidSimpleBufferQueueItf bufferQueue`
6. 注册播放回调 `RegisterCallback
7. 设置播放状态`SetPlayState`
8. 等待音频帧加入队列触发播放回调`(*mBufferQueue)->Enqueue`
9. 释放资源

具体参考官方提供的示例demo [native-audio 是一个简单的音频录制器/播放器 ](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fandroid%2Fndk-samples%2Ftree%2Fmain%2Fnative-audio)

## 36.3 OpenSL ES播放解码PCM的代码实现

了解了OpenSL ES的基本知识和使用流程，下面我们开始具体的代码实现。



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
#include <SLES/OpenSLES.h>
#include <SLES/OpenSLES_Android.h>
}

//函数声明
jint playPcmBySL(JNIEnv *env,  jstring pcm_path);

extern "C"
JNIEXPORT jint JNICALL
Java_android_spport_mylibrary2_Demo_decodeAudio(JNIEnv *env, jobject thiz, jstring video_path,
                                                jstring pcm_path) {

....
//在音频解码完成后调用使用sl播放的函数
 playPcmBySL(env,pcm_path);
}

// engine interfaces
static SLObjectItf engineObject = NULL;
static SLEngineItf engineEngine;

// output mix interfaces
static SLObjectItf outputMixObject = NULL;
static SLEnvironmentalReverbItf outputMixEnvironmentalReverb = NULL;

static SLObjectItf pcmPlayerObject = NULL;
static SLPlayItf pcmPlayerPlay;
static SLAndroidSimpleBufferQueueItf pcmBufferQueue;

FILE *pcmFile;
void *buffer;
uint8_t *out_buffer;


jint playPcmBySL(JNIEnv *env, const _jstring *pcm_path);

// aux effect on the output mix, used by the buffer queue player
static const SLEnvironmentalReverbSettings reverbSettings = SL_I3DL2_ENVIRONMENT_PRESET_STONECORRIDOR;

//播放回调
void playerCallback(SLAndroidSimpleBufferQueueItf bufferQueueItf, void *context) {


    if (bufferQueueItf != pcmBufferQueue) {
        LOGE("SLAndroidSimpleBufferQueueItf is not equal");
        return;
    }

    while (!feof(pcmFile)) {
        size_t size = fread(out_buffer, 44100 * 2 * 2, 1, pcmFile);
        if (out_buffer == NULL || size == 0) {
            LOGI("read end %ld", size);
        } else {
            LOGI("reading %ld", size);
        }
        buffer = out_buffer;
        break;
    }
    if (buffer != NULL) {
        LOGI("buffer is not null");
        SLresult result = (*pcmBufferQueue)->Enqueue(pcmBufferQueue, buffer, 44100 * 2 * 2);
        if (SL_RESULT_SUCCESS != result) {
            LOGE("pcmBufferQueue error %d",result);
        }
    }

}



jint playPcmBySL(JNIEnv *env,  jstring pcm_path) {
    const char *pcmPath = env->GetStringUTFChars(pcm_path, NULL);
    pcmFile = fopen(pcmPath, "r");
    if (pcmFile == NULL) {
        LOGE("open pcmfile error");
        return -1;
    }
    out_buffer = (uint8_t *) malloc(44100 * 2 * 2);

    //1. 创建引擎`
//    SLresult result;
//1.1 创建引擎对象
    SLresult result = slCreateEngine(&engineObject, 0, 0, 0, 0, 0);
    if (SL_RESULT_SUCCESS != result) {
        LOGE("slCreateEngine error %d", result);
        return -1;
    }
    //1.2 实例化引擎
    result = (*engineObject)->Realize(engineObject, SL_BOOLEAN_FALSE);
    if (SL_RESULT_SUCCESS != result) {
        LOGE("Realize engineObject error");
        return -1;
    }
    //1.3获取引擎接口SLEngineItf
    result = (*engineObject)->GetInterface(engineObject, SL_IID_ENGINE, &engineEngine);
    if (SL_RESULT_SUCCESS != result) {
        LOGE("GetInterface SLEngineItf error");
        return -1;
    }
    slCreateEngine(&engineObject, 0, 0, 0, 0, 0);
    (*engineObject)->Realize(engineObject, SL_BOOLEAN_FALSE);
    (*engineObject)->GetInterface(engineObject, SL_IID_ENGINE, &engineEngine);

    //获取到SLEngineItf接口后，后续的混音器和播放器的创建都会使用它

    //2. 创建输出混音器

    const SLInterfaceID ids[1] = {SL_IID_ENVIRONMENTALREVERB};
    const SLboolean req[1] = {SL_BOOLEAN_FALSE};

    //2.1 创建混音器对象
    result = (*engineEngine)->CreateOutputMix(engineEngine, &outputMixObject, 1, ids, req);
    if (SL_RESULT_SUCCESS != result) {
        LOGE("CreateOutputMix  error");
        return -1;
    }
    //2.2 实例化混音器
    result = (*outputMixObject)->Realize(outputMixObject, SL_BOOLEAN_FALSE);
    if (SL_RESULT_SUCCESS != result) {
        LOGE("outputMixObject Realize error");
        return -1;
    }
    //2.3 获取混音接口 SLEnvironmentalReverbItf
    result = (*outputMixObject)->GetInterface(outputMixObject, SL_IID_ENVIRONMENTALREVERB,
                                           &outputMixEnvironmentalReverb);
    if (SL_RESULT_SUCCESS == result) {
        result = (*outputMixEnvironmentalReverb)->SetEnvironmentalReverbProperties(
                outputMixEnvironmentalReverb, &reverbSettings);
    }


    //3 设置输入输出数据源
//setSLData();
//3.1 设置输入 SLDataSource
    SLDataLocator_AndroidSimpleBufferQueue loc_bufq = {SL_DATALOCATOR_ANDROIDSIMPLEBUFFERQUEUE,2};

    SLDataFormat_PCM formatPcm = {
            SL_DATAFORMAT_PCM,//播放pcm格式的数据
            2,//2个声道（立体声）
            SL_SAMPLINGRATE_44_1,//44100hz的频率
            SL_PCMSAMPLEFORMAT_FIXED_16,//位数 16位
            SL_PCMSAMPLEFORMAT_FIXED_16,//和位数一致就行
            SL_SPEAKER_FRONT_LEFT | SL_SPEAKER_FRONT_RIGHT,//立体声（前左前右）
            SL_BYTEORDER_LITTLEENDIAN//结束标志
    };

    SLDataSource slDataSource = {&loc_bufq, &formatPcm};

    //3.2 设置输出 SLDataSink
    SLDataLocator_OutputMix loc_outmix = {SL_DATALOCATOR_OUTPUTMIX, outputMixObject};
    SLDataSink audioSnk = {&loc_outmix, NULL};


    //4.创建音频播放器

    //4.1 创建音频播放器对象

    const SLInterfaceID ids2[1] = {SL_IID_BUFFERQUEUE};
    const SLboolean req2[1] = {SL_BOOLEAN_TRUE};

    result = (*engineEngine)->CreateAudioPlayer(engineEngine, &pcmPlayerObject, &slDataSource, &audioSnk,
                                                1, ids2, req2);
    if (SL_RESULT_SUCCESS != result) {
        LOGE(" CreateAudioPlayer error");
        return -1;
    }

    //4.2 实例化音频播放器对象
    result = (*pcmPlayerObject)->Realize(pcmPlayerObject, SL_BOOLEAN_FALSE);
    if (SL_RESULT_SUCCESS != result) {
        LOGE(" pcmPlayerObject Realize error");
        return -1;
    }
    //4.3 获取音频播放器接口
    result = (*pcmPlayerObject)->GetInterface(pcmPlayerObject, SL_IID_PLAY, &pcmPlayerPlay);
    if (SL_RESULT_SUCCESS != result) {
        LOGE(" SLPlayItf GetInterface error");
        return -1;
    }

    //5. 注册播放器buffer回调 RegisterCallback

    //5.1  获取音频播放的buffer接口 SLAndroidSimpleBufferQueueItf
    result = (*pcmPlayerObject)->GetInterface(pcmPlayerObject, SL_IID_BUFFERQUEUE, &pcmBufferQueue);
    if (SL_RESULT_SUCCESS != result) {
        LOGE(" SLAndroidSimpleBufferQueueItf GetInterface error");
        return -1;
    }
    //5.2 注册回调 RegisterCallback
    result = (*pcmBufferQueue)->RegisterCallback(pcmBufferQueue, playerCallback, NULL);
    if (SL_RESULT_SUCCESS != result) {
        LOGE(" SLAndroidSimpleBufferQueueItf RegisterCallback error");
        return -1;
    }

    //6. 设置播放状态为Playing
    result = (*pcmPlayerPlay)->SetPlayState(pcmPlayerPlay, SL_PLAYSTATE_PLAYING);
    if (SL_RESULT_SUCCESS != result) {
        LOGE(" SetPlayState  error");
        return -1;
    }

    //7.触发回调
    playerCallback(pcmBufferQueue,NULL);

    return 0;
}
```

OpenSL ES 还有更多丰富的功能，比如，混音、设置音量、录音、播放url或者assert中的音频。详细了解可以查看官方文档和NDK的demo，

本篇就学习实践到这里，**越学习发下身边优秀的人越多，自己不会的东西、要学习的就越多，抓住一个核心痛点，一起学习实践吧。**

代码已上传至github。[[https://github.com/ayyb1988/ffmpegvideodecodedemo\]](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fayyb1988%2Fffmpegvideodecodedemo%5D) 欢迎交流，一起学习成长。

## 36.4 遇到的问题

**问题1: 拿到混音接口对象后没有SetEnvironmentalReverbProperties设置后result不为0导致家了为0判断，导致这里一直提示出错。**
 解决方案，去掉此处的result检查，官方的demo也返回一样的值16



```php
   result = (*outputMixObject)->GetInterface(outputMixObject, SL_IID_ENVIRONMENTALREVERB,
                                           &outputMixEnvironmentalReverb);
    if (SL_RESULT_SUCCESS == result) {
        result = (*outputMixEnvironmentalReverb)->SetEnvironmentalReverbProperties(
                outputMixEnvironmentalReverb, &reverbSettings);
        if (SL_RESULT_SUCCESS != result) {
            LOGE(" SetEnvironmentalReverbProperties error");
            return -1;
        }
    }

改为如下：
result = (*outputMixEnvironmentalReverb)->SetEnvironmentalReverbProperties(
                outputMixEnvironmentalReverb, &reverbSettings);
```

**问题2: 创建播放器对象一直为空，导致无法播放**

原因：给SLData 设置数据源时
 SL_DATALOCATOR_ANDROIDSIMPLEBUFFERQUEUE错误的写成了SL_DATALOCATOR_ANDROIDBUFFERQUEUE



```rust
    SLDataLocator_AndroidSimpleBufferQueue loc_bufq =      {SL_DATALOCATOR_ANDROIDBUFFERQUEUE, 2};
  
-->改为

  SLDataLocator_AndroidSimpleBufferQueue loc_bufq = {SL_DATALOCATOR_ANDROIDSIMPLEBUFFERQUEUE,2};
```

**问题3. 播放音频时音频卡住不断重复**



```cpp
    while (!feof(pcmFile)) {
        size_t size = fread(out_buffer, 44100 * 2 * 2, 1, pcmFile);
        if (out_buffer == NULL || size == 0) {
            LOGI("read end %ld", size);
        } else {
            LOGI("reading %ld", size);
        }
        buffer = out_buffer;
        //原因是，忘记跳出循环了
        break;
    }
```

在学习的初期一个小错误就可能折腾几个小时，在采用逐步排查流程和查看细节、以及和可运行的demo进行对比分析排查出问题所在。
 根源还在于不够细心和理解的不透彻。

## 36.5 资料

1. [ OpenSL-ES 官方文档](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.khronos.org%2Fregistry%2FOpenSL-ES%2F)
2. [NDK指南: OpenSL ES](https://links.jianshu.com/go?to=https%3A%2F%2Fdeveloper.android.google.cn%2Fndk%2Fguides%2Faudio%2Fopensl)
3. [NDK指南demo：native-audio 是一个简单的音频录制器/播放器 ](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fandroid%2Fndk-samples%2Ftree%2Fmain%2Fnative-audio)
4. [音视频学习 (七) AudioTrack、OpenSL ES 音频渲染](https://links.jianshu.com/go?to=http%3A%2F%2Fwww.zyiz.net%2Ftech%2Fdetail-104896.html)
5. [FFmpeg 开发(03)：FFmpeg + OpenSL ES 实现音频解码播放](https://links.jianshu.com/go?to=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FkWBuhjwvBF0NWvAB9P8q9Q)
6. [android平台OpenSL ES播放PCM数据](https://www.jianshu.com/p/cccb59466e99)
7. [Android通过OpenSL ES播放音频套路详解](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fywl5320%2Farticle%2Fdetails%2F78503768)

## 36.6 收获

1. 了解了OpenSl ES的基本知识和播放音频数据的流程
2. 代码实现OpenSL ES播放音频流
3. 和FFmpeg结合，实现opensl播放解码后的音频数据
4. 解决遇到的问题