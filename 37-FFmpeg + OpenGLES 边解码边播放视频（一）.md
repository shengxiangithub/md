# 37-FFmpeg + OpenGLES 边解码边播放视频（一）

## 37.0 目录

1. 基础知识
2. 使用GLSurfaceView播放边解码边播放视频
3. 遇到的问题
4. 资料
5. 收获

## 37.1 基础知识

### 37.1.1. YUV和RGB

视频是由一幅幅图像或者说一帧帧 YUV 数据组成
 表示图片、视频的色彩空间有几种：YUV、RGB、HSV等，FFmpeg解码后的视频数据是YUV数据，而OpenGL ES 渲染时要使用RGB数据，为此我们需要把YUV先转成RGB，对应的转换公式如下：



```undefined
 rgb = mat3(
    1.0, 1.0, 1.0,
    0.0, -0.39465, 2.03211,
    1.13983, -0.5806, 0.0
    ) *yuv;
```

### 37.1.2 OpenGL ES基础知识

我们在第二个系列中已经对OpenGLES的基本流程和GLSL语法以及绘制各种图形、矩阵变换等进行过学习实践。不清楚的或者遗忘的可以回顾下。
 OpenGL ES涉及的知识点和可以做的东西是非常丰富，后面还会对其有一系列更深入的学习实践。
 [音视频开发之旅（七） OpenGL ES 基本概念](https://links.jianshu.com/go?to=http%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzU5NjkxMjE5Mg%3D%3D%26mid%3D2247483764%26idx%3D1%26sn%3Da526d2ecd5c826ef15ebe9973ffd1d55%26chksm%3Dfe5a305bc92db94dfe9d45d387d71bb8521a383075b94f3613f729bd1cdbe86e1eb0b9334bdb%26scene%3D21%23wechat_redirect)

[音视频开发之旅（八）GLSL及Shader的渲染流程](https://links.jianshu.com/go?to=http%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzU5NjkxMjE5Mg%3D%3D%26mid%3D2247483771%26idx%3D1%26sn%3D9b122a361188aa4cc0d75549be0ee4a4%26chksm%3Dfe5a3054c92db942d350306739c2c775c95d107b8f2a213d6ddc9cbc5c042c702dc1d36c3581%26scene%3D21%23wechat_redirect)

[音视频开发之旅（九） OpenGL ES 绘制平面图形](https://links.jianshu.com/go?to=http%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzU5NjkxMjE5Mg%3D%3D%26mid%3D2247483783%26idx%3D1%26sn%3D6c8fa673eff0aaffe0872227432c3214%26chksm%3Dfe5a30a8c92db9bea01b92d35c37efa16a7acb08237bdf6ad0db510549e3b8a14d692fbac638%26scene%3D21%23wechat_redirect)

[音视频开发之旅（十） GLSurfaceView源码解析&EGL环境](https://links.jianshu.com/go?to=http%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzU5NjkxMjE5Mg%3D%3D%26mid%3D2247483789%26idx%3D1%26sn%3D0aa8af3e68c75070b7bc9cdf763e1561%26chksm%3Dfe5a30a2c92db9b428cb9725873d81cafe1695b5d946a1238e73aed021c21a89773ccd1b3626%26scene%3D21%23wechat_redirect)

[音视频开发之旅（11） OpenGL ES矩阵变换与坐标系统](https://links.jianshu.com/go?to=http%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzU5NjkxMjE5Mg%3D%3D%26mid%3D2247483815%26idx%3D1%26sn%3D03c4416b5d595c44a5edbfa157205a51%26chksm%3Dfe5a3088c92db99e5207ebc3ec17fe86d1d555489c75665205a56119a8fd2ca628038221def0%26scene%3D21%23wechat_redirect)

[音视频开发之旅（12） OpenGL ES之纹理](https://links.jianshu.com/go?to=http%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzU5NjkxMjE5Mg%3D%3D%26mid%3D2247483829%26idx%3D1%26sn%3D87fc8bf33bb5ed8c91ad2fba0f7f295f%26chksm%3Dfe5a309ac92db98ce5ad7b63f22d65f383547c1f12c93cbd86ca4296530fe2c791b443db3a40%26scene%3D21%23wechat_redirect)

## 37.2 使用GLSurfaceView播放解码的YUV数据

在**前面几篇**我们实现了对视频流的解码生成了YUV裸流，当时是通过YUVplayer和ffplayer在pc上进行的验证。这一小节，我们通过Android 提供的GLSurfaceview来进行视频的渲染。因为GLsurfaceView已经有了EGL渲染线程，本篇我们先通过使用熟悉渲染流程

首先我们写下顶点着色器和片源着色器。
 顶点着色器



```cpp
//#version 120

attribute vec4 aPosition;
attribute vec2 aTextureCoord;

varying vec2 vTextureCoord;

void main() {
    gl_Position = aPosition;
    vTextureCoord = aTextureCoord;
}
```

片源着色器



```cpp
//#version 120
precision mediump float;

varying vec2 vTextureCoord;

uniform sampler2D samplerY;
uniform sampler2D samplerU;
uniform sampler2D samplerV;

void main() {
    vec3 yuv;
    vec3 rgb;

    yuv.r=texture2D(samplerY, vTextureCoord).g;
    yuv.g=texture2D(samplerU, vTextureCoord).g -0.5;
    yuv.b=texture2D(samplerV, vTextureCoord).g-0.5;

    rgb = mat3(
    1.0, 1.0, 1.0,
    0.0, -0.39465, 2.03211,
    1.13983, -0.5806, 0.0
    ) *yuv;

    gl_FragColor = vec4(rgb,1.0);
}
```

Render代码如下,也是比较常规的操作，又不清楚的，可以回看下OpenGL系列内容



```java
package android.spport.mylibrary2;

import android.content.res.Resources;
import android.opengl.GLES20;
import android.opengl.GLSurfaceView;
import android.util.Log;

import java.nio.ByteBuffer;
import java.nio.ByteOrder;
import java.nio.FloatBuffer;

import javax.microedition.khronos.egl.EGLConfig;
import javax.microedition.khronos.opengles.GL10;

public class MyRender implements GLSurfaceView.Renderer {
    private Resources resources;
    private int program;

    private float verCoords[] = {
        //            1.0f, -1.0f,
        //            -1.0f, -1.0f,
        //            1.0f, 1.0f,
        //            -1.0f, 1.0f
        -1f, -1f,
        1f, -1f,
        -1f, 1f,
        1f, 1f
    };
    private float textureCoords[] = {
        //            1.0f, 0.0f,
        //            0.0f, 0.0f,
        //            1.0f, 1.0f,
        //            0.0f, 1.0f
        0f,1f,
        1f, 1f,
        0f, 0f,
        1f, 0f
    };
    private final int BYTES_PER_FLOAT = 4;

    private int aPositionLocation;
    private int aTextureCoordLocation;
    private int samplerYLocation;
    private int samplerULocation;
    private int samplerVLocation;

    private FloatBuffer verCoorFB;
    private FloatBuffer textureCoorFB;

    private int[] textureIds;

    public MyRender(Resources resources) {
        this.resources = resources;
    }

    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        GLES20.glClearColor(0.0f, 0.0f, 0.0f, 1.0f);

        String vertexShader = ShaderHelper.loadAsset(resources, "vertex_shader.glsl");
        String fragShader = ShaderHelper.loadAsset(resources, "frag_shader.glsl");
        program = ShaderHelper.loadProgram(vertexShader, fragShader);

        aPositionLocation = GLES20.glGetAttribLocation(program, "aPosition");
        aTextureCoordLocation = GLES20.glGetAttribLocation(program, "aTextureCoord");
        samplerYLocation = GLES20.glGetUniformLocation(program, "samplerY");
        samplerULocation = GLES20.glGetUniformLocation(program, "samplerU");
        samplerVLocation = GLES20.glGetUniformLocation(program, "samplerV");

        verCoorFB = ByteBuffer.allocateDirect(verCoords.length * BYTES_PER_FLOAT)
            .order(ByteOrder.nativeOrder())
            .asFloatBuffer()
            .put(verCoords);
        verCoorFB.position(0);

        textureCoorFB = ByteBuffer.allocateDirect(textureCoords.length * BYTES_PER_FLOAT)
            .order(ByteOrder.nativeOrder())
            .asFloatBuffer()
            .put(textureCoords);
        textureCoorFB.position(0);

        //对应Y U V 三个纹理
        textureIds = new int[3];
        GLES20.glGenTextures(3, textureIds, 0);

        for (int i = 0; i < 3; i++) {
            GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, textureIds[i]);

            GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_WRAP_S, GLES20.GL_REPEAT);
            GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_WRAP_T, GLES20.GL_REPEAT);
            GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_MIN_FILTER, GLES20.GL_LINEAR);
            GLES20.glTexParameteri(GLES20.GL_TEXTURE_2D, GLES20.GL_TEXTURE_MAG_FILTER, GLES20.GL_LINEAR);
        }
    }

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        GLES20.glViewport(0, 0, width, height);

    }

    @Override
    public void onDrawFrame(GL10 gl) {
        Log.i("MyRender", "onDrawFrame: width="+width+" height="+height);
        if (width > 0 && height > 0 && y != null && u != null && v != null) {

            GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);
            GLES20.glUseProgram(program);

            GLES20.glEnableVertexAttribArray(aPositionLocation);
            GLES20.glVertexAttribPointer(aPositionLocation, 2, GLES20.GL_FLOAT, false, 2 * BYTES_PER_FLOAT, verCoorFB);

            GLES20.glEnableVertexAttribArray(aTextureCoordLocation);
            GLES20.glVertexAttribPointer(aTextureCoordLocation, 2, GLES20.GL_FLOAT, false, 2 * BYTES_PER_FLOAT, textureCoorFB);

            //激活纹理
            GLES20.glActiveTexture(GLES20.GL_TEXTURE0);
            GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, textureIds[0]);
            GLES20.glTexImage2D(GLES20.GL_TEXTURE_2D,
                                0,
                                GLES20.GL_LUMINANCE,
                                width,
                                height,
                                0,
                                GLES20.GL_LUMINANCE,
                                GLES20.GL_UNSIGNED_BYTE,
                                y
                               );

            GLES20.glActiveTexture(GLES20.GL_TEXTURE1);
            GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, textureIds[1]);
            GLES20.glTexImage2D(GLES20.GL_TEXTURE_2D,
                                0,
                                GLES20.GL_LUMINANCE,
                                width / 2,
                                height / 2,
                                0,
                                GLES20.GL_LUMINANCE,
                                GLES20.GL_UNSIGNED_BYTE,
                                u
                               );

            GLES20.glActiveTexture(GLES20.GL_TEXTURE2);
            GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, textureIds[2]);
            GLES20.glTexImage2D(GLES20.GL_TEXTURE_2D,
                                0,
                                GLES20.GL_LUMINANCE,
                                width/2,
                                height/2,
                                0,
                                GLES20.GL_LUMINANCE,
                                GLES20.GL_UNSIGNED_BYTE,
                                v
                               );

            GLES20.glUniform1i(samplerYLocation, 0);
            GLES20.glUniform1i(samplerULocation, 1);
            GLES20.glUniform1i(samplerVLocation, 2);

            y.clear();
            y = null;
            u.clear();
            u = null;
            v.clear();
            v = null;

            GLES20.glDrawArrays(GLES20.GL_TRIANGLE_STRIP, 0, 4);

            GLES20.glDisableVertexAttribArray(aPositionLocation);
            GLES20.glDisableVertexAttribArray(aTextureCoordLocation);
        }

    }

    private int width;
    private int height;
    private ByteBuffer y;
    private ByteBuffer u;
    private ByteBuffer v;

    public void setYUVRenderData(int width, int height, byte[] y, byte[] u, byte[] v) {
        this.width = width;
        this.height = height;
        this.y = ByteBuffer.wrap(y);
        this.u = ByteBuffer.wrap(u);
        this.v = ByteBuffer.wrap(v);
    }
}
```

视频解码后通过JNI，CPP调用Java的回调函数把YUV数据给到java层的借助GlSurfaceView进行渲染。



```cpp
extern "C" {
    #include "include/libavcodec/avcodec.h"
    #include "include/libavformat/avformat.h"
    #include "include/log.h"
    #include <libswscale/swscale.h>
    #include <libavutil/imgutils.h>
    #include <libswresample/swresample.h>
    #include <SLES/OpenSLES.h>
    #include <SLES/OpenSLES_Android.h>
    #include <libavutil/time.h>
}

jmethodID onCallYuvData;
jobject jcallJavaobj;

JavaVM* javaVM;
extern "C"
    JNIEXPORT void JNICALL
    Java_android_spport_mylibrary2_Demo_initYUVNativeMethod(JNIEnv *env, jobject thiz) {
    //    jcallJavaobj = thiz;
    jcallJavaobj = env->NewGlobalRef(thiz);

    env->GetJavaVM(&javaVM);
    onCallYuvData = env->GetMethodID(env->GetObjectClass(thiz), "onCallYUVData",
                                     "(II[B[B[B)V");
}

extern "C"
    JNIEXPORT jint JNICALL
    Java_android_spport_mylibrary2_Demo_decodeVideo(JNIEnv *env, jobject thiz, jstring inputPath,
                                                    jstring outPath) {
    ...
        //把数据回调给java层，通过OpenGL进行渲染（当然也可以在native层构建OpenGL环境进行实现，这里借助了GLSurfaceView）
        if(onCallYuvData!=NULL)
        {
            jbyteArray yData = env->NewByteArray(y_size);
            jbyteArray uData = env->NewByteArray(y_size/4);
            jbyteArray vData = env->NewByteArray(y_size/4);

            env->SetByteArrayRegion(yData, 0, y_size,
                                    reinterpret_cast<const jbyte *>(pFrameYUV->data[0]));
            env->SetByteArrayRegion(uData, 0, y_size/4, reinterpret_cast<const jbyte *>(pFrameYUV->data[1]));
            env->SetByteArrayRegion(vData, 0, y_size/4, reinterpret_cast<const jbyte *>(pFrameYUV->data[2]));
            //                env->SetByteArrayRegion(vData, 0, y_size/4, reinterpret_cast<const jbyte *>(pFrameYUV->data[1]));
            //                env->SetByteArrayRegion(uData, 0, y_size/4, reinterpret_cast<const jbyte *>(pFrameYUV->data[2]));


            LOGI("native onCallYuvData widith=%d",pCodecParameters->width);

            //jcallJavaobj 在赋值时候要通过env->NewGlobalRef(thiz);设置伟全局变量，否则会出现野导致指针异常
            env->CallVoidMethod(jcallJavaobj,onCallYuvData,pCodecParameters->width,pCodecParameters->height,yData,uData,vData);

            env->DeleteLocalRef(yData);
            env->DeleteLocalRef(uData);
            env->DeleteLocalRef(vData);

            //解码太快，来不及渲染，导致前面待渲染但还没有渲染的数据被后解码的数据给覆盖了。
            //由于渲染和解码线程现在还没有做分离同步以及加入解码buffer，所以此处采用延迟的方案处理解决。
            av_usleep(1000 * 50);
        }
    ...
}
```

实现视频的播放。

我们可以通过JNI回调，把解码后的yuv传给java层进行渲染，这本是是一个消耗，是否能够通过CPP层直接完成渲染呐？当然可以，音频OpenGL ES提供了Java和native的支持，我们完全可以在native层进行渲染，只不过nativew层没有类似GLSuerfaceView即封装好的EGL环境，这样就需要我们自己创建GL渲染线程进行渲染。我们后续来进行学习实践，在native层通过解码线程和渲染线程 使用OpenSL ES渲染播放音频、OpenGL ES渲染视频。

代码已上传至[github](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fayyb1988%2Fffmpegvideodecodedemo) [[https://github.com/ayyb1988/ffmpegvideodecodedemo\]](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fayyb1988%2Fffmpegvideodecodedemo%5D)  欢迎交流，一起学习成长。

## 37.3 遇到的问题

1. **运行时出现 JNI DETECTED ERROR IN APPLICATION异常**



```rust
5.963 5247-5247/? A/DEBUG: Abort message: 'JNI DETECTED ERROR IN APPLICATION: use of invalid jobject 0x7fcea23564
        from int android.spport.mylibrary2.Demo.decodeVideo(java.lang.String, java.lang.String)'
2021-03-10 06:57:35.963 5247-5247/? A/DEBUG:     x0  0000000000000000  x1  0000000000000dd6  x2  0000000000000006  x3  0000007fcea22390
2021-03-10 06:57:35.963 5247-5247/? A/DEBUG:     x4  fefeff7939517f97  x5  fefeff7939517f97  x6  fefeff7939517f97  x7  7f7f7f7f7f7fffff
2021-03-10 06:57:35.963 5247-5247/? A/DEBUG:     x8  00000000000000f0  x9  2cd4cdcb09dc01f0  x10 0000000000000001  x11 0000000000000000
2021-03-10 06:57:35.963 5247-5247/? A/DEBUG:     x12 fffffff0fffffbdf  x13 ffffffffffffffff  x14 0000000000000004  x15 ffffffffffffffff
2021-03-10 06:57:35.963 5247-5247/? A/DEBUG:     x16 0000007a3a7618c0  x17 0000007a3a73d900  x18 0000007a3c048000  x19 0000000000000dd6
2021-03-10 06:57:35.963 5247-5247/? A/DEBUG:     x20 0000000000000dd6  x21 00000000ffffffff  x22 000000799cca7cc0  x23 00000079b5130625
2021-03-10 06:57:35.963 5247-5247/? A/DEBUG:     x24 00000079b51520fd  x25 0000000000000001  x26 00000079b4fbc258  x27 0000007a3b8067c0
2021-03-10 06:57:35.963 5247-5247/? A/DEBUG:     x28 00000079b565b338  x29 0000007fcea22430
2021-03-10 06:57:35.963 5247-5247/? A/DEBUG:     sp  0000007fcea22370  lr  0000007a3a6ef0c4  pc  0000007a3a6ef0f0
2021-03-10 06:57:36.026 2647-2647/? E/ndroid.systemu: Invalid ID 0x00000000.
2021-03-10 06:57:36.047 12827-20222/? E/Hack.Hub: net.connect = I'm afraid to call its toString()
2021-03-10 06:57:36.071 5247-5247/? A/DEBUG: backtrace:
2021-03-10 06:57:36.071 5247-5247/? A/DEBUG:       #00 pc 00000000000830f0  /apex/com.android.runtime/lib64/bionic/libc.so (abort+160) (BuildId: e55e6e4c631509598633769798683023)
...
2021-03-10 06:57:36.072 5247-5247/? A/DEBUG:       #08 pc 000000000036771c  /apex/com.android.runtime/lib64/libart.so (art::(anonymous namespace)::ScopedCheck::Check(art::ScopedObjectAccess&, bool, char const*, art::(anonymous namespace)::JniValueType*)+652) (BuildId: d700c52998d7d76cb39e2001d670e654)
2021-03-10 06:57:36.072 5247-5247/? A/DEBUG:       #09 pc 000000000036c76c  /apex/com.android.runtime/lib64/libart.so (art::(anonymous namespace)::CheckJNI::CheckCallArgs(art::ScopedObjectAccess&, art::(anonymous namespace)::ScopedCheck&, _JNIEnv*, _jobject*, _jclass*, _jmethodID*, art::InvokeType, art::(anonymous namespace)::VarArgs const*)+132) (BuildId: d700c52998d7d76cb39e2001d670e654)
```

原因

> 将jobject保存在了一个全局变量里面，而没有使用全局引用，以上面的代码为例，即本地JNI代码里进行了类似object = caller;的赋值，这显然是没有用的，一旦函数返回，caller就会被GC回收销毁，object指向的就是一个非法地址，最终导致上面的JNI错误。

解决方案：



```rust
   jcallJavaobj = thiz;
-->改为
    jcallJavaobj = env->NewGlobalRef(thiz);
```

**2. 设置RENDERMODE_WHEN_DIRTY模式黑屏**
 通过查看log 数据到来后调用了requestRender，但没有触发onDrawFrame。

时序问题，GlSurfaceview被inflater之后其EGL环境的准备没有那么早，通过post延迟解码渲染



```java
    glSurfaceView.postDelayed(new Runnable() {
            @Override
            public void run() {
                demo.initYUVNativeMethod();
                demo.decodeVideo(folderurl+"/input.mp4", externalFilesDir+"/output7.yuv");
            }
        },300);
```

**3. 渲染出来的视频是颠倒的**

![img](https:////upload-images.jianshu.io/upload_images/1791669-ad26212fb04458d1.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)





```cpp
private float verCoords[] = {
            1.0f, -1.0f,//RB
           -1.0f, -1.0f,//LB
           1.0f, 1.0f,//RT
           -1.0f, 1.0f//LT

    };
    private float textureCoords[] = {
          1.0f, 0.0f,//RB
            0.0f, 0.0f,//LB
            1.0f, 1.0f,//RT
           0.0f, 1.0f//LT

    };

--》改为
private float verCoords[] = {

            -1f, -1f,//LB
            1f, -1f,//RB
            -1f, 1f,//LT
            1f, 1f//RT
    };
    private float textureCoords[] = {

            0f,1f, //LT
            1f, 1f,//RT
            0f, 0f,//LB
            1f, 0f //RB
    };
```

原因：OpenGL 中纹理坐标系和顶点坐标系的y轴方向都是向上的，android手机坐标系的y轴是向下的。所以openGL->手机显示，需要把坐标做上下旋转

**4. 渲染出来的视频跳帧了**
 通过log查看分析，发现是解码太快，来不及渲染，导致前面待渲染但还没有渲染的数据被后解码的数据给覆盖了。
 由于渲染和解码线程现在还没有做分离同步以及加入解码buffer，所以此处采用延迟的方案处理解决。
 在Packet解码渲染时加上50ms的延迟

![img](https:////upload-images.jianshu.io/upload_images/1791669-04aa31cfc204b2ed.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)



```undefined
 av_usleep(1000 * 50);
```

**5. 出现部分区域有绿屏并且播放的某些时刻会出现部分区域花屏的情况**
 在pc上通过ffplay播放解码后的yuv数据是正常的，而在手机上渲染出来的有问题，那边肯定是渲染出了问题，查看render代码发现，YUV纹理中的V纹理的宽度和高度设置不对导致

![img](https:////upload-images.jianshu.io/upload_images/1791669-54d8e96922939fee.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)





```rust
GLES20.glTexImage2D(GLES20.GL_TEXTURE_2D,
                    0,
                    GLES20.GL_LUMINANCE,
                    width,
                    height,
                    0,
                    GLES20.GL_LUMINANCE,
                    GLES20.GL_UNSIGNED_BYTE,
                    v
            );


--> 修改为

GLES20.glTexImage2D(GLES20.GL_TEXTURE_2D,
                    0,
                    GLES20.GL_LUMINANCE,
                    width/2,
                    height/2,
                    0,
                    GLES20.GL_LUMINANCE,
                    GLES20.GL_UNSIGNED_BYTE,
                    v
            );
```

![img](https:////upload-images.jianshu.io/upload_images/1791669-88da39945c21c641.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)

## 37.4 资料

1. [音视频学习 (八) 掌握视频基础知识并使用 OpenGL ES 2.0 渲染 YUV 数据](https://links.jianshu.com/go?to=https%3A%2F%2Fjuejin.cn%2Fpost%2F6844904064401178632)
2. [YUV <——> RGB 转换算法](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.shenyuanluo.com%2FColorConverter.html)
3. [Android平台上基于OpenGl渲染yuv视频](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fsinat%5C_23092639%2Farticle%2Fdetails%2F103046553)
4. [Android万能视频播放器04-OpenGL ES渲染YUV纹理](https://www.jianshu.com/p/741466b8f67c)
5. [JNI DETECTED ERROR IN APPLICATION解决记录](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fu010457465%2Farticle%2Fdetails%2F51221182)

## 37.5 收获

1. 回顾YUV和RGB基础知识
2. 通过GLSurfaceView实现编解码变渲染视频数据
3. 解决遇到的解码和渲染不同步导致跳帧、渲染时出现绿屏 花屏、渲染画面时颠倒的等问题

感谢你的阅读

**篇外话：**
 原计划时接下来几篇是**Native层渲染、音视频同步、编码、倍速播放、rtmp推拉流**等。但最近变得有些浮躁了是因为，需要学习的太多了，不止音视频还有Android进阶的各种知识，有个想分散精力的想法，兼顾两者，但是精力有限，有时候**必须要专注到像激光一样才能成事**。
 **考虑到工作上最近遇到的新领域，业余时间和工作上的不能够相互帮助，导致这种心理，其实是在逃避**。遇到困难，面对它，解决它。
 最近工作中使用OpenGL的比较多，很多内容也在学习实践，为了工作和学习相结合达到事半功倍的效果，**决定先暂停FFmpeg系列的更文，接下来我们聚焦在OpenGL ES渲染上**。
 调整下优先级和顺序。FFmpeg我们后会有期。



 