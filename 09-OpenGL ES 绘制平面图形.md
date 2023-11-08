# 09-OpenGL ES 绘制平面图形

## 9.0 目录

1. 写着色器代码
2. 通过GLSurfaceview加载Shader并运行
3. 遇到的问题
4. 参考
5. 收获

我们前两篇介绍了[OpenGL ES 基本概念](https://www.jianshu.com/p/1cd9682148af)和[GLSL及Shader的渲染流程](https://www.jianshu.com/p/e1a2b3d96e38)，这篇我们开始实战，通过GLSurfaceView加载着色器，来绘制三角形、正方形和直线这些平面图形。在实践过程中遇到的问题有时候让人没有头绪，检查了一遍又一遍代码，发现流程没有问题，但屏幕就是一片漆黑。。通过近一个小时的排查，发现问题出在了这里。。。下面开始我们今天的学习时间之旅，希望对你也有帮助。

## 9.1 GLSL着色器的编写

如果对OpenGL的基本概念以及渲染流程不清晰的，建议看下前两篇文章，这些基本概念和流程要了解或者理解，否则后面实践之旅就是跳坑之旅。

我们先通过下面重要的二张图，快速回顾下



![img](https:////upload-images.jianshu.io/upload_images/1791669-0b517d67b39c1ded.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

![img](https:////upload-images.jianshu.io/upload_images/1791669-3a46f5e12972f8b5.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

工欲善其事，必先利其器，如何方便的编写GLSL代码呐？ AndoridStudio提供了“Support for GLSL”插件。VS Code也有比较强大的插件比如“Shader Toy
 ”和“Shader languages support for VS Code”。但是都不足之处，就是没有自动提示和补全的功能。所以编写GLSL代码是要细心。

### 9.1.1  着色器代码的编写

首先我们来编写下顶点着色器和片元着色器



```cpp
//vertex_shader.glsl 顶点着色器

attribute vec4 a_Position;
attribute vec4 a_Color;

varying vec4 v_Color;

void main() {
    v_Color = a_Color;
    gl_Position = a_Position;
}
```

**上述代码简单语法回顾**



```undefined
attribute是修饰符 只能用于顶点着色器，用于修饰可变的参数
vec4: 浮点型向量

gl_Position：内置变量

varying：也是一个修饰符，用于顶点着色器和片元着色器的值传递。
注意：必须要在顶点着色器和片元着色器都定义同名同类型同varying修饰符的变量，才能正常传递。
```



```cpp
//fragment_shader.glsl 片元着色器

precision mediump float;

varying vec4 v_Color;

void main() {
    gl_FragColor = v_Color;
}
```

这里对精度precision mediump 做下说明
 用于修饰浮点型和整形，有三个等级 highp\mediump、lowp



![img](https:////upload-images.jianshu.io/upload_images/1791669-eef69a1bb77b7472.png?imageMogr2/auto-orient/strip|imageView2/2/w/903/format/webp)

### 9.1.2  着色器代码的读取

着色器代码通常放在assets目录下或者raw目录下，当然也见到过直接写在代码里。我们为了方便，采用比较通用的方式：把glsl代码文件放在了assets下，再加载前需要先把他们读到内存中，常规的文件读写



```csharp
public static String loadAsset(Resources res, String path) {
     StringBuilder stringBuilder = new StringBuilder();
     try {
         InputStream is = res.getAssets().open(path);

         byte[] buffer = new byte[1024];
         int count;
         while (-1 != (count = is.read(buffer))) {
             stringBuilder.append(new String(buffer, 0, count));
         }
         String result = stringBuilder.toString().replaceAll("\\r\\n", "\n");
        return result;
     } catch (IOException e) {
         e.printStackTrace();
     }
     return "";
 }
```

### 9.1.3  着色器的创建、设置源码、编译

![img](https:////upload-images.jianshu.io/upload_images/1791669-27cb9b04f6c12e21.jpeg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



```cpp
private static int loadShader(int type, String codeStr) {
        //1. 根据类型（顶点着色器、片元着色器）创建着色器，拿到着色器句柄
        int shader = GLES20.glCreateShader(type);
        Log.i(TAG, "compileShaderCode: type=" + type + " shaderId=" + shader);

        if (shader > 0) {
            //2. 设置着色器代码 ，shader句柄和code进行绑定
            GLES20.glShaderSource(shader, codeStr);
            //3. 编译着色器，
            GLES20.glCompileShader(shader);

            //4. 查询编译状态
            int[] status = new int[1];
            GLES20.glGetShaderiv(shader, GLES20.GL_COMPILE_STATUS, status, 0);
            Log.i(TAG, "loadShader: status[0]=" + status[0]);
            //如果失败，释放资源
            if (status[0] == 0) {
                GLES20.glDeleteShader(shader);
                return 0;
            }
        }
        return shader;
    }
```

### 9.1.4 程序的创建、attach着色器、链接、使用



```cpp
public static int loadProgram(String verCode, String fragmentCode) {
        //1. 创建Shader程序，获取到program句柄
        int programId = GLES20.glCreateProgram();
        if(programId == 0){
            Log.e(TAG, "loadProgram: glCreateProgram error" );
            return 0;
        }
        //2. 根据着色器语言类型和代码，attach着色器
        GLES20.glAttachShader(programId, loadShader(GLES20.GL_VERTEX_SHADER, verCode));
        GLES20.glAttachShader(programId, loadShader(GLES20.GL_FRAGMENT_SHADER, fragmentCode));
        //3. 链接
        GLES20.glLinkProgram(programId);
        //4. 使用
        GLES20.glUseProgram(programId);
        return programId;
    }
```

### 9.1.5 状态查询

1. 着色器Shader和Program创建后会拿到对应的句柄，通过检查是否大于0验证是否可用
2. GLES20.glGetShaderiv(shader, GLES20.GL_COMPILE_STATUS, status, 0);着色器编译后检查编译的状态是否大于0判断可用性。
3. glGetProgramiv(programObjectId, GL_VALIDATE_STATUS,
    validateStatus, 0);//程序链接后，检查程序的可用性

### 9.1.6   输入与输出

输入：给着色器赋值
 输出：在屏幕上显示

前面5个步骤把准备工作都做好了，那边现在面临两个问题.

**1. 我们看到顶点着色器中有两个attribute修饰的变量，如何给它们赋值?**
 **2. 如何把着色器再屏幕上绘制出来？**

这个环节我们就来解决这两个问题

**首先定义好顶点坐标和颜色的位数和着色器的数据**



```java
    //每个顶点坐标的个数
    private final static int COORDS_PER_VERTEX = 2;

    //每个顶点颜色的个数
    private final static int COLOR_PER_VERTEX = 3;
    
    // 浮点类型占用的字节数
    private final static int BYTES_PER_FLOAT = 4;
    
    //下面两个字符串常量就是GLSL顶点着色器的输入

    private static final String A_POSITION = "a_Position";
    private static final String A_COLOR = "a_Color";

    //STRIDE是一个顶点的字节偏移（顶点坐标xy+颜色rgb）
    private final int STRIDE = (COORDS_PER_VERTEX+ COLOR_PER_VERTEX )* BYTES_PER_FLOAT;

    private FloatBuffer mVertexData;

    public MyRender2() {
        //顶点数组
        float[] TRIANGLE_COORDS = {
                0.5f, 0.5f,1f, 0.5f,0.5f,
                -0.5f, -0.5f,0.5f, 1f,0.5f,
                0.5f, -0.5f,0.5f, 0.5f,1f

        };

       //通过nio ByteBuffer把设置的顶点数据加载到内存
        mVertexData = ByteBuffer
                .allocateDirect(TRIANGLE_COORDS.length * BYTES_PER_FLOAT) //需要多少字节内存
                .order(ByteOrder.nativeOrder())//大小端排序
                .asFloatBuffer()
                .put(TRIANGLE_COORDS);//设置数据
    }
```

**然后给顶点着色器的变量赋值**



```dart
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
```

**关键API说明**

1. 获取着色器attribute一个属性：int aPosition = GLES20.glGetAttribLocation(programId, A_POSITION);_
2. 数据的偏移：mVertexData.position(0); 因为有坐标和颜色两个变量的值，数据又是根据顶点一一设定的。
3. 给attribute赋值：GLES20. glVertexAttribPointer(
    int indx,//attribute的句柄
    int size,//在数组中占用的位数
    int type,//数据的类型
    boolean normalized,
    int stride,//步幅 单位字节数
    java.nio.Buffer ptr // 元数据
    )
4. 使能对应的attribute属性：GLES20.glEnableVertexAttribArray(aPosition);

通过上面几个环节我们可以看到，我们可以看到，是如何给着色器语言中的变量赋值的。下面我们看来下如何渲染。

我们采用GlSurfaceView，通过Render来实现，具体如下：



```java
    //1. 设置OpenGL ES的版本
    glSView.setEGLContextClientVersion(3);

     //2. 给glSurfaceView设置render
    glSView.setRenderer(new MyRender2());

  
public class MyRender2 implements GLSurfaceView.Renderer {

@Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
      //着色器的加载、赋值
    }

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        GLES20.glViewport(0, 0, width, height);
    }

    @Override
    public void onDrawFrame(GL10 gl) {
        //清屏
        GLES20.glClear(GL_COLOR_BUFFER_BIT);
        
        //绘制
        GLES20.glDrawArrays(GLES20.GL_TRIANGLE_FAN,0,3);
    }
}

是的，在onDrawFrame中进行不断的 glDrawArrays来绘制刷新   
 public static native void glDrawArrays(
        int mode, //点、线、三角形
        int first,//顶点的第一个数据的index
        int count//顶点的总数
    );
```

## 9.2 实践：用GLSurfaceView加载GLSL绘制屏幕图形

### 9.2.1 三角形

上面的代码中定义的就是三角形，对应顶点数据如下



```bash
float[] TRIANGLE_COORDS = {0.5f, 0.5f,1f, 0.5f,0.5f,
                -0.5f, -0.5f,0.5f, 1f,0.5f,
                0.5f, -0.5f,0.5f, 0.5f,1f

        };

两个坐标位x&y,和三个颜色位rgb
```

效果如下

<img src="https:////upload-images.jianshu.io/upload_images/1791669-badc86ff6c9d5316.png?imageMogr2/auto-orient/strip|imageView2/2/w/1080/format/webp" alt="img" style="zoom:50%;" />

### 9.2.2 正方形

只需要修改 顶点数组和glDrawArrays的count参数即可



```cpp
float[] TRIANGLE_COORDS = {
               0.5f, 0.5f,1f, 0.5f,0.5f,
               -0.5f, -0.5f,0.5f, 1f,0.5f,
               0.5f, -0.5f,0.5f, 0.5f,1f,
               0.5f, 0.5f,1f, 0.5f,0.5f,
               -0.5f, 0.5f,0.5f, 0.5f,1f,
               -0.5f, -0.5f,0.5f, 1f,0.5f,
       };


  public void onDrawFrame(GL10 gl) {
       GLES20.glClear(GL_COLOR_BUFFER_BIT);
       GLES20.glDrawArrays(GLES20.GL_TRIANGLES,0,6);
   }
```

<img src="https:////upload-images.jianshu.io/upload_images/1791669-eef5fec63a25aa0a.png?imageMogr2/auto-orient/strip|imageView2/2/w/1080/format/webp" alt="img" style="zoom:50%;" />

### 2.3  直线



```cpp
     float[] TRIANGLE_COORDS = {
            -0.5f, 0f, 1f, 0f, 0f,
             0.5f, 0f, 0f, 0f, 1f,
        };

    public void onDrawFrame(GL10 gl) {
        GLES20.glClear(GL_COLOR_BUFFER_BIT);
        GLES20.glDrawArrays(GLES20.GL_LINES,0,2);
    }
```

<img src="https:////upload-images.jianshu.io/upload_images/1791669-618a65dd9613c922.png?imageMogr2/auto-orient/strip|imageView2/2/w/1080/format/webp" alt="img" style="zoom:50%;" />

通过log分析，我们发现GLSurfaceView.Renderer是运行中一个叫GLThread的线程中，它的作用和意义是什么，下一篇我们通过对GLSurfaceView的源码分析来理解EGL和GLThread，了解完整的流程，然后不使用Render，采用自建管理EGL和创建GLThread，通过TextureView实现图形的绘制。做到知其然，也知其所以然。

## 9.3 遇到的问题



```undefined
1. A/libc: Fatal signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x0 in tid 9940 (GLThread 6703)_

发现是GLES20.glCreateProgram()时出现的上述崩溃
原因是因为没有glSView.setEGLContextClientVersion(3);
```



```bash
2. GLES20.glGetShaderiv(shader, GLES20.GL_COMPILE_STATUS, status, 0)
Shader编译后获取状态为0（失败了）

原因是因为glsl语言注释不对用成了 # 而不是 //
```



```cpp
3.  无法正常的看到期望的图片
这个就是开头说的说的那个折腾了一个小时的问题，一个“逗号”引发的问题

具体如下：

float[] TRIANGLE_COORDS = {0.5f, 0.5f,1f, 0.5f,0.5f ///注意这里没有加逗号，就是这个导致的，glsl认为到这里就结束了。。但是代码上写的是3个顶点，找不到就无法正常绘制。多么痛的领悟。
                -0.5f, -0.5f,0.5f, 1f,0.5f,
                0.5f, -0.5f,0.5f, 0.5f,1f

        };
```

## 9.4 参考

《OpenGL ES应用开发实践指南》
 《音视频开发进阶指南》
 [搭建OpenGL ES环境的两种方式]
 [Android OpenGL ES(一)-开始描绘一个平面三角形]
 [Android OpenGL ES(三)-平面图形]
 [EGL 环境搭建流程]

## 9.5 收获

1. 通过代码实践加深了对GLSL语法，OpenGL基本概念和绘制流程的熟悉。
2. glsl程序编写androidstudio等IDE插件的了解
3. 理解实现了如何给着色器输入数据，又如何在屏幕上绘制。
4. 绘制三角形、正方形、直线等平面图形
5. 遇到的问题分析解决，弥补认知不足。