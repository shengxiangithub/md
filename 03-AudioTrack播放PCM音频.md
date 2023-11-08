# 03-AudioTrack播放PCM音频

 [腾讯课堂 零声教育](https://0voice.ke.qq.com) 整理，  版权归原作者所有，如有侵权请联系删除，原文链接：[音视频开发之旅（三）AudioTrack播放PCM音频 - 简书 (jianshu.com)](https://www.jianshu.com/p/d83a5a40cf8a)

## 3.0 目录

1. AudioTrack和MediaPlayer
2. AudioTrack的API介绍（构造、操作、状态机）
3. 具体实现（Static和Stream两种模式）
4. 遇到的问题
5. 收获

## 3.1 MediaPlayer和AudioTrack

Android SDK 中提供了三种播放声音的API，常见的是MediaPlayer和AudioTrack

- 其中AudioTrack管理、播放单一音频资源。可以将PCM音频数据传输到音频接收器，以供播放，只能播放源码流即PCM，wav封装格式的音频也可以用AudioTrack播放，但是wav头部分在播放解析是会发出噪音。
- 而MediaPlayer可以播放多种格式的音频文件，比如 mp3 aac等，因为MediaPlayer会在framework层创建对应的音频解码器。

既然MediaPlayer可以播放那么多的音频格式，为什么我们还要学习AudioTrack呐？
首先MediaPlayer在framwork层还是会创建AudioTrack，把解码后的PCM流传递给AudioTrack，再传递给AudioFliger进行混音播放。

每一个音频流对应一个AudioTrack，AudioTrack会在创建时注册到AudioFlinger中，AudioFlinger把所有的AudioTrack进行混合Mixer，然后输送到AudioHardware进行播放。

Android最多可以同时创建32个音频流。在短视频编辑等应用领域，**对视频进行添加配乐进行编辑，需要把视频中的音轨和配乐中的音轨进行解码PCM进行混合再编码**。**再或者我们在剪映等视频编辑app可以添加多个音轨**，就想Audition一样强大。这些都都需要我们对AudioTrack有一定的了解掌握。

## 3.2 AudioTrack的介绍

我们先简单看下AudioTrack提供了哪些API

### 3.2.1. 构造方法

```java
public AudioTrack(int streamType, int sampleRateInHz, int channelConfig, int audioFormat,
 int bufferSizeInBytes, int mode)
```

 其中采样率sampleRateInHz、声道数channelConfig、音频格式audioFormat以及音频缓冲区大小bufferSizeInBytes 这四个概念和上一篇《AudioRecord录制PCM音频》中的介绍的AudioRecord的构造方法的参数意义以及获取方式基本一致。下面我们看下另外两个参数streamType以及mode

streamType音频流的类型，有如下几种

-  AudioManager#STREAM_VOICE_CALL：电话声音
- AudioManager#STREAM_SYSTEM：系统声音
- AudioManager#STREAM_RING：铃声
- AudioManager#STREAM_MUSIC：音乐声
- AudioManager#STREAM_ALARM：闹铃声
-  AudioManager#STREAM_NOTIFICATION：通知声

这里我们使用的是AudioManager#STREAM_MUSIC。
 下面我们重点看下mode
 @param mode streaming or static buffer.
 **MODE_STATIC and MODE_STREAM**

- STATIC模式：一次性将所有的数据放到一个固定的buffer，然后直接传送给AudioTrack，简单有效，通常应用于播放铃声或者系统提示音等，占用内存较少的音频数据
- STREAM模式：一次一次的将音频数据流写入到AudioTrack对象中，并持续处于阻塞状态，当数据从Java层到Native层执行播放完毕后才返回，这种方式可以避免由于音频过大导致内存占用过多。当然对应的不足就是总是在java和native层进行交互，并且阻塞知道播放完毕，效率损失较大。

### 2.2. Action  写入、播放、暂停、停止、释放

write(byte audioData, int offsetInBytes, int sizeInBytes)把pcm数据写入到AudioTrack对象
 播放、暂停、停止、释放 常规的播放Action操作。

### 2.3. 状态机（getState以及getPlayState）

AudioTrack中有两个state，一个是AudioTrack是否已经初始化，后续的Action操作都依赖于此，这个有点类似MediaPlayer的prepared状态，只有处于**prepared**状态之后才可以进行其他播放相关操作
 另外一个就是**playstate**，用于记录判断当前处于什么播放状态。
 状态的改变加速，处理多线程同步问题
 private final Object mPlayStateLock = new Object();

## 3.3 具体实现

我们那上一篇《AudioRecord录制PCM音频》中示例代码产生的pcm作为AudioTrack的数据播放源来。跟进mode不同，分别实现

### 3.3.1  STATIC模式

```java
//1. 初始化参数和buffer

private void initAudioTrackParams() {
    sampleRateInHz = 44100;
    channels = AudioFormat.CHANNEL_OUT_MONO;//错误的写成了CHANNEL_IN_MONO
    audioFormat = AudioFormat.ENCODING_PCM_16BIT;
    bufferSize = AudioTrack.getMinBufferSize(sampleRateInHz, channels, audioFormat);

    pcmFile = new File(getExternalFilesDir(Environment.DIRECTORY_MUSIC), "raw.pcm");
    if (pcmFile.exists()) {
        hasPcmFile = true;
    }
}


private void initStaticBuff() {

    //staic模式是一次读取全部的数据，在play之前要先完成{@link audioTrack.write()}


    if (audioTrackThread != null) {
        audioTrackThread.interrupt();
    }

    audioTrackThread = new Thread(new Runnable() {
        @Override
        public void run() {
            FileInputStream fileInputStream = null;
            try {
                //init audioTrack 需要先确定buffersize
                fileInputStream = new FileInputStream(pcmFile);
                long size = fileInputStream.getChannel().size();
                staicBuff = new byte[(int) size];

                ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream(staicBuff.length);
                int byteValue = 0;
                long startTime = System.currentTimeMillis();
                while ((byteValue = fileInputStream.read()) != -1) {
                    //                        Log.d(TAG, "run: " + byteValue);
                    //耗时操作
                    byteArrayOutputStream.write(byteValue);
                }
                Log.d(TAG, "byteArrayOutputStream write Time: " + (System.currentTimeMillis() - startTime));
                staicBuff = byteArrayOutputStream.toByteArray();

                isReadying = true;

            } catch (IOException e) {
                e.printStackTrace();
            } catch (Throwable e) {
                e.printStackTrace();
            } finally {
                if (fileInputStream != null) {
                    try {
                        fileInputStream.close();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
                Log.d(TAG, "playWithStaicMode: end");
            }
        }
    });
    audioTrackThread.start();

}

//. 2. 点击播放

private void play(byte[] staicBuff) {
    //1. static模式是一次读去pcm到内存，比较耗时，只有读取完之后才可以调用play
    if (!isReadying) {
        Toast.makeText(this, "请稍后", Toast.LENGTH_SHORT).show();
        return;
    }

    //2.如果正在播放中，重复点击播放，则停止当次播放，调用reloadStaticData重新加载数据，然后play
    if (isPlaying) {
        audioTrack.stop();
        audioTrack.reloadStaticData();
        Log.d(TAG, "playWithStaicMode: reloadStaticData");
        audioTrack.play();
        return;
    }
    //3。否则，就先释放audiotrack，然后重新初始化audiotrack进行

    releaseAudioTrack();

    int state = initAudioTrackWithMode(AudioTrack.MODE_STATIC, staicBuff.length);
    if (state == AudioTrack.STATE_UNINITIALIZED) {
        Log.e(TAG, "run: state is uninit");
        return;
    }
    //4. 把pcm写入audioTrack，然后进行播放
    long startTime = System.currentTimeMillis();
    int result = audioTrack.write(staicBuff, 0, staicBuff.length);
    Log.d(TAG, "audioTrack.write staic: result=" + result+" totaltime="+ (System.currentTimeMillis() - startTime));
    audioTrack.play();
    isPlaying = true;
}


private void pausePlay() {
    if (audioTrack != null) {
        if (audioTrack.getState() == AudioTrack.STATE_INITIALIZED) {
            audioTrack.pause();
            audioTrack.flush();
        }
        isPlaying = false;
        Log.d(TAG, "pausePlay: isPlaying false");
    }
    if (audioTrackThread != null) {
        audioTrackThread.interrupt();
    }
}

private void releaseAudioTrack() {
    if (audioTrack != null && audioTrack.getState() == AudioTrack.STATE_INITIALIZED) {
        audioTrack.stop();
        audioTrack.release();
        isPlaying = false;
        Log.d(TAG, "pausePlay: isPlaying false");
    }
    if (audioTrackThread != null) {
        audioTrackThread.interrupt();
    }
}
```

### 3.3.2   STREAM模式

```java
1. 初始化参数   
    private void initAudioTrackParams() {
    sampleRateInHz = 44100;
    channels = AudioFormat.CHANNEL_OUT_MONO;//错误的写成了CHANNEL_IN_MONO
    audioFormat = AudioFormat.ENCODING_PCM_16BIT;
    bufferSize = AudioTrack.getMinBufferSize(sampleRateInHz, channels, audioFormat);

    pcmFile = new File(getExternalFilesDir(Environment.DIRECTORY_MUSIC), "convert.wav");//"raw.pcm"
    if (pcmFile.exists()) {
        hasPcmFile = true;
    }
}

2. 点击进行播放
    private void play() {
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


private void pausePlay() {
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
```

## 3.4 遇到的问题

纸上得来终觉浅，绝知此事要实践
 本篇文章原计划昨天完成，但是在实践中遇到了不少问题，不过最终都得以解决，记录如下

1. stream模式快速点击 声音重叠，如何停止：在触发播放前先停止和释放auidoTrack，然后在进行init，在audioTrack写入数据的线程中write操作要做好audiotTrack的状态判断。具体实现见上面小节的代码
2. 如何监听播放进度：AudioTrack有没有像MediaPlayer的丰富的监听回调，比如说，播放进度，播放完成回调，异常回调等。遗憾的是还真没有，针对STATIC模式的播放结束监听倒是可以借助setNotificationMarkerPosition 和 setPlaybackPositionUpdateListener来判断来判断。具体见上面小节中STATIC模式的实现
3. staic模式下有时候无法播放；音频在快速连续点击中加了isplaying的片段，如果正在playing中有触发了play，会先stop然后调用audioTrack.reloadStaticData()加载数据流，再进行播放，但是发现快速连续点击是间隔一次才会播放生效，原因还是audioTrack资源没有被正确使用，改为了先release在进行init的方式。
4. IllegalStateException: Unable to retrieve AudioTrack pointer for write()：这个异常是stream模式时在主线程出发了stop或者release，而在audioTrack子线程write时抛出的异常，原因就是播放状态不对，如果已经处于Stropped状态，再进行write操作就会报这个错误，所以write时加个playstate状态的检验。具体解决实现见上面小节的代码。

## 3.5 参考

《音视频开发进阶指南》
 [《使用 AudioTrack 播放PCM音频》](https://links.jianshu.com/go?to=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FiiqOP5AqMYYSssHVAhW6Gg)
 [Android 音频系统：从 AudioTrack 到 AudioFlinger](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fzyuanyun%2Farticle%2Fdetails%2F60890534)

## 3.6 收获

通过对AudioTrack的学习实践，收获如下

1. 了解了AudioTrack的在音频系统中的作用以及使用场景
2. AudioTrack的两种模式STATIC和STREAM的区别和使用方式
3. AudioTrack的基本使用
4. 实践中遇到的问题以及解决方案和思路

