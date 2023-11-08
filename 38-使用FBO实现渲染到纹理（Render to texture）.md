# 38-使用FBO实现渲染到纹理（Render to texture）

## 38.0 目录

1. FBO基本知识
2. FBO实现渲染到纹理的流程
3. 实践
4. 遇到的问题
5. 资料
6. 收获

在之前的学习实践中我们把图片、视频、图形等渲染到屏幕时，采用的是直接屏幕上即默认的帧缓冲区，如果我们在渲染时不想直接渲染到屏幕，而是把一些列的filter处理好之后，在渲染到屏幕上，
 比如，绘制一个图形，然后给这个图形/片依次做特效1、特效2… 然后在渲染到屏幕上。
 再比如，从camera采集到的视频数据，不直接渲染到屏幕上，而是经过美颜、滤镜、特效等处理后再在屏幕上渲染
 这时我们就需要用FBO的技术，先把素材渲染到纹理，然后针对纹理链式的依次进行离屏渲染，最终再把数据copy到屏幕缓冲区进行渲染显示。

首先我们来了解下FBO的基础知识。

## 38.1 基本知识

![img](https:////upload-images.jianshu.io/upload_images/1791669-0ab89d9881413093.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

OpenGL在图元光栅化，得到的是fragment，fragment不是最后的像素数据，但和像素对应；fragment需要经过一系列处理，blend，texture，lighting...，才会得到最后的像素。用来缓存fragment数据的缓冲区，就是frame buffer。frame buffer包含color buffer，stencil buffer，depth buffer等若干buffer。只有color buffer用于最后的像素显示，其他的都是用来辅助fragment的处理。

屏幕渲染的过程中把上述的colorBuffer等渲染到屏幕上，处理直接渲染之外，OpenGL ES也提供一种被广泛使用的离屏渲染方案，即把渲染目标重定向到非屏幕的其他存储空间，比如将渲染目标重定向到纹理空间，实现渲染到纹理功能（Render to Texture），针对这个纹理可以做各种filter处理。还有一个特点就是图片屏幕的限制。不论是直接渲染到屏幕还是进行离屏渲染，都需要创建震缓冲区对象即FBO，只不过直接渲染到屏幕的FBO的GL_FRAMEBUFFER_BINDING为0。渲染到其他存储空间的frambuffer的id大于0.

FBO（Frame Buffer Object）帧缓冲对象提供了与颜色缓冲区（color buffer）、深度缓冲区（depth buffer）和模版缓冲区（stencil buffer） ，但并不会直接为这些缓冲区分配空间，而只是为这些缓冲区提供一个或多个挂接点。我们需要分别为各个缓冲区创建对象，申请空间，然后挂接到相应的挂接点上。FBO提供的挂接点如下图所示



![img](https:////upload-images.jianshu.io/upload_images/1791669-cd5177f5c4ee5971.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

图片来自：[OpenGL Frame Buffer Object (FBO)](https://links.jianshu.com/go?to=http%3A%2F%2Fwww.songho.ca%2Fopengl%2Fgl%5C_fbo.html)
 能够与FBO挂接的对象有两种，一种是纹理对象（texture object），另一种是渲染缓冲区对象（renderbuffer object）

## 38.2 使用FBO实现渲染到纹理的流程

我们根据挂在FBO上的挂载的不同对象来分别看下流程（类似）

**纹理对象**
 首先通过调用glGenFrameBuffers（）分配一个程序创建未使用的帧缓存对象标示



```cpp
    // C function void glGenFramebuffers ( GLsizei n, GLuint *framebuffers )

    public static native void glGenFramebuffers(
        int n,//分配多少个未使用的帧缓存对象
        int[] framebuffers,//分配的帧缓存对象id存放在数组中
        int offset//偏移
    );
```

然后通过glBindFramebuffer（）绑定FBO，并初始化



```java
    // C function void glBindFramebuffer ( GLenum target, GLuint framebuffer )

    public static native void glBindFramebuffer(
        int target,
        int framebuffer
    );

参数说明：
target 可以为GLES20.GL_FRAMEBUFFER，在OpenGLES3.0后   GL_READ_FRAMEBUFFER 和 GL_DRAW_FRAMEBUFFER两种 只读或者只写的framebuffer类型

framebuffer 为 glGenFramebuffers分配到帧缓存id
```

> 使用glBindFramebuffer()进行离屏渲染时，Opengl的渲染和读取都是通过attached的纹理对帧缓存进行操作的，不再是对windows系统提供的默认帧缓存进行操作，所以我们见到的屏幕上显示出来的图像并不是一个可见的颜色缓存位面（visible color buffer bitplane），只不过是一个“离屏”的颜色纹理（"off-screen" color image attachment），所以双缓存就不起作用了（即使调用swapbuffer(),像素也不会被渲染到前后缓存中），所glReadBuffer(GL_FRONT)也就读取不出像素出来。
>  帧缓存可以实现理屏渲染技术、纹理贴图的更新、以及缓存乒乓技术（GPGPU用到的一种数据传输技术）的实现非常的有意义，但需要注意的是，应用程序创建的帧缓存是不能与窗口系统的缓存关联的，窗口系统有一套自己的缓存对象。

如果挂在的是颜色缓冲区（color buffer），采用纹理对象的形式进行挂载，对应的挂载方法是glFramebufferTexture2D（）



```java
    // C function void glFramebufferTexture2D ( GLenum target, GLenum attachment, GLenum textarget, GLuint texture, GLint level )

    public static native void glFramebufferTexture2D(
        int target,
        int attachment,
        int textarget,
        int texture,
        int level
    );

参数说明：
attachment 必须为 GL_COLOR_ATTACHMENTi（i为0-15）或GL_DEPTH_ATTACHMENT  或GL_STENCIL_ATTACHMENT 
textarget：需要挂载的纹理类型，这个方法对应的值为GLES20.GL_TEXTURE_2D
texture：需要挂载的纹理id
```

而纹理的生成和绑定逻辑和普通纹理一样,只不过glTexImage2D传入的buffer为null，生成一份纹理地址空间，挂载到FBO上，纹理的内容动态的生成。



```dart
final int[] textureObjectIds = new int[1];   
int texType =GLES20.GL_TEXTURE_2D; 
 GLES20.glGenTextures(1, textureObjectIds, 0);
        GLES20.glBindTexture(texType, textureObjectIds[0]);
        GLES20.glTexImage2D(texType, 0, GLES20.GL_RGBA, width, height,
                0, texFormat, GLES20.GL_UNSIGNED_BYTE, null);

        //设置缩小过滤为使用纹理中坐标最接近的一个像素的颜色作为需要绘制的像素颜色
        GLES20.glTexParameteri(texType, GLES20.GL_TEXTURE_MIN_FILTER, GLES20.GL_LINEAR);
        //设置放大过滤为使用纹理中坐标最接近的若干个颜色，通过加权平均算法得到需要绘制的像素颜色
        GLES20.glTexParameteri(texType, GLES20.GL_TEXTURE_MAG_FILTER, GLES20.GL_LINEAR);
        //设置环绕方向S，截取纹理坐标到[1/2n,1-1/2n]。将导致永远不会与border融合
        GLES20.glTexParameteri(texType, GLES20.GL_TEXTURE_WRAP_S, GLES20.GL_CLAMP_TO_EDGE);
        //设置环绕方向T，截取纹理坐标到[1/2n,1-1/2n]。将导致永远不会与border融合
        GLES20.glTexParameteri(texType, GLES20.GL_TEXTURE_WRAP_T, GLES20.GL_CLAMP_TO_EDGE);
```

获取正在绑定的纹理 GL_FRAMEBUFFER_BINDING



```csharp
    // C function void glGetIntegerv ( GLenum pname, GLint *params )

    public static native void glGetIntegerv(
        int pname,
        int[] params,
        int offset
    );

参数 pname 传GL_FRAMEBUFFER_BINDING
```

使用完之后调用glDeleteFramebuffers()删除帧缓存对象



```java
    // C function void glDeleteFramebuffers ( GLsizei n, const GLuint *framebuffers )

    public static native void glDeleteFramebuffers(
        int n,
        int[] framebuffers,
        int offset
    );
```

帧缓存对象完整性检查glCheckFramebufferStatus



```java
    // C function GLenum glCheckFramebufferStatus ( GLenum target )

    public static native int glCheckFramebufferStatus(
        int target
    );

参数说明：
target 可以为GLES20.GL_FRAMEBUFFER，在OpenGLES3.0后   GL_READ_FRAMEBUFFER 和 GL_DRAW_FRAMEBUFFER两种 只读或者只写的framebuffer类型
如果有错误发生，返回0
```

**渲染缓存对象**
 基本流程和挂载纹理对象一致，也是要先glGenRenderbuffers，在glBindRenderbuffer



```java
    // C function void glGenRenderbuffers ( GLsizei n, GLuint *renderbuffers )

    public static native void glGenRenderbuffers(
        int n,
        int[] renderbuffers,
        int offset
    );

    // C function void glBindRenderbuffer ( GLenum target, GLuint renderbuffer )

    public static native void glBindRenderbuffer(
        int target,
        int renderbuffer
    );
```

这里的target的和挂载纹理对象时传的GLES20.GL_TEXTURE_2D不同，而是GLES20.GL_RENDERBUFFER

调用glBindRenderbuffer之后还没有分配存储空间来存储图像信息，只是创建了一个所有状态都为默认值的渲染缓存，需要使用glRenderbufferStorage来分配对应存储空间



```java
   // C function void glRenderbufferStorage ( GLenum target, GLenum internalformat, GLsizei width, GLsizei height )

    public static native void glRenderbufferStorage(
        int target,
        int internalformat,
        int width,
        int height
    );

参数说明：
target 必须时GLES20.GL_RENDERBUFFER，
渲染缓存的类型可以是深度缓存、模版缓存，比如GLES20.GL_DEPTH_COMPONENT16，GL_STENCIL_INDEX8等
```

然后通过glFramebufferRenderbuffer对渲染缓存进行挂载，类似于纹理对象的挂载方式glFramebufferTexture2D



```java
    // C function void glFramebufferRenderbuffer ( GLenum target, GLenum attachment, GLenum renderbuffertarget, GLuint renderbuffer )

    public static native void glFramebufferRenderbuffer(
        int target,
        int attachment,
        int renderbuffertarget,
        int renderbuffer
    );

参数说明
target和renderbuffertarget为GLES20.GL_FRAMEBUFFER
attachment为渲染缓存的类型 例如深度缓存对象为GLES20.GL_DEPTH_ATTACHMENT
```

使用完之后对应渲染缓存对象的删除方法为glDeleteRenderbuffers



```java
   // C function void glDeleteRenderbuffers ( GLsizei n, const GLuint *renderbuffers )

    public static native void glDeleteRenderbuffers(
        int n,
        int[] renderbuffers,
        int offset
    );
```

## 38.3 实践

通过上面两小节的学习，我们知道了FBO的概念以及使用流程，下面我们通过对其实践应用加深理解。
 实现目标：画一个三角形（但不直接显示在屏幕上），然后进行高斯模糊，然后在渲染到屏幕上,
 该实践会涉及到高斯模糊，它的实现原理和具体实现方案我们在下一篇中结合GPUImage源码来一起解读学习。欢迎关注公众号“音视频开发之旅”，一起学习成长。

## 38.4 遇到的问题

**在实践中遇到各种渲染不出来的问题，归纳了常见的场景和分析解决方案**

1. location的解析 名称和glsl不对应
2. 渲染一直FBO上,即使解绑后也没有在屏幕上渲染
3. shader或者program加载出错
4. 数据设置问题
5. FBO代码顺序书写问题导致GL_FRAMEBUFFER_BINDING值不是主屏幕
6. 在ondrawframe的时候没有use当前的program, 如果只有一个filter不会有这种问题,但是链式filter就会暴露该问题.不设置使用的还是最初的program

-->分析过程

1. 针对glsl进行最小化,比如片元着色器直接指定颜色值
2. GLES20.glGetProgramiv(program, GLES20.GL_LINK_STATUS, status, 0);
3. 添加GLES20.glGetError()检查
4. 断点调试program shader location vbo 等值
5. 断点分析出问题的filter的ondrawframe

## 38.5 资料

《OpenGL ES 3.0编程指南》
 《OpenGL 编程指南》
 [OpenGL Frame Buffer Object (FBO)](https://links.jianshu.com/go?to=http%3A%2F%2Fwww.songho.ca%2Fopengl%2Fgl%5C_fbo.html)
 [GPGPU计算观念和基本思路总结](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fwwybj%2Farticle%2Fdetails%2F4218868)
 [OpenGL中的FBO对象（含源码）](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.cnblogs.com%2Faokman%2Farchive%2F2010%2F11%2F14%2F1876987.html)
 [OpenGL.FrameBuffer Object](https://links.jianshu.com/go?to=http%3A%2F%2Fwww.cppblog.com%2Finit%2Farchive%2F2012%2F02%2F16%2F165778.aspx)
 [OpenGL ES 帧缓冲对象（FBO）：Render to texture](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fcauchyweierstrass%2Farticle%2Fdetails%2F53166940)
 [OpenGL编程指南第十章：Framebuffer](https://links.jianshu.com/go?to=http%3A%2F%2Fblog.sina.com.cn%2Fs%2Fblog%5C_9fe6ba930102x607.html)
 [glBindFramebuffer() 离屏渲染+双缓存+读取opengl像素 glReadPixels()](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fu011450490%2Farticle%2Fdetails%2F51336302)

## 38.6 收获

1. 理解了FBO的基本知识和使用流程
2. 实践中遇到渲染不出的问题解决



 