# 11-OpenGL ES矩阵变换与坐标系统

## 11.0 目录

1. 矩阵与矩阵变换
2. 坐标系统
3. OpenGL的矩阵与矩阵变换
4. 实践：平移、旋转、缩放、3D
5. 资料
6. 收获

## 11.1 矩阵与矩阵变换

### 11.1.1  矩阵的基本知识回顾

OpenGL大量使用向量和矩阵，矩阵的最重要的用途之一就是建立向量投影（比如：正交和透视投影）、使物体旋转（rotation）、平移（translation）以及缩放（scaling）。下面我们来介绍下几个常用的矩阵类型。

**单位矩阵**

![img](https:////upload-images.jianshu.io/upload_images/1791669-457de022170347ce.png?imageMogr2/auto-orient/strip|imageView2/2/w/224/format/webp)



**缩放矩阵**

![img](https:////upload-images.jianshu.io/upload_images/1791669-90912db7b82159b8.png?imageMogr2/auto-orient/strip|imageView2/2/w/220/format/webp)



**旋转矩阵**

![img](https:////upload-images.jianshu.io/upload_images/1791669-1c6ea659ccfac3d2.png?imageMogr2/auto-orient/strip|imageView2/2/w/402/format/webp)



**平移矩阵**

![img](https:////upload-images.jianshu.io/upload_images/1791669-dc47bcc9a61236e4.png?imageMogr2/auto-orient/strip|imageView2/2/w/370/format/webp)



**矩阵乘法**
 矩阵变换时用到的乘法是左乘，不符合交换律，但遵循结合定律
 AB  != BA
 C(BA)=(CB)A=CBA
 C(B(Av))=(CBA)v

![img](https:////upload-images.jianshu.io/upload_images/1791669-cf7b125b27f487d0.png?imageMogr2/auto-orient/strip|imageView2/2/w/758/format/webp)

**正交投影矩阵**

![img](https:////upload-images.jianshu.io/upload_images/1791669-fc752b7095dc58a1.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/660/format/webp)



![img](https:////upload-images.jianshu.io/upload_images/1791669-d45ad6f5701c3330.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

**投影矩阵**

![img](https:////upload-images.jianshu.io/upload_images/1791669-21b2e064235c632f.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/507/format/webp)



![img](https:////upload-images.jianshu.io/upload_images/1791669-76fce02a8fcb2c20.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

**齐次坐标**
 什么是齐次坐标？

3维数据可以通过3维向量与3x3矩阵的乘法操作，来完成缩放和旋转的线性变换。但是对于笛卡尔坐标（欧氏坐标）的平移操作是时加法操作，这样无办法做到变换的线性连续性。这就引入了齐次坐标，齐次坐标用N+1维来代表N维坐标。在3D笛卡尔坐标尾部加上一个额外的变量就变成了3D齐次坐标
 一个点（X,Y,Z）在齐次坐标中变成了（x,y,z,w）
 并且 X= x/w；   Y= y/w;   Z=z/w
 引入齐次坐标后平移也可以通过矩阵相乘的来表示了，保证了形式上的线性一致性，从而做到不管怎么样变换，变换多少次，都可以表示成一连串的矩阵相乘，真是太方便了。
 在OpenGL中对于3维数据引入的第四分量事实就是用来实现透视投影变换的。

齐次坐标的作用，合并矩阵运算中的乘法和加法，把各种变换都统一了起来，即 把缩放，旋转，平移等变换都统一起来，都表示成一连串的矩阵相乘的形式。保证了形式上的线性一致性

## 11.2 坐标系统

### 11.2.1  左手坐标和右手坐标系统

![img](https:////upload-images.jianshu.io/upload_images/1791669-49a1d9ddabdcd11d.png?imageMogr2/auto-orient/strip|imageView2/2/w/890/format/webp)

大拇指指向x轴正方向、其余是个手指指向y轴正方向，分别用左手和右手来向内握拳就表示上述的不同的坐标系。归一化设备坐标使用的左手坐标系统，而OpenGL早期的版本使用的却是右手系统，这也是Android的Matrix会默认生成反转z的矩阵的原因。

### 11.2.2  物体坐标系（建模坐标系）

是一个局部坐标系，为了模型与变换的方便。比如，在创建圆形时，一般将圆心作为参考点来创建圆周上的各个点，实际上就是构建了一个以圆心为原点的参考坐标系。

### 11.2.3 世界坐标系

对物体建模之后，下一步是将各个图形组合放到绘制平面中，为了确定每个图形的位置，必须放弃各自的物体坐标系（建标坐标系），将其纳入一个统一的坐标系，这个坐标系就称为世界坐标系

### 11.2.4 眼睛坐标系（观察坐标系）

用户可根据图形显示的要求定义观察区域和观察方向，即要定义视点（或者成为照相机）的位置和方向，从观察者的角度对整个世界坐标系内的图形进行重新定位和描述。

### 11.2.5 归一化设备坐标系（规范化设备坐标系）

为了使观察处理独立于输出设备，将观察坐标系的对象描述转换到一个中间坐标系，这个坐标系即独立于设备，又可以容易地转变为设备坐标系。其坐标范围为【0，1】提高应用的可移植性。

### 11.2.6 设备坐标系（屏幕坐标系）

主要用于某一特殊的计算机图形显示设备表面的像素定义，对于每个具体的显示设备，都有一个单独的坐标系。

## 11.3 OpenGL的矩阵变换

**回顾、学习了上述基础知识后，我们来看下OpenGL的矩阵变换**

![img](https:////upload-images.jianshu.io/upload_images/1791669-c5ba5e2f39abd8c0.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/738/format/webp)

![img](https:////upload-images.jianshu.io/upload_images/1791669-5812fdc280970f8f.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1063/format/webp)

**顶点着色器的位置输入保存为物体坐标，而输出位置保存为裁剪坐标。**

模型—视图—投影（MVP）矩阵
 模型矩阵——将物体坐标变换为世界坐标
 视图矩阵——将世界坐标变换为眼睛坐标
 投影矩阵——将眼睛坐标变换为裁剪坐标（齐次坐标）

在传统的OpenGL功能中，模型和试图合并为一个矩阵，成为模型-视图矩阵，将物体坐标变换为眼睛坐标。空间中任意位置和任何想要的方向都可以由一个4X4矩阵唯一确定，并且如果用一个对象的所有向量乘以这个矩阵，那么我们就将整个对象变换到了空间中的给定位置和方向。可以用glRotatef、glTranslatef、glScalef等函数创建，需要应用程序自己处理

> 视图变换
>  视图变换允许我们把观察点放在所希望的任何位置，并允许在任何方向上观察场景。确定视图变换就像在场景中放置照相机并让它指向某个方向。
>
> 模型变换
>  模型变换用于操纵模型和其中的特定对象。这些变换将对象移动到需要的位置，然后再对它们进行旋转和缩放。

> 投影变换
>  投影变换将在模型视图变换之后应用到顶点上，它将指定一个完成的场景（所有模型变换都已完成）是如何投影到屏幕上的最终图像。
>  正投影：所有多边形都是精确地按照指定的相对大小来在屏幕上绘制的。
>  透视投影：透视投影的特点是透视缩短，这种特性使得远处的物体看起来比进出同样大小的物体更小一些。
>
> 视口变换
>  当所有变换完成后，就得到了一个场景的二维投影，它将被映射到屏幕上某处的窗口上。这种到物理窗口坐标的映射是我们最后要做的变换，称为视口变换。

![img](https:////upload-images.jianshu.io/upload_images/1791669-f765520456c546b5.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

图片来自《OpenGL编程指南》（红宝书）

## 11.4 实践

### 11.4.1 平移、旋转、缩放



```cpp
//定义顶点着色器

//定义一个不经常更改的mat4矩阵
uniform mat4 u_Matrix;

attribute vec4 a_Position;
attribute vec4 a_Color;

varying vec4 v_Color;

void main() {
    v_Color = a_Color;
    //gl_Position的赋值采用矩阵和a_Position相乘
    gl_Position = u_Matrix * a_Position;
}
```

Render关键代码

```java
    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        Log.i(TAG, "onSurfaceCreated: curThread= " + Thread.currentThread());
        glClearColor(0.0f, 0.0f, 0.0f, 0.0f);

        String vertexCode = ShaderHelper.loadAsset(MyApplication.getContext().getResources(), "vertex_shader.glsl");
        String fragmentCode = ShaderHelper.loadAsset(MyApplication.getContext().getResources(), "fragment_shader.glsl");
        //创建着色器程序
        programId = ShaderHelper.loadProgram(vertexCode, fragmentCode);

        uMatrixLocation = GLES20.glGetUniformLocation(programId, U_MATRIX);

        int aPosition = GLES20.glGetAttribLocation(programId, A_POSITION);
        Log.i(TAG, "drawFrame: aposition="+aPosition);
        mVertexData.position(0);
        GLES20.glVertexAttribPointer(aPosition,
                COORDS_PER_VERTEX,//用几个偏移描述一个顶点
                GLES20.GL_FLOAT,//顶点数据类型
                false,
                STRIDE,//一个顶点需要多少个字节偏移
                mVertexData//分配的buffer
        );

        //开启顶点着色器的attribute
        GLES20.glEnableVertexAttribArray(aPosition);

        int aColor = GLES20.glGetAttribLocation(programId, A_COLOR);
        mVertexData.position(COORDS_PER_VERTEX);
        GLES20.glVertexAttribPointer(aColor,COLOR_PER_VERTEX,GL_FLOAT,false,STRIDE,mVertexData);
        GLES20.glEnableVertexAttribArray(aColor);
    }

    @Override
    public void onDrawFrame(GL10 gl) {
        Log.i(TAG, "onDrawFrame: " + "curThread= " + Thread.currentThread());
        GLES20.glClear(GL_COLOR_BUFFER_BIT);

        GLES20.glUniformMatrix4fv(uMatrixLocation, 1, false, modelMatrix, 0);

        GLES20.glDrawArrays(GLES20.GL_TRIANGLES,0,6);



    }
```

下面的平移、旋转、缩放等都是再onSurfaceChanged进行模型矩阵的赋值。

**原图**

<img src="https:////upload-images.jianshu.io/upload_images/1791669-82aa6a7d386edd05.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/914/format/webp" alt="img" style="zoom:50%;" />



**乘以单位矩阵**效果是和原来一模一样的



```css
Matrix.setIdentityM(modelMatrix,0);
```

<img src="https:////upload-images.jianshu.io/upload_images/1791669-82aa6a7d386edd05.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/914/format/webp" alt="img" style="zoom:50%;" />

**进行缩放**



```css
  Matrix.setIdentityM(modelMatrix,0);
  Matrix.scaleM(modelMatrix,0,0.5f,0.5f,0);
```

<img src="https:////upload-images.jianshu.io/upload_images/1791669-b0974b59e92a7179.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/924/format/webp" alt="img" style="zoom:50%;" />

**进行旋转**， 没有设置投影和z方向的偏移，所以还是一个平面图



```css
 Matrix.setIdentityM(modelMatrix,0);
 Matrix.rotateM(modelMatrix,0,-60,1,0,0);
```

![img](https:////upload-images.jianshu.io/upload_images/1791669-2522fc64b906a3f0.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/598/format/webp)

**进行平移**



```css
Matrix.setIdentityM(modelMatrix,0);
Matrix.translateM(modelMatrix,0,0.3f,0.3f,0);
```

<img src="https:////upload-images.jianshu.io/upload_images/1791669-6febd37a3d71931c.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/930/format/webp" alt="img" style="zoom:50%;" />

### 11.4.2 通过添加w实现3D效果



```java
private final static int COORDS_PER_VERTEX = 4;

float[] TRIANGLE_COORDS = {
               // x,y,z,w.   r,g,b
                0.5f, 0.5f,0,5f,   1f, 0.5f,0.5f,
                -0.5f, -0.5f,0, 1f,  0.5f, 1f,0.5f,
                0.5f, -0.5f,0,1f,   0.5f, 0.5f,1f,
                0.5f, 0.5f,0,5f,   1f, 0.5f,0.5f,
                -0.5f, 0.5f,0,5f,   0.5f, 0.5f,1f,
                -0.5f, -0.5f,0,1f,   0.5f, 1f,0.5f,
}
```

效果如下：



<img src="https:////upload-images.jianshu.io/upload_images/1791669-504457a57881ac64.png?imageMogr2/auto-orient/strip|imageView2/2/w/1080/format/webp" alt="img" style="zoom:50%;" />

Render的完整代码如下：



```java
public class MyRender implements GLSurfaceView.Renderer {
    private static final String TAG = "MyRender";
    private int programId;

    //每个顶点坐标的个数
    private final static int COORDS_PER_VERTEX = 4;

    //每个顶点颜色的个数
    private final static int COLOR_PER_VERTEX = 3;

    private final static int BYTES_PER_FLOAT = 4;

    private static final String A_POSITION = "a_Position";
    private static final String A_COLOR = "a_Color";

    //-个float占用4个字节，STRIDE是一个点的字节偏移
    private final int STRIDE = (COORDS_PER_VERTEX+ COLOR_PER_VERTEX )* BYTES_PER_FLOAT;

    private FloatBuffer mVertexData;


    public MyRender2() {
        //顶点坐标数组
        float[] TRIANGLE_COORDS = {
                0.5f, 0.5f,0,5f,   1f, 0.5f,0.5f,
                -0.5f, -0.5f,0, 1f,  0.5f, 1f,0.5f,
                0.5f, -0.5f,0,1f,   0.5f, 0.5f,1f,
                0.5f, 0.5f,0,5f,   1f, 0.5f,0.5f,
                -0.5f, 0.5f,0,5f,   0.5f, 0.5f,1f,
                -0.5f, -0.5f,0,1f,   0.5f, 1f,0.5f,

        };
        mVertexData = ByteBuffer
                .allocateDirect(TRIANGLE_COORDS.length * BYTES_PER_FLOAT)
                .order(ByteOrder.nativeOrder())
                .asFloatBuffer()
                .put(TRIANGLE_COORDS);
    }

    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        Log.i(TAG, "onSurfaceCreated: curThread= " + Thread.currentThread());
        glClearColor(0.0f, 0.0f, 0.0f, 0.0f);

        String vertexCode = ShaderHelper.loadAsset(MyApplication.getContext().getResources(), "vertex_shader.glsl");
        String fragmentCode = ShaderHelper.loadAsset(MyApplication.getContext().getResources(), "fragment_shader.glsl");
        //创建着色器程序
        programId = ShaderHelper.loadProgram(vertexCode, fragmentCode);


        int aPosition = GLES20.glGetAttribLocation(programId, A_POSITION);
        Log.i(TAG, "drawFrame: aposition="+aPosition);
        mVertexData.position(0);
        GLES20.glVertexAttribPointer(aPosition,
                COORDS_PER_VERTEX,//用几个偏移描述一个顶点
                GLES20.GL_FLOAT,//顶点数据类型
                false,
                STRIDE,//一个顶点需要多少个字节偏移
                mVertexData//分配的buffer
        );

        //开启顶点着色器的attribute
        GLES20.glEnableVertexAttribArray(aPosition);

        int aColor = GLES20.glGetAttribLocation(programId, A_COLOR);
        mVertexData.position(COORDS_PER_VERTEX);
        GLES20.glVertexAttribPointer(aColor,COLOR_PER_VERTEX,GL_FLOAT,false,STRIDE,mVertexData);
        GLES20.glEnableVertexAttribArray(aColor);
    }

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        Log.i(TAG, "onSurfaceChanged: width=" + width + " h=" + height + "curThread= " + Thread.currentThread());
        GLES20.glViewport(0, 0, width, height);
    }

    @Override
    public void onDrawFrame(GL10 gl) {
        Log.i(TAG, "onDrawFrame: " + "curThread= " + Thread.currentThread());
        GLES20.glClear(GL_COLOR_BUFFER_BIT);
        GLES20.glDrawArrays(GLES20.GL_TRIANGLES,0,6);

    }
}
```

但是w的值是在代码中写死的，下面我们透过投影矩阵的方式自动生成w值，为动态的改变做好铺垫。

### 11.4.3  投影矩阵实现3D

首选需要修改顶点着色器，gl_position的赋值不是完全通过glVertexAttribPointer，而是借用矩阵相乘的方式实现。具体顶点着色器代码如下：_



```cpp
//定义一个不经常更改的mat4矩阵
uniform mat4 u_Matrix;

attribute vec4 a_Position;
attribute vec4 a_Color;

varying vec4 v_Color;

void main() {
    v_Color = a_Color;
    //gl_Position的赋值采用矩阵和a_Position相乘
    gl_Position = u_Matrix * a_Position;
}
```

下面我们在Render中给u_Matrix矩阵赋值_
 首先定义四个矩阵变量和matlocatin



```java
    private final float[] projectionMatrix = new float[16]; //4*4的投影矩阵
    private final float[] modelMatrix = new float[16]; //4*4的模型矩阵
    private final float[] viewMatrix = new float[16];//4*4d的视图矩阵
    private final float[] mvpMatrix = new float[16];//4*4d的MVP矩阵

    private int uMatrixLocation;//glsl中的uniform mat4变量的location
```

然后在onSurfaceChanged 给上述矩阵赋值



```java
    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        Log.i(TAG, "onSurfaceChanged: width=" + width + " h=" + height + "curThread= " + Thread.currentThread());
        GLES20.glViewport(0, 0, width, height);

        //设置模型矩阵 即M
        Matrix.setIdentityM(modelMatrix,0);
        Matrix.translateM(modelMatrix,0,0,0,-2.5f);
        Matrix.rotateM(modelMatrix,0,-60,1,0,0);

        //设置视图矩阵 即V
        Matrix.setLookAtM(viewMatrix,0,0,0,1,0,0,0,1,0,0);

        //设置投影矩阵 即P
        Matrix.perspectiveM(projectionMatrix,0,60,(float) width/ height,1f,200f);

  Matrix.multiplyMM(projectionMatrix,0,projectionMatrix,0,modelMatrix,0);
        Matrix.multiplyMM(mvpMatrix,0,projectionMatrix,0,viewMatrix,0);
    }
```

最后再onDrawFrame中调用



```bash
        GLES20.glUniformMatrix4fv(uMatrixLocation, 1, false, mvpMatrix, 0);
```

效果如下：



<img src="https:////upload-images.jianshu.io/upload_images/1791669-dd6957b251dc762d.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/938/format/webp" alt="img" style="zoom:50%;" />

## 11.5 资料

《计算机图形学基础（OpenGL版）》
 《OpenGL ES 3.0 编程指南》
 《OpenGL编程指南》（红宝书）
 《OpenGL ES应用开发实践指南》
 [OpenGL 矩阵变换（讲的太好了~！）]
 [ songho ]
 [为什么要引入齐次坐标，齐次坐标的意义]
 [OpenGL入门第三课--矩阵变换与坐标系统]
 [Android OpenGL ES(二)-正交投影]
 [OpenGL坐标变换]
 [OpenGL坐标系解析(顶点从对象坐标系到屏幕坐标系的计算流程)]
 [OpenGL中各种坐标系的理解]
 [OpenGL透视投影下的模型视图矩阵/投影矩阵/观察者矩阵]

## 11.6 收获

这篇文章的信息量很大，最重要的是对概念的理解清楚，而代码实现上各个平台都有提供封装比较好的方法比如Matrix。

1. 回顾了矩阵的基本知识（单位矩阵、正交矩阵、投影矩阵、矩阵相乘规则等）和左右手坐标，
2. 学习了很多新的概念，比如世界坐标、齐次坐标、视口坐标、归一化坐标、设备坐标、视图变换、模型变换、投影变换、视口变换等。为了搞懂这些基本的概念查看了多本书籍，才把这些基本概念搞明白。
3. 介绍android中Matrix类的API
4. 实践了3D效果、实现平移、缩放、旋转。

 