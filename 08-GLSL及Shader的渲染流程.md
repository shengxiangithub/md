# 08-GLSL及Shader的渲染流程

## 8.0 目录

1. GLSl是什么？
2. GLSL特有语法
3. Shader的渲染流程
4. EGL上下文环境
5. 参考
6. 收获

## 8.1 GLSL是什么？

GLSL（OpenGL Shading Language），是用来在OpenGL中编写着色器（顶点着色器和片元着色器）程序的语言，该程序会在GPU（Graphic Processor Unit）上执行，使得渲染管线具有可编程行。

## 8.2 GLSL的特有语法

GLSL和C非常相似，基本类型、函数、结构体（stuct、对应java的class）比较常用，流程控制等。没有指针，但增加了不少着色器特有的基本类型。
 **学习一门语言首先看它的数据类型的表示，再学习具体的运行流程**，由于运行流程上和C、Java等语言基本一致。所以我们来重点看看GLSL特有的数据类型的表示，从基本类型、修饰符、内置变量、内置函数四个方面说明。

### 8.2.1 基本类型

在计算机图形中，向量和矩阵是变换的基础，这两种数据类型也是GLSL的核心

vec2 ，vec3 ，vec4：浮点向量
 ivec2 ，ivec3 ，ivec4：整数向量
 uvec2 ，uvec3 ，uvec4：无符号整型向量
 bvec2 ，bvec3 ，bvec4：boolean向量
 mat2,mat3,mat4,mat2x3…： 浮点矩阵

sampler2D ： 二维纹理句柄

### 8.2.2  修饰符

const：    （只读） 常量变量
 attribute： 只能用于顶点着色器，用于经常更改的信息
 uniform:   (始终如一的)用于不经常更改的信息，可用于顶点和片元着色器
 varying：  （易变的）用于修饰从顶点着色器向片元着色器传递变量。

### 8.2.3 内置变量

最常见的是顶点着色器和片元着色器的输出变量

Vertex Shader的内置变量 ： gl_position和gl_pointSize
 FragmentShader的内置变量： gl_FragColor

### 8.2. 4 函数与内置函数

GLSL的函数和C的函数使用基本一致，在定义函数前，必须要先声明。
 不同之处在于GLSL在函数参数的传递上提供了特殊的限定符(修饰符)，和aidl有点类似，具体如下

in : 默认模式， 值传递的方式，不能修改
 inout: 引用传递，允许修改，修改后函数退出后发生变化
 out： 表示改变量的值不被传入函数，但在函数返回时将被修改

GLSL语言最强大的功能之一是提供了内置函数

常用的如下：

abs： 绝对值
 floor： 向下取整
 ceil：向上取整
 mod：取模
 min：最小
 max：最大
 clamp：中间值
 dot： 计算两个向量的点积
 pow：计算标量的幂次

在每个shader中必须有且只能有一个main函数

## 8.3 Shader的渲染流程

了解了GLSL的基本语法，下面我们来学习下如何把Shader传递给OpenGL的渲染管线。

### 8.3.1  OpenGL的渲染架构

![img](https:////upload-images.jianshu.io/upload_images/1791669-197599d4d46bf993.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

图片来源：[OpenGL入门第二课--常用的固定存储着色器]

> 从上图中我们可以看出整个管线分为2部分，Client和Server
>  Client就是我们编写的程序代码以及OpenGL API，这部分运行在CPU上
>  Server 是真正完成渲染操作的，它运行中GPU上
>  Client和Server的通信只能通过Attributes属性、Uniforms和Texture Data纹理数据这三种数据类型。并且Attributes属性只能传给顶点着色器

### 8.3.2 如何创建Shader可执行程序？

![img](https:////upload-images.jianshu.io/upload_images/1791669-8f5d5ad196aaa27c.jpeg?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

图片来自《音视频开发进阶指南》

下面看下如何创建创建在GPU可执行程序（Program）

我们通过上图可以看到分为四个环节

1. 创建program  ：glCreateProgram
2. attachShader: 创建顶点和片元着色器、设置source、编译，attach到program
3. 链接： glLinkProgram
4. 使用: glUseProgram

## 8.4 EGL环境

OpenGL 不负责窗口和上下文的管理，该职责有各平台自己实现，比如android平台的EGL就是担当这个角色, 它是 OpenGL ES 和 native window system 之间的接口.
 EGL采用双缓冲工作模式，Front Frame Buffer和Back Frame Buffer，正常绘制操纵的目标都在Back Frame Buffer ，操作完毕之后调用eglSwapBuffer将绘制完毕的FrameBuffer交换到Front Frame Buffer并显示。
 在android中GlSurfaceView实现了EGL环境和GLThread线程，下一篇我们通过对GlSurfaceView使用来绘制平面图形。

## 8.5 参考

《音视频开发进阶指南》
 《OpenGL ES 3.0编程指南》
 [GLSL基础语法介绍]

## 8.6 收获

1. 了解GLSL的概念和语法
2. 了解Shader渲染架构和程序的执行流程
3. 了解EGL

