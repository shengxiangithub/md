# 12-OpenGL ES之纹理

## 12.0 目录

1. 纹理相关的基本概念
2. 纹理绘制的流程以及关键方法
3. 实践（纹理加载、二分屏、三分屏、八分屏、镜像、纹理和颜色混合）
4. 遇到的问题
5. 收获

## 12.1 基本概念

**纹理**
 纹理(Texture)是一个2D图片（甚至也有1D和3D的纹理），它可以用来添加物体的细节；把它像贴纸一样贴在什么东西上面，让那个东西看起来像我们贴纸所要表现的东西那样。从而使图形更加真实

**纹理坐标**

> OpenGL中纹理坐标系是以纹理左下角为坐标原点的，而图片中像素的存储顺序是从左上到右下的，因此我们需要对我们的坐标系进行一次Y轴的“翻转”。

图片坐标系的(0,0)在图片左上角，纹理坐标的(0,0)在纹理左下角

**纹理映射**

> 为了能够把纹理映射(Map)到三角形上，我们需要指定三角形的每个顶点各自对应纹理的哪个部分。这样每个顶点就会关联着一个纹理坐标(Texture Coordinate)，用来标明该从纹理图像的哪个部分采样。之后在图形的其它片段上进行片段插值(Fragment Interpolation)。

**纹理单元**

> 纹理单元是能够被着色器采样的纹理对象的引用， 纹理通过调用glBindTexture函数绑定到指定的纹理单元。没有明确指定使用哪个纹理单元时纹理被默认绑定到GL_TEXTURE0_

glActiveTexture:激活纹理单元

为什么sampler2D变量是个uniform，我们却不用glUniform给它赋值。使用glUniform1i？
 使用glUniform1i，我们可以给纹理采样器分配一个位置值，这样的话我们能够在一个片段着色器中设置多个纹理

**纹理环绕方式**

> GL_REPEAT： 默认方案，重复纹理图片。
>  GL_MIRRORED_REPEAT：类似于默认方案，不过每次重复的时候进行镜像重复。
>  GL_CLAMP_TP_EDGE：将坐标限制在0到1之间。超出的坐标会重复绘制边缘的像素，变成一种扩展边缘的图案。（通常很难看）
>  GL_CLAMP_TO_BORDER：超出的坐标将会被绘制成用户指定的边界颜色。

**纹理过滤**

> GL_NEAREST:最近点过滤：
>  纹理坐标最靠近哪个纹素，就用哪个纹素。这是OpenGL默认的过滤方式，速度最快，但是效果比较差。
>  GL_LINEAR：（双）线性过滤：
>  纹理坐标位置附近的几个纹素值进行某种插值计算之后的结果。这是应用最广泛的一种方式，效果一般，速度较快。

**多级渐进纹理（多级渐远纹理）**
 mipmaps，就是一系列的纹理图片，每一张纹理图的大小都是前一张的1/4，直到剩最后一个像素为止

> 它简单来说就是一系列的纹理图像，后一个纹理图像是前一个的二分之一。多级渐远纹理背后的理念很简单：距观察者的距离超过一定的阈值，OpenGL会使用不同的多级渐远纹理，即最适合物体的距离的那个。由于距离远，解析度不高也不会被用户注意到。同时，多级渐远纹理另一加分之处是它的性能非常好。

![img](https:////upload-images.jianshu.io/upload_images/1791669-f2ddbf2b01e27597.png?imageMogr2/auto-orient/strip|imageView2/2/w/570/format/webp)

> GL_NEAREST_MIPMAP_NEAREST：采用最近的mipmap图，在纹理采样的时候使用最近点过滤采样。
>  GL_LINEAR_MIPMAP_NEAREST：采用最近的mipmap图，纹理采样的时候使用线性过滤采样。
>  GL_NEAREST_MIPMAP_LINEAR：采用两张mipmap图的线性插值纹理图，纹理采样的时候采用最近点过滤采样。
>  GL_LINEAR_MIPMAP_LINEAR：采用两张mipmap图的线性插值纹理图，纹理采样的时候采用线性过滤采样。
>  生成mimap对应方法如下

## 12.2 纹理绘制流程和关键方法



```dart
final int[] textureObjectIds = new int[1];
//初始化纹理
glGenTextures(1, textureObjectIds, 0);

if (textureObjectIds[0] == 0) {
    return 0;
} 

//获取纹理图片
final BitmapFactory.Options options = new BitmapFactory.Options();
options.inScaled = false;

final Bitmap bitmap = BitmapFactory.decodeResource(
    context.getResources(), resourceId, options);

if (bitmap == null) {
    glDeleteTextures(1, textureObjectIds, 0);
    return 0;
} 
// 绑定纹理 2D纹理和纹理id
glBindTexture(GL_TEXTURE_2D, textureObjectIds[0]);

//设置纹理环绕方式为 GL_REPEAT
glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_WRAP_S,GL_REPEAT);
glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_WRAP_T,GL_REPEAT);

//设置纹理过滤 缩小和放大的filter
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

// 加载纹理图片
texImage2D(GL_TEXTURE_2D, 0, bitmap, 0);


//生成多级渐变纹理
glGenerateMipmap(GL_TEXTURE_2D);

//回收bitmap
bitmap.recycle();

// 解绑纹理
glBindTexture(GL_TEXTURE_2D, 0);
```

**重要方法**

glActiveTexture
 glGenTextures
 glBindTexture
 glTexParameteri
 glTexImage2D
 glGenerateMipmap
 glUniform1i

## 12.3 实践 ：加载纹理 （纹理加载、二分屏、三分屏、八分屏、镜像、纹理和颜色混合）

我们通常需要使用一张JPG和PNG等格式的图片文件作为模型的纹理，而OpenGL中并没有提供相关API用于将这些图片文件转换成我们所需要的数组。在java层我们可以通过如下方法加载bitmap到纹理，我们用广州塔灯光节的一张图片作为纹理



<img src="https:////upload-images.jianshu.io/upload_images/1791669-de2ec4542a335d2b.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp" alt="img" style="zoom:50%;" />



```dart
final BitmapFactory.Options options = new BitmapFactory.Options();
options.inScaled = false;

// Read in the resource
final Bitmap bitmap = BitmapFactory.decodeResource(
    context.getResources(), resourceId, options);

texImage2D(GL_TEXTURE_2D, 0, bitmap, 0);
```

**1. 首先来写顶点着色器和片元着色器glsl程序**



```cpp
//texture_vertex_shader.glsl

//顶点坐标
attribute vec4 a_Position;

//纹理坐标
attribute vec2 a_TextureCoordinates;

varying vec2 v_TextureCoordinates;


void main()                    
{                            
    v_TextureCoordinates = a_TextureCoordinates;

    gl_Position = a_Position;
}          
```



```cpp
//texture_fragment_shader.glsl


precision mediump float; 

//纹理单元                          
uniform sampler2D u_TextureUnit;                
//纹理坐标          
varying vec2 v_TextureCoordinates;

  
void main()                         
{      
//通过texture2D方法，传入纹理单元和纹理坐标获取颜色                         
    gl_FragColor = texture2D(u_TextureUnit, v_TextureCoordinates) ;
}
```

**2. 然后，写纹理程序**



```java
//在Render的onSurfaceCreated中创建着色器程序

public class TextureShaderProgram extends ShaderProgram {

    private final int uTextureUnitLocation;

    // Attribute locations
    private final int aPositionLocation;
    private final int aTextureCoordinatesLocation;

    public TextureShaderProgram(Context context) {
        String vertexCode = ShaderHelper.loadAsset(MyApplication.getContext().getResources(), "texture_vertex_shader.glsl");
        String fragmentCode = ShaderHelper.loadAsset(MyApplication.getContext().getResources(), "texture_fragment_shader.glsl");
        //创建着色器程序
        programId = ShaderHelper.loadProgram(vertexCode, fragmentCode);

        //纹理单元location
        uTextureUnitLocation = glGetUniformLocation(programId, U_TEXTURE_UNIT);

        //顶点坐标location
        aPositionLocation = glGetAttribLocation(programId, A_POSITION);

        //纹理坐标location
        aTextureCoordinatesLocation = 
            glGetAttribLocation(program, A_TEXTURE_COORDINATES);


    }


    public int getPositionAttributeLocation() {
        return aPositionLocation;
    }

    public int getTextureCoordinatesAttributeLocation() {
        return aTextureCoordinatesLocation;
    }

}
```

**3. 接着，生成顶点数据**



```java
public class GuangzhouTa {
    private static final int POSITION_COMPONENT_COUNT = 2;
    private static final int TEXTURE_COORDINATES_COMPONENT_COUNT = 2;
    private static final int STRIDE = (POSITION_COMPONENT_COUNT 
                                       + TEXTURE_COORDINATES_COMPONENT_COUNT) * BYTES_PER_FLOAT;


    private final VertexArray vertexArray;

    public GuangzhouTa(float[] vertexData) {
        vertexArray = new VertexArray(vertexData);
    }

    //把顶点数据和顶点着色器的location绑定赋值
    public void bindData(TextureShaderProgram textureProgram) {
        vertexArray.setVertexAttribPointer(
            0, 
            textureProgram.getPositionAttributeLocation(), 
            POSITION_COMPONENT_COUNT,
            STRIDE);


        vertexArray.setVertexAttribPointer(
            POSITION_COMPONENT_COUNT,
            textureProgram.getTextureCoordinatesAttributeLocation(),
            TEXTURE_COORDINATES_COMPONENT_COUNT, 
            STRIDE);
    }

    public void draw() {                                
        glDrawArrays(GL_TRIANGLE_FAN, 0, 6);
    }
}
```

**4. 再 加载纹理获取到纹理id**



```java
public static int loadTexture(Context context, int resourceId) {
    final int[] textureObjectIds = new int[1];
    //初始化纹理
    glGenTextures(1, textureObjectIds, 0);

    if (textureObjectIds[0] == 0) {
        return 0;
    } 

    //获取纹理图片
    final BitmapFactory.Options options = new BitmapFactory.Options();
    options.inScaled = false;

    final Bitmap bitmap = BitmapFactory.decodeResource(
        context.getResources(), resourceId, options);

    if (bitmap == null) {
        glDeleteTextures(1, textureObjectIds, 0);
        return 0;
    } 
    // 绑定纹理 2D纹理和纹理id
    glBindTexture(GL_TEXTURE_2D, textureObjectIds[0]);

    //设置纹理环绕方式为 GL_REPEAT
    glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_WRAP_S,GL_REPEAT);
    glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_WRAP_T,GL_REPEAT);

    //设置纹理过滤 缩小和放大的filter
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR);
    glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

    // 加载纹理图片
    texImage2D(GL_TEXTURE_2D, 0, bitmap, 0);


    //生成多级渐变纹理
    glGenerateMipmap(GL_TEXTURE_2D);

    //回收bitmap
    bitmap.recycle();

    // 解绑纹理
    glBindTexture(GL_TEXTURE_2D, 0);

    //范围纹理id
    return textureObjectIds[0];
}
```

**5. 最后在Render的onDrawFrame中进行绘制**



```java
public class GuangZhouTaRenderer implements Renderer {
    private final Context context;


    private GuangzhouTa guangzhouta;

    private TextureShaderProgram textureProgram;

    private int textureId;

    public GuangZhouTaRenderer(Context context) {
        this.context = context;
    }

    @Override
    public void onSurfaceCreated(GL10 glUnused, EGLConfig config) {
        glClearColor(0.0f, 0.0f, 0.0f, 0.0f);

        guangzhouta = new GuangzhouTa(VertexDataUtils.VERTEX_DATA);

        textureProgram = new TextureShaderProgram(context);

        textureId = TextureHelper.loadTexture(context, R.drawable.guangzhou);
    }

    @Override
    public void onSurfaceChanged(GL10 glUnused, int width, int height) {

        glViewport(0, 0, width, height);

    }

    @Override
    public void onDrawFrame(GL10 glUnused) {

        glClear(GL_COLOR_BUFFER_BIT);

        textureProgram.useProgram();
        textureProgram.setUniforms(textureId);
        guangzhouta.bindData(textureProgram);
        guangzhouta.draw();

    }
}


public class TextureShaderProgram{
    ....
        public void setUniforms( int textureId) {
        //激活纹理单元0
        glActiveTexture(GL_TEXTURE0);

        // 绑定纹理id
        glBindTexture(GL_TEXTURE_2D, textureId);

        //使用纹理单元0
        glUniform1i(uTextureUnitLocation, 0);
    }
    ...
}
```

我们上面使用的顶点数据矩阵是



```java
public static final float[] VERTEX_DATA = {
    // Order of coordinates: X, Y,  S, T
    // Triangle Fan
    0f,    0f,  0.5f, 0.5f,
    -1f, -1f,     0f, 1f,
    1f, -1f,   1f, 1f,
    1f,  1f,    1f, 0.0f,
    -1f,  1f,    0f, 0.0f,
    -1f, -1f,    0f, 1f };
```

效果如下



<img src="https:////upload-images.jianshu.io/upload_images/1791669-309b51110fa313de.png?imageMogr2/auto-orient/strip|imageView2/2/w/844/format/webp" alt="img" style="zoom:50%;" />

我们发现被拉伸了，**为什么会被拉伸？**
 因为纹理原图是宽高比是1:1，但是手机屏幕的宽高比一般是9:16，在水平方向上9相当于1，在垂直方向上16/9就是图片被拉伸的倍数。那么该如何处理呐？矩阵的数据纹理坐标的S和T的根据实际屏幕宽高比进行计算。

**简单的把矩阵在T坐标上放大一倍**



```java
public static final float[] SPLIT_SCREEN_2_VERTEX_DATA = {
    // Order of coordinates: X, Y,  S, T
    // Triangle Fan
    0f,    0f,  0.5f, 1f,
    -1f, -1f,   0f, 2f,
    1f, -1f,     1f, 2f,
    1f,  1f,    1f, 0.0f,
    -1f,  1f,   0f, 0.0f,
    -1f, -1f,    0f, 2f };
```

效果如下（即2分屏的效果）



<img src="https:////upload-images.jianshu.io/upload_images/1791669-a8c5a162b5f0d93a.png?imageMogr2/auto-orient/strip|imageView2/2/w/838/format/webp" alt="img" style="zoom:50%;" />

还记得我们 上面设置的纹理环绕方式为 GL_REPEAT，纹理坐标限制在0到1之间。超出的坐标会重复绘制



```undefined
glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_WRAP_S,GL_REPEAT);
glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_WRAP_T,GL_REPEAT);
```

**镜像重复的效果**

```undefined
glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_WRAP_S,GL_MIRRORED_REPEAT);
glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_WRAP_T,GL_MIRRORED_REPEAT);
```

<img src="https:////upload-images.jianshu.io/upload_images/1791669-377e4ba40d4da7a6.png?imageMogr2/auto-orient/strip|imageView2/2/w/834/format/webp" alt="img" style="zoom:50%;" />

**设置为边缘扩展效果如下（的确很难看）**



```undefined
glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_WRAP_S,GL_CLAMP_TO_EDGE);
glTexParameteri(GL_TEXTURE_2D,GL_TEXTURE_WRAP_T,GL_CLAMP_TO_EDGE);
```

<img src="https:////upload-images.jianshu.io/upload_images/1791669-7dcbf01fa2e97a9b.png?imageMogr2/auto-orient/strip|imageView2/2/w/840/format/webp" alt="img" style="zoom:50%;" />

**三分屏和八分屏也是类似，只需要修改矩阵即可**
 修改及效果如下

```java
public static final float[] SPLIT_SCREEN_3_VERTEX_DATA = {

    0f,    0f,  0.5f, 1.5f,
    -1f, -1f,   0f, 3f,
    1f, -1f,   1f, 3f,
    1f,  1f,    1f, 0.0f,
    -1f,  1f,    0f, 0.0f,
    -1f, -1f,    0f, 3f };
```

<img src="https:////upload-images.jianshu.io/upload_images/1791669-337fd11ebe284e74.png?imageMogr2/auto-orient/strip|imageView2/2/w/842/format/webp" alt="img" style="zoom:50%;" />



```java
    public static final float[] SPLIT_SCREEN_8_VERTEX_DATA = {
            // Order of coordinates: X, Y, S, T
            // Triangle Fan
            0f,    0f,  1f, 2f,
            -1f, -1f,    0f, 4f,
            1f, -1f,   2f, 4f,
            1f,  1f,   2f, 0.0f,
            -1f,  1f,    0f, 0.0f,
            -1f, -1f,    0f, 4f };
```

<img src="https:////upload-images.jianshu.io/upload_images/1791669-fed7782185c63f52.png?imageMogr2/auto-orient/strip|imageView2/2/w/834/format/webp" alt="img" style="zoom:50%;" />

**纹理与颜色混合**

上面的分屏、镜像等都是直接针对纹理图片改变顶点着色器的S和T坐标实现。如果想在上面的结果上再和颜色混合该如何做？先上结果



![img](https:////upload-images.jianshu.io/upload_images/1791669-5cbc382f30f61918.png?imageMogr2/auto-orient/strip|imageView2/2/w/836/format/webp)

还是和上面一样的流程
 首先修改着色器，顶点着色器添加color的attribute和varying，然后片元着色器生成gl_FragColor时，乘以颜色的向量_
 然后在GLprograme中拿到color的loaction，顶点着色器的矩阵数据添加rgb值,



```cpp
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
```



```cpp
precision mediump float; 
                        
uniform sampler2D u_TextureUnit;                                        
varying vec2 v_TextureCoordinates;
varying vec3 v_Color;
  
void main()                         
{                               
    gl_FragColor = texture2D(u_TextureUnit, v_TextureCoordinates) * vec4(v_Color,1.0f);
}
```



```java
public static final float[] SPLIT_SCREEN_2_VERTEX_DATA = {
    // Order of coordinates: X, Y, R, G, B, S, T
    // Triangle Fan
    0f,    0f, 1.0f,0.0f,0.0f,  0.5f, 1f,
    -1f, -1f,   1.0f,1.0f,0.0f,  0f, 2f,
    1f, -1f,   0.0f,0.0f,1.0f,  1f, 2f,
    1f,  1f,   0.0f,0.0f,0.0f,  1f, 0.0f,
    -1f,  1f,   1.0f,0.0f,0.0f,  0f, 0.0f,
    -1f, -1f,  1.0f,1.0f,0.0f,   0f, 2f };
```



```java
public class GuangzhouTa {
    private static final int POSITION_COMPONENT_COUNT = 2;
    private static final int COLOR_COMPONENT_COUNT = 3;
    private static final int TEXTURE_COORDINATES_COMPONENT_COUNT = 2;
    private static final int STRIDE = (POSITION_COMPONENT_COUNT + COLOR_COMPONENT_COUNT
                                       + TEXTURE_COORDINATES_COMPONENT_COUNT) * BYTES_PER_FLOAT;


    private final VertexArray vertexArray;

    public GuangzhouTa(float[] vertexData) {
        vertexArray = new VertexArray(vertexData);
    }

    public void bindData(TextureShaderProgram textureProgram) {
        vertexArray.setVertexAttribPointer(
            0, 
            textureProgram.getPositionAttributeLocation(), 
            POSITION_COMPONENT_COUNT,
            STRIDE);

        vertexArray.setVertexAttribPointer(
            POSITION_COMPONENT_COUNT,
            textureProgram.getColorAttributeLocation(),
            COLOR_COMPONENT_COUNT,
            STRIDE);

        vertexArray.setVertexAttribPointer(
            POSITION_COMPONENT_COUNT+COLOR_COMPONENT_COUNT,
            textureProgram.getTextureCoordinatesAttributeLocation(),
            TEXTURE_COORDINATES_COMPONENT_COUNT, 
            STRIDE);
    }

    public void draw() {                                
        glDrawArrays(GL_TRIANGLE_FAN, 0, 6);
    }
}
```

## 12.4 资料

《OpenGL ES 3.0 编程指南》
 《OpenGL编程指南》（红宝书）
 《OpenGL ES应用开发实践指南》

[OpenGL入门第七课--纹理]
 [Android OpenGL ES 2.0绘图:绘制纹理]
 [从0开始的OpenGL学习（五）-纹理]
 [OpenGL纹理详解（上）]
 [OpenGL纹理详解（下）实践篇]
 [OpenGL（十二） 纹理映射（贴图）]
 [OpenGL纹理显示]
 [纹理]

## 12.5 收获

1. 了解纹理坐标、纹理单元、纹理环绕、纹理过滤、mipmap等概念
2. 了解纹理加载流程以及重要API分析
3. 通过实践加载纹理，熟悉概念和流程
4. 纹理倒立等问题分析解决

