# 13-OpenGL ES 滤镜 (篇一)

## 13.0 目录

1. 颜色和滤镜的基本知识
2. 实践：通过ColorFilter实现颜色颜色调节
3. 实践：图片滤镜(黑白、冷暖色)
4. 遇到的问题
5. 资料
6. 收获

## 13.1 颜色和滤镜的基本知识

我们是如何看到图不同颜色的？



![img](https:////upload-images.jianshu.io/upload_images/1791669-3a6faf63c41557b4.png?imageMogr2/auto-orient/strip|imageView2/2/w/640/format/webp)

图片来源：[播放器色觉辅助功能开发，助力提升色觉障碍用户的视频观看体验]

不同波长的光具有不同的颜色，在我们可见光范围内蓝色光波长是短波，长波长的光呈现红色。
 我们人类有三种不同的视锥细胞，它们对不同的光有不同的敏感度，由于不同人的视锥细胞，也造成了世界上有4%-6%的色弱或者色盲者。



<img src="https:////upload-images.jianshu.io/upload_images/1791669-2977ef61c0cc786a.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp" alt="img" style="zoom:50%;" />

图片来源：[播放器色觉辅助功能开发，助力提升色觉障碍用户的视频观看体验]

我们这篇学习的滤镜，对于视觉有障碍的人群可以起到视觉校正的作用。同时对于视觉正常的人群，也可以通过滤镜使色彩发生变化，适合不同的场景，比如，哀悼日模式，在或者应用更广的短视频或者直播中的美颜、滤镜等功能，增加一些趣味性。

了解背景后，下面我们从颜色的三个要素来一起学习下颜色。
 **颜色三要素：色调（色相）、饱和度、亮度**

> 色调是区别各种不同色彩的最准确的标准，任何黑白灰以外的颜色都有色相的属性，而色相也就是由原色、间色和复色来构成的。色相，色彩可呈现出来的质的面貌。
>  根据色环的色彩排列，相邻色相混合，饱和度基本不变(如红黄相混合所得的橙色)。对比色相混合，最易降低饱和度，以至成为灰暗色彩
>  亮度不仅决定物体照明程度，而且决定物体表面的反射系数。如果我们看到的光线来源于光源，那么亮度决定于光源的强度。如果我们看到的是来源于物体表面反射的光线，那么亮度决定于照明的光源的强度和物体表面的反射系数。
>  来自 颜色三要素百科

<img src="https:////upload-images.jianshu.io/upload_images/1791669-54ffe112d441535a.jpeg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp" alt="img" style="zoom:50%;" />

图片来自：[色环百科]

在android中有ColorMatrix颜色矩阵工具类，帮我们封装实现了颜色的三要素的调节以及不同矩阵相乘的实现。

**色调**



```cpp
public void setRotate(int axis, float degrees) {
    reset();
    double radians = degrees * Math.PI / 180d;
    float cosine = (float) Math.cos(radians);
    float sine = (float) Math.sin(radians);
    switch (axis) {
            // Rotation around the red color
        case 0:
            mArray[6] = mArray[12] = cosine;
            mArray[7] = sine;
            mArray[11] = -sine;
            break;
            // Rotation around the green color
        case 1:
            mArray[0] = mArray[12] = cosine;
            mArray[2] = -sine;
            mArray[10] = sine;
            break;
            // Rotation around the blue color
        case 2:
            mArray[0] = mArray[6] = cosine;
            mArray[1] = sine;
            mArray[5] = -sine;
            break;
        default:
            throw new RuntimeException();
    }
}
```

我们可以看到色调的调节有两个参数，参数axis代表 围绕哪种颜色进行旋转，degress是指旋转的角度。范围是【-180度，180度】
 比如下面这个矩阵就是axis为红色时的结果。



![img](https:////upload-images.jianshu.io/upload_images/1791669-d0f1260ca7ec83d7.png?imageMogr2/auto-orient/strip|imageView2/2/w/228/format/webp)

**饱和度**
 饱和度(saturation)色彩的鲜艳程度



```java
public void setSaturation(float sat) {
    reset();
    float[] m = mArray;

    final float invSat = 1 - sat;
    final float R = 0.213f * invSat;
    final float G = 0.715f * invSat;
    final float B = 0.072f * invSat;

    m[0] = R + sat; m[1] = G;       m[2] = B;
    m[5] = R;       m[6] = G + sat; m[7] = B;
    m[10] = R;      m[11] = G;      m[12] = B + sat;
}
```

参数sat的代表饱和度的强弱。范围是【0，1】
 例如，当sat为0时，RGB对应的值为0.213f，0.715f，0.072f，这是就可以实现黑白模式

**亮度**



```java
public void setScale(float rScale, float gScale, float bScale,
                     float aScale) {
    final float[] a = mArray;

    for (int i = 19; i > 0; --i) {
        a[i] = 0;
    }
    a[0] = rScale;
    a[6] = gScale;
    a[12] = bScale;
    a[18] = aScale;
}
```

四个参数分别代表rgba四个通道的的范围，范围区间在【0，1】。

通过上面的介绍，我们了解到可以通过修改颜色的三要素实现滤镜的功能。下面开启我们的实践。

## 13.2 实践：ColorFilter对View进行换色

我们先通过颜色矩阵设置固定的值来对普通图片的颜色修改，实现黑白、暖色、冷色三种变化，然后在通过动态的调整色调、饱和度、以及亮度进行实现图片的颜色调节。下面开启我们这一小节的旅程。

首先我们可以直接给View的Bitamp设置ColorFilter，来实现颜色的变化



```cpp
//黑白模式 Gray=R*0.3+G*0.59+B*0.11
float[] mBWMatrix = {
    0.3f,0.59f,0.11f,0,0,
    0.3f,0.59f,0.11f,0,0,
    0.3f,0.59f,0.11f,0,0,
    0,0,0,1,0};

//暖色调的处理可以增加红绿通道的值
float[] mWarmMatrix = {
    2,0,0,0,0,
    0,2,0,0,0,
    0,0,1,0,0,
    0,0,0,1,0};

//冷色调的处理可以通过单一增加蓝色通道的值
float[] mCoolMatrix = {
    1,0,0,0,0,
    0,1,0,0,0,
    0,0,2,0,0,
    0,0,0,1,0};

public void setImageMatrix(float[] mColorMatrix) {
    Bitmap bmp = Bitmap.createBitmap(mBitmap.getWidth(),mBitmap.getHeight(),Bitmap.Config.ARGB_8888);
    ColorMatrix colorMatrix = new ColorMatrix();
    colorMatrix.set(mColorMatrix);
    Canvas canvas = new Canvas(bmp);
    Paint paint = new Paint();
    paint.setColorFilter(new ColorMatrixColorFilter(colorMatrix));
    canvas.drawBitmap(mBitmap,0,0,paint);
    ivImage.setImageBitmap(bmp);
}
```

我们也可以直接通过修改颜色的饱和度来实现黑白色



```cpp
public static Bitmap handleImageEffect(Bitmap oriBmp,  float hue, float saturation, float lum) {
    Bitmap bmp = Bitmap.createBitmap(oriBmp.getWidth(), oriBmp.getHeight(), Bitmap.Config.ARGB_8888);
    Canvas canvas = new Canvas(bmp);
    Paint paint = new Paint();

    ColorMatrix saturationMatrix = new ColorMatrix();
    saturationMatrix.setSaturation(saturation);

    paint.setColorFilter(new ColorMatrixColorFilter(saturationMatrix));
    canvas.drawBitmap(oriBmp, 0, 0, paint);

    return bmp;
}
```

效果如下：



![img](https:////upload-images.jianshu.io/upload_images/1791669-1680cc2f59b56cd8.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)

ColorMatrix也提供了矩阵相乘的功能，这样就可以同时图片修改色调、饱和度、亮度。



```java
private float mSaturaion =1;
private float mLum=1;
private float mHue=0;
@Override
public void onProgressChanged(SeekBar seekBar, int progress, boolean fromUser) {
    int id = seekBar.getId();
    switch (id){
        case R.id.sb_hue:
            mHue = (progress - midValue) * 1.0F / midValue * 180;
            break;
        case R.id.sb_saturation:
            mSaturaion = progress*1.0f/midValue;
            break;
        case R.id.sb_lum:
            mLum = progress*1.0f/midValue;
            break;
    }
    Log.d(TAG, "onProgressChanged: midValue="+midValue+" mhue="+mHue+" msaturation="+mSaturaion+" mlum="+mLum+" progress="+progress);

    ivImage.setImageBitmap(handleImageEffect(mBitmap,mHue,mSaturaion,mLum));

}

/**
     * 色调、饱和度、亮度 通过ColorMatrix的PostConcat相乘，对原始图片进行变换处理
     * @param oriBmp
     * @param hue 色调调节范围 -180度至180度
     * @param saturation
     * @param lum
     * @return
     */
public static Bitmap handleImageEffect(Bitmap oriBmp,  float hue, float saturation, float lum) {
    Bitmap bmp = Bitmap.createBitmap(oriBmp.getWidth(), oriBmp.getHeight(), Bitmap.Config.ARGB_8888);
    Canvas canvas = new Canvas(bmp);
    Paint paint = new Paint();

    //调节色调
    Log.i(TAG, "handleImageEffect: 色调 rotate="+hue);
    ColorMatrix hueMatrix = new ColorMatrix();
    hueMatrix.setRotate(0, hue);//围绕red旋转 hue角度
    hueMatrix.setRotate(1, hue);//围绕green旋转 hue角度
    hueMatrix.setRotate(2, hue);//围绕blue旋转 hue角度

    //调节饱和度
    Log.i(TAG, "handleImageEffect: 饱和度saturation="+saturation);
    ColorMatrix saturationMatrix = new ColorMatrix();
    saturationMatrix.setSaturation(saturation);

    //调节亮度
    Log.i(TAG, "handleImageEffect: 亮度lum="+lum);
    ColorMatrix lumMatrix = new ColorMatrix();
    lumMatrix.setScale(lum, lum, lum, 1);

    ColorMatrix imageMatrix = new ColorMatrix();
    imageMatrix.postConcat(hueMatrix);
    imageMatrix.postConcat(saturationMatrix);
    imageMatrix.postConcat(lumMatrix);

    paint.setColorFilter(new ColorMatrixColorFilter(imageMatrix));
    canvas.drawBitmap(oriBmp, 0, 0, paint);

    return bmp;
}
```

效果如下：



![img](https:////upload-images.jianshu.io/upload_images/1791669-8e98d8e7ad261200.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)

## 13.3 实践：OpenGL ES实现图片滤镜

这一小节，我们通过OpenGL ES来实现颜色的变化，具体流程如下

1. 通过GlSurfaceView的Render中进行加载着色器和进行Frame的绘制
2. 通过外部设置给glsl传入不同的滤镜类型，在glsl中进行根据不同的type进行不同的颜色变化。

顶点着色器和上一篇OpenGL ES添加纹理中基本一致，片元着色器我们要添加两个uniform类型，分别代表滤镜类型和滤镜的颜色向量。



```cpp
//texture_vertex_shader.glsl


uniform mat4 u_Matrix;

attribute vec4 a_Position;
attribute vec3 a_Color;
attribute vec2 a_TextureCoordinates;

varying vec2 v_TextureCoordinates;
varying vec3 v_Color;


void main()                    
{                            
    v_TextureCoordinates = a_TextureCoordinates;
    v_Color = a_Color;

    gl_Position = a_Position;
}       

//image_filter_texture_fragment_shader.glsl

precision mediump float;

uniform sampler2D u_TextureUnit;
uniform int u_TypeIndex;
varying vec2 v_TextureCoordinates;
varying vec3 v_Color;

void main()
{
    vec4 color = texture2D(u_TextureUnit, v_TextureCoordinates);

    if (u_TypeIndex == 0){
        gl_FragColor = color;
    } else if (u_TypeIndex == 1){
        float c = color.r * v_Color.r +
            color.g * v_Color.g +
            color.b * v_Color.b;
        gl_FragColor = vec4(c, c, c, 1.0f);
    } else {
        vec4 newColor = color + vec4(v_Color, 0.0f);
        gl_FragColor = newColor;
    }
}
```

下面看下Render中如何加载着色器和给着色器中属性设置值



```java
public class ImageFilterRender implements GLSurfaceView.Renderer {

    private Context context;
    private int textureId;
    private TextureShaderProgram textureProgram;
    private BgTextureObject textureObject;

    public ImageFilterRender(Context context) {
        this.context = context;
    }

    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        GLES10.glClearColor(0f, 0f, 0f, 0f);

        textureObject = new BgTextureObject(ImageBgData.VERTEX_DATA);

        String fragmentCode = ShaderHelper.loadAsset(MyApplication.getContext().getResources(), "image_filter_texture_fragment_shader.glsl");

        String vertexCode = ShaderHelper.loadAsset(MyApplication.getContext().getResources(), "texture_vertex_shader.glsl");

        textureProgram = new TextureShaderProgram(context, vertexCode, fragmentCode);

        textureId = TextureHelper.loadTexture(context, R.drawable.bg);

    }

    private void refreshTexureProgram(String fragmentCode) {

    }

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        GLES20.glViewport(0, 0, width, height);
    }

    @Override
    public void onDrawFrame(GL10 gl) {
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);
        textureProgram.useProgram();


        //
        GLES20.glUniform1i(textureProgram.getFilterIndexUniformLocation(), mIndex);

        switch (mIndex){
            case 0:
                //原图效果，不同处理
                break;
            case 1:
                //黑白滤镜
                GLES20.glVertexAttrib3fv(textureProgram.getProgram(),ImageBgData.GRAY_FILTER_COLOR_DATA,0);
                break;
            case 2:
                //暖色滤镜
                GLES20.glVertexAttrib3fv(textureProgram.getProgram(), ImageBgData.WARM_FILTER_COLOR_DATA, 0);
                break;
            case 3:
                //冷色滤镜
                GLES20.glVertexAttrib3fv(textureProgram.getProgram(), ImageBgData.COOL_FILTER_COLOR_DATA, 0);
                break;
        }

        textureProgram.setUniforms(textureId);

        textureObject.bindData(textureProgram);

        textureObject.draw();


    }

    private int mIndex;

    public void setFilter(int index) {

        mIndex = index;

    }
}
```

效果如下



![img](https:////upload-images.jianshu.io/upload_images/1791669-48e233901e7993ce.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)

[源码已上传到github](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fayyb1988%2Fmediajourney)

## 13.4 遇到的问题

问题1. 从新加载shader时，报错：ShaderHelper: loadProgram: glCreateProgram error errorCode=0

目的是想通过点击按钮进行 原图、黑白、暖色、冷色的效果切换。采用了重新加载shadercode和程序的方案，报了上面的错误，后来想一想更合理的做法是通过设置不同类型的滤镜告诉glsl，然后在glsl内部进行处理，这样在onDrawFrame刷新时即可看到效果，而不用重新创建program。

问题2:  ‘u_TypeIndex' : Syntax error:  syntax error_

用了vec1或者ivec1， 向量最少是两个，glsl本身支持int，float类型，我这里目的只是给片元着色器传一个tpye类型用于，片元着色器内针对不同滤镜使用不同算法逻辑。

如何把录制的mp4转为大小合适的gif图片，用于上传



```swift
ffmpeg -i colorfilter2.mp4 -r 16 -s 320x480 -filter:v "setpts=0.5*PTS" -b 240k colorfilter2.gif

-r 16: 帧率 16fps
-s 320x480: 宽高
-filter:v "setpts=0.5*PTS" ： 倍速
```

## 13.5 资料

[Android滤镜效果实现及原理分析]
 [播放器色觉辅助功能开发，助力提升色觉障碍用户的视频观看体验]
 [专栏：Android图像处理之实时滤镜]
 [专栏：图像处理]
 [Android OpenGL ES(四)-为平面图添加滤镜]
 [Android学习笔记22：图像颜色处理（ColorMatrix）]

## 13.6 收获

1. 学习了解了颜色和滤镜的基本知识
2. 通过ColorFilter来修改颜色转换（黑白、暖色、冷色）
3. 通过openGl es来修改颜色实现滤镜效果（黑白、暖色、冷色）
4. 加强了对glsl的认知学习

