# 10-GLSurfaceView源码解析&EGL环境

## 10.0 目录

1. GLSurfaceView源码解析
2. 通过TextureView+自定义EGL+GLThread来加载Shader并运行
3. 参考
4. 收获

## 10.1 GLSurfaceView源码解析

**查看源码的原则：以常用的API为入口，依据地图、带着问题、沿着主线来寻找答案**

比如GLSurfaceView我们在使用时调用如下两个方法

- setEGLContextClientVersion
- setRenderer



随后的操作都是在Render的回调中处理

void onSurfaceCreated(GL10 gl, EGLConfig config);
void onSurfaceChanged(GL10 gl, int width, int height);
void onDrawFrame(GL10 gl);

具体使用见上一篇 [ OpenGL ES 绘制平面图形 ]

我们今天的主角是EGL和GLThread,

### 10.1.1  setRenderer的实现

```csharp
public void setRenderer(Renderer renderer) {
    // 1. 检查GLThread的状态,如果不为空，则抛出异常
    //即一个GLSurfaceView只能有一个GLThread
    checkRenderThreadState();
    //2. 生成 EGLConfig 对象.用于指定OpenGL颜色、深度、模版等设置
    if (mEGLConfigChooser == null) {
        mEGLConfigChooser = new SimpleEGLConfigChooser(true);
    }

    //3. 生成 EGLContext 对象.用于提供EGLContext创建和销毁的处理
    if (mEGLContextFactory == null) {
        mEGLContextFactory = new DefaultContextFactory();
    }

    //4. 生成 EGLSurface窗口对象. 用于提供EGLSurface创建和销毁的处理
    if (mEGLWindowSurfaceFactory == null) {
        mEGLWindowSurfaceFactory = new DefaultWindowSurfaceFactory();
    }

    //5. 开启 GLThread线程 
    mRenderer = renderer;
    mGLThread = new GLThread(mThisWeakRef);
    mGLThread.start();
}
```

对象的含义

**EGLDisplay** 是对实际显示设备的抽象。
 **EGLSurface** 是对用来存储图像的内存区域FrameBuffer的抽象，包括Color Buffer, Stencil Buffer ,Depth Buffer.
 **EGLContext**  存储OpenGL ES绘图的一些状态信息。

### 10.1.2 接下来我们看下GLThread的实现

```java
static class GLThread extends Thread {
    GLThread(WeakReference<GLSurfaceView> glSurfaceViewWeakRef) {
        super();
        ...
    }

    @Override
    public void run() {
        setName("GLThread " + getId());
        if (LOG_THREADS) {
            Log.i("GLThread", "starting tid=" + getId());
        }

        try {
            guardedRun();
        } catch (InterruptedException e) {
            // fall thru and exit normally
        } finally {
            sGLThreadManager.threadExiting(this);
        }
    }
}
```

可以看到GLTread就是是一个Thread的子类，关键的处理逻辑在guardedRun()方法中，下面我们来看它的实现，这也是本篇的重点。

在guardedRun涉及了很多状态的同步问题，这个我们下一小节在来看，这里我们

### 10.1.3  GLThread的关键方法guardedRun （渲染的核心逻辑）【重点！】

![img](https:////upload-images.jianshu.io/upload_images/1791669-be968d7a17c23f7b.png?imageMogr2/auto-orient/strip|imageView2/2/w/990/format/webp)

图片来自[GLSurfaceView渲染过程详解]

private void guardedRun(){



```cpp
//1. 创建EGLContext上下文
mEglHelper.start();
```

//2. 创建EGLSurface，本质是申请一块内存
 mEglHelper.createSurface()
 //3. 获取GL对象，包装OpenGL API环境
 mEglHelper.createGL()

---》Render执行渲染必须在EGLContext和EGLSurface生成并绑定后才能进行。



```cpp
// 这里就是Render的回调方法
view.mRenderer.onSurfaceCreated(gl, mEglHelper.mEglConfig);
view.mRenderer.onSurfaceChanged(gl, w, h);
view.mRenderer.onDrawFrame(gl);
```

---》Render渲染后的数据要想在视图窗口显示，必须调用eglSwapBuffers交换Framebuffer



```swift
//opengl使用的双内存缓冲，一个进行显示，另一个则后台进行绘制，绘制OK后，交互内存进行显示
mEglHelper.swap();
```

}

### 10.1.4 EglHelper又是什么呐？

从1.3中EglHelper的几个方法调用入手





~~~cpp
public void start(){
    //1. 获取EGL对象
    mEgl = (EGL10) EGLContext.getEGL();

    //2. 获取EGLDisplay即显示对象
    mEglDisplay = mEgl.eglGetDisplay(EGL10.EGL_DEFAULT_DISPLAY);

    //3. 初始化
    mEgl.eglInitialize(mEglDisplay, version)

        //4. egl配置
        //这里的mEGLConfigChooser是在setRenderer中实现的
        mEglConfig = view.mEGLConfigChooser.chooseConfig(mEgl, mEglDisplay);

    //5. 创建EGL上下文. EGLContext是用来链接渲染和视图窗口的上下文。
    // 注意：createContext()很耗资源
    mEglContext = view.mEGLContextFactory.createContext(mEgl, mEglDisplay, mEglConfig);
}

public boolean createSurface() {
    //创建EGLsurface窗口
    //这里的mEGLWindowSurfaceFactory是在setRenderer中实现的
    mEglSurface = view.mEGLWindowSurfaceFactory.createWindowSurface(mEgl,
                                                                    mEglDisplay, mEglConfig, view.getHolder());



    ```cpp
        //将EGLSurface和EGLContext进行绑定
        mEgl.eglMakeCurrent(mEglDisplay, mEglSurface, mEglSurface,  mEglContext)
        ```

}

GL createGL() {
    //创建EGL上下文的gl对下
    //GLSurfaceView的使用方在回调函数中接受，比如：onDrawFrame(GL10 gl)

    GL gl = mEglContext.getGL();
}

public int swap() {
    //通过切换前后Frame，实现Surface的展示
    mEgl.eglSwapBuffers(mEglDisplay, mEglSurface)

}
~~~



## 10.2 TextureView +EGL+ GLThread绘制图形

将GLSurfaceView内容全部复制出来，然后取消其继承和接口实现，将所有报错的代码删除掉，这样就相当于剔除了GLSurfaceView的SurfaceView而保留了它的GL环境，我们可以使用GLEnvironment来进行渲染，并自由的指定渲染载体，可以是SurfaceView/TextureView.
 我原本的想法是自己把整个流程以及状态的同步等都要自己实现一遍。
 看到来自[GLSurfaceView的简单分析及巧妙借用] 的思路，觉得比较高效省事，如果一上来就自己去实现GL环境，可能会有不少收获，但是也会踩不少坑，先学会使用，在学造轮子。

其实这些都是借口了。。。 时间没有安排好是主要原因，周末加班赶需求，是这次实践不足的主要原因。先完成再完美，不断升级迭代。下一篇我们来学习实践OpenGL ES的矩阵变换和坐标系。

## 10.3 参考

《OpenGL ES应用开发实践指南》
 《音视频开发进阶指南》
 [从源码角度剖析Android系统EGL及GL线程]
 [GLSurfaceView渲染过程详解]
 [GLSurfaceView的简单分析及巧妙借用]

## 10.4 收获

通过本篇的学习实践，
 了解了GLSurfaceView内部是如何工作。
 了解了EGThread的实现和EGL上下文的意义。
 在TextureView的基础上创建EGL上文和GLThread来实现OpenGL的绘制。

