# 07-OpenGL ES 基本概念

## 7.0 目录

1. OpenGL ES的简介
2. OpenGL ES的基本流程和概念

篇外话：本来这篇要写SurfaceView和TextureView相关的，但是没有理解清楚，主要是对于纹理和SurfaceFlinger等认知不足，而纹理又是OpenGL的一个重要概念，所以先开启OpenGL的系列，后面再补上SurfaceView和TextureView。

我第一次接触OpenGL ES是一年前，但是看到OpenGL中各种专业名词和专业术语，感觉云里雾里，虽然按照书中的介绍实现了效果，但是终究还是没有理解。这个系列我们一起对OpenGL ES进行重新学习实践，掌握OpenGL ES 3.0，编写迷人的OpenGL ES 3.0的程序。

下面开始今天的主题。

## 7.1 OpenGL ES的简介

OpenGL（Open Graphics Library）是一个跨平台图像程序接口，用于二维和三位图片的渲染。OpenGL ES（Open Graphic Library Embedded Systems）是OpenGL的一个子集，用于手机和嵌入式设备
 在android中对OpenGL ES的支持如下

1. OpenGL ES 2.0 支持android2.2以后的版本
2. OpenGL ES 3.0 支持android4.3以后的版本
3. OpenGL ES 3.1 支持android5.0以后的版本
    基于目前android系统占比分布和OpenGL ES的版本使用趋势，我们基于OpenGL ES3.0进行学习实践。

**有个问题：OpenGL ES在整个系统中的作用是什么？**

通过下面两种图我们可以看到OpenGL ES是CPU和GPU交互的桥梁，是一个Lib



![img](https:////upload-images.jianshu.io/upload_images/1791669-17d4b8d9b97620a7.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)


 图片来源 [OpenGL(一) OpenGL入门](https://www.jianshu.com/p/92208a75283d)



![img](https:////upload-images.jianshu.io/upload_images/1791669-8648b444440541c2.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



图片来源 [Android官方文档]

**那为什么需要OpenGL ES 这个桥梁呐？能不能CPU和GPU直接交互呐？**

将数据从一块内存复制到另一块内存中，传递速度是非常慢的，CPU主要处理逻辑控制，GPU在逻辑运算方面比较强大，可以高并行处理图形数据和复杂算法，我们对比看下CPU和GPU的内部结构

![img](https:////upload-images.jianshu.io/upload_images/1791669-3302c07a7693de44.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)


 图片来源 [OpenGL(一) OpenGL入门](https://www.jianshu.com/p/92208a75283d)



## 7.2 基本流程和概念

OpenGL ES 3.0 实现了具有可编程着色功能的图形管线，有两个规范组成：OpenGL ES 3.0 API 和 GLSL着色器语言。在学习这些规范之前，我们先了解下相关的基本概念

### 7.2.1 渲染的基本流程



![img](https:////upload-images.jianshu.io/upload_images/1791669-e1dbfb19b859c4b2.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)


 图片来源 [LearnOpenGL CN](https://links.jianshu.com/go?to=https%3A%2F%2Flearnopengl-cn.github.io%2F)



> OpenGL在处理Shader时，和其他编译器一样。通过编译、链接等步骤，生成了着⾊器程序(glProgram)，着⾊器程序同时包含了顶点着⾊器和⽚段着⾊器的运算逻辑。在OpenGL进行绘制的时候，⾸先由顶点着⾊器对传⼊的顶点数据进行运算。再通过图元装配，将顶点转换为图元。然后进行光栅化，将图元这种⽮量图形，转换为栅格化数据。最后，将栅格化数据传入⽚段着⾊器中进行运算。⽚段着色器会对栅格化数据中的每⼀个像素进行运算，并决定像素的颜⾊。

### 7.2.2 管线

它是一系列数据处理的过程，将应用程序的数据最终转换到渲染的图像。下图展示了OpenGL ES 图形管线



![img](https:////upload-images.jianshu.io/upload_images/1791669-57ebe11292d6b4c5.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)



图片来源：《OpenGL ES 3.0 编程指南》

### 7.2.3 顶点

OpenGL ES 的基本元素有点、线、三角形，就像盖钢架结构的房子一样，先确定好位置，搭出框架，OpenGL ES 也需要确定好位置即顶点

![img](https:////upload-images.jianshu.io/upload_images/1791669-00193c37b7df7cc1.png?imageMogr2/auto-orient/strip|imageView2/2/w/1018/format/webp)


 图片来源 [通俗易懂的 OpenGL ES 3.0（一）入门必备知识](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fa8688555%2Farticle%2Fdetails%2F82819646)



### 7.2.4 纹理

纹理通常是一张图片，也可以用来存储大量的数据，这些数据可以发送到着色器。我们这个章节了解下纹理作为一张图片的意义。
 **那么为什么需要纹理呐？**
 我们可以为每个顶点添加颜色来丰富图像的细节，如果想让图形开起来更加有趣真实，我们就必须有足够的顶点从而指定足够多的颜色，但这会产生很多额外开销。使用纹理可以避免这个问题

> 你可以想象纹理是一张绘有砖块的纸，无缝折叠贴合到你的3D的房子上，这样你的房子看起来就像有砖墙外表了。因为我们可以在一张图片上插入非常多的细节，这样就可以让物体非常精细而不用指定额外的顶点。
>
> ![img](https:////upload-images.jianshu.io/upload_images/1791669-86a5ecd18bad8494.png?imageMogr2/auto-orient/strip|imageView2/2/w/824/format/webp)
>
> 图片来源
>
> LearnOpenGL CN

下面是一张怀旧滤镜的纹理图



![img](https:////upload-images.jianshu.io/upload_images/1791669-a070d28cf05c1401.png?imageMogr2/auto-orient/strip|imageView2/2/w/512/format/webp)

### 7.2.5 顶点着色器（VertexShader）

顶点着色器一般用来处理图形每个顶点的变换，例如 平移、缩放、旋转、投影等
 顶点着色器是OpenGL ES中用于计算顶点属性的程序，每个顶点都会并行的执行一次顶点着色器程序。

### 7.2.6 图元装配

图元（Primitive）是点、线、三角形等基本几何对象，图元的每个顶点被发送到顶点着色器处理，而图元装配就是把这些顶点做合成图元

### 7.2.7 光栅化

光栅化是把顶点数据转换为片元的过程。将有基本元素（点、线、三角形）组成的几何图元变为二维图像的过程。
 这个过程包含两部分内容

1. 确定视图坐标中哪些区域被基本图元占用
2. 分配一个颜色值和一个深度值到各个区域

### 7.2.8 片段着色器（FragmentShader）

片段着色器又叫片元着色器，用来处理图形中每个像素点颜色计算和填充

### 7.2.9 逐片段操作

在逐片段操作阶段，每个片段上会执行 像素归属测试、裁剪测试、模版和深度测试、混合、抖动

> 在测试阶段之后，如果像素依然没有被剔除，那么像素的颜色将会和帧缓冲区中颜色附着上的颜⾊进⾏混合，混合的算法可以通过OpenGL的函数进行指定。但是OpenGL提供的混合算法是有限的，如果需要更加复杂的混合算法，一般可以通过像素着色器进行实现，当然性能会⽐比原⽣的混合算法差一些.

## 7.3 参考

《音视频开发进阶指南》
 《OpenGL ES 3.0 编程指南》
 [LearnOpenGL CN](https://links.jianshu.com/go?to=https%3A%2F%2Flearnopengl-cn.github.io%2F)
 [OpenGL(一) OpenGL入门](https://www.jianshu.com/p/92208a75283d)
 [通俗易懂的 OpenGL ES 3.0（一）入门必备知识](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fa8688555%2Farticle%2Fdetails%2F82819646)

## 7.4 收获

1. 了解OpenGL ES的版本和android系统兼容性
2. 了解OpenGL ES在手机CPU和GPU之间的桥梁作用
3. 了解OpenGL ES中顶点、着色器、图元装配、光栅化等基本概念
4. 了解OpenGL ES的图形管线渲染的流程

 