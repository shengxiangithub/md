# 14-OpenGL ES 实时滤镜

## 14.0 目录

1. OES是什么？SurfaceTeture是什么？
2. 实时滤镜的流程
3. 具体实践
4. 遇到的问题
5. 收获

## 14.1 基本知识介绍

**外部纹理是什么？ **



```java
    public static native void glBindTexture(
        int target,
        int texture
    );
```

我们在前面几篇中target一直是**GL_TEXTURE_2D**，即一张纹理图片，直接进行渲染。而在播放视频或者Camera预览 对数据进行滤镜、特效处理后在渲染到屏幕上而不是直接渲染到屏幕，这是就需要用到OES外部纹理，**GLES11Ext.GL_TEXTURE_EXTERNAL_OES**

**SurfaceTeture是什么？**

我们在前面章节介绍Camera预览时使用的是SurfaceHolder进行的preview。

SurfaceTexture 类是在 Android 3.0 中引入的，它对图像流的处理并不直接显示，而是转为GL外部纹理，可用于图像流数据的二次处理（如Camera滤镜，特效等）

SurfaceTexture从图像流（Camera预览、视频解码）中获得帧数据，调用updateTexImage()，根据内容流中最近的图像更新SurfaceTexture对应的GL纹理对象，接下来，就可以像操作普通GL纹理一样操作它了。

## 14.2 流程

Camera采集数据后不直接显示到屏幕上，而是先对这个图像做滤镜处理，即先渲染在一个外部纹理上，处理完之后在显示在屏幕上。

OpenGL纹理绘制的基本流程。

1. 创建Camera
2. 创建和加载glsl，创建编译链接program, 获取location
3. 创建外部纹理、绑定纹理、设置参数
4. 根据创建的纹理id生成一个SurfaceTexture
5. 设置Camera通过SurfaceTeture方式而不是SurfaceHolder进行预览
6. Camera将预览数据输出至此SurfaceTexture上，将一帧预览数据推送给外部纹理上
7. 给此纹理添加滤镜
8. 处理后的数据通过OpenGL ES绘制出来

## 14.3 实践：Camera预览添加实时滤镜（原图、黑白、冷暖色）

**1. 创建Camera**



```cpp
private void initCamera(int cameraId) {
        curCameraId = cameraId;
        mCamera = Camera.open(curCameraId);
        Log.d(TAG, "initCamera: Camera Open ");


        Camera.Parameters parameters = mCamera.getParameters();
        Camera.Size closelyPreSize = CameraUtil.getCloselyPreSize(true, SystemUtils.getDisplayWidth(), SystemUtils.getDisplayHeight(), parameters.getSupportedPreviewSizes());
        Log.i(TAG, "initCamera: closelyPreSizeW=" + closelyPreSize.width + " closelyPreSizeH=" + closelyPreSize.height);
        parameters.setPreviewSize(closelyPreSize.width, closelyPreSize.height);
        //这里把Camera的宽高做下反转
        mPreviewWidth = closelyPreSize.height;
        mPreviewHeight = closelyPreSize.width;

        mCamera.setParameters(parameters);
    }
```

**2. 加载glsl，创建program,获取location**



```undefined
省略
```

**3. 创建外部纹理**



```cpp
    private int genOesTextureId() {
        int[] textureObjectId = new int[1];
        GLES20.glGenTextures(1, textureObjectId, 0);
        //绑定纹理
        GLES20.glBindTexture(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, textureObjectId[0]);
        //设置放大缩小。设置边缘测量
        GLES20.glTexParameterf(GLES11Ext.GL_TEXTURE_EXTERNAL_OES,
                GL10.GL_TEXTURE_MIN_FILTER, GL10.GL_NEAREST_MIPMAP_LINEAR);
        GLES20.glTexParameterf(GLES11Ext.GL_TEXTURE_EXTERNAL_OES,
                GL10.GL_TEXTURE_MAG_FILTER, GL10.GL_LINEAR);
        GLES20.glTexParameteri(GLES11Ext.GL_TEXTURE_EXTERNAL_OES,
                GL10.GL_TEXTURE_WRAP_S, GL10.GL_CLAMP_TO_EDGE);
        GLES20.glTexParameteri(GLES11Ext.GL_TEXTURE_EXTERNAL_OES,
                GL10.GL_TEXTURE_WRAP_T, GL10.GL_CLAMP_TO_EDGE);
        return textureObjectId[0];
    }

int[] textureObjectId = new int[1];
        GLES20.glGenTextures(1, textureObjectId, 0);
         绑定外部纹理
        GLES20.glBindTexture(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, textureObjectId[0]);
         设置放大缩小。设置边缘测量
     GLES20.glTexParameterf(GLES11Ext.GL_TEXTURE_EXTERNAL_OES,
                GL10.GL_TEXTURE_MIN_FILTER, GL10.GL_NEAREST_MIPMAP_LINEAR);
        GLES20.glTexParameterf(GLES11Ext.GL_TEXTURE_EXTERNAL_OES,
                GL10.GL_TEXTURE_MAG_FILTER, GL10.GL_LINEAR);
        GLES20.glTexParameteri(GLES11Ext.GL_TEXTURE_EXTERNAL_OES,
                GL10.GL_TEXTURE_WRAP_S, GL10.GL_CLAMP_TO_EDGE);
        GLES20.glTexParameteri(GLES11Ext.GL_TEXTURE_EXTERNAL_OES,
                GL10.GL_TEXTURE_WRAP_T, GL10.GL_CLAMP_TO_EDGE);
```

**4. 根据创建的纹理id生成一个SurfaceTexture**



```cpp
mSurfaceTexture = new SurfaceTexture(mTextureId);
```

**5. 设置Camera通过SurfaceTeture方式而不是SurfaceHolder进行预览**



```css
mCamera.setPreviewTexture(mSurfaceTexture);
```

**6. Camera将预览数据输出至此SurfaceTexture上，将一帧预览数据推送给外部纹理上**



```csharp
public void onDrawFrame(GL10 gl) {
                //每次绘制后，都通知texture刷新
                if (mSurfaceTexture != null) {              

                     mSurfaceTexture.updateTexImage();
                }
                
            }
```

**7. OpenGL ES就可以操作此纹理，加滤镜**



```cpp
 private void onBindFilter() {
        GLES20.glUniform1i(mUFilterIndex, mIndex);

        switch (mIndex){
            case 0:
                //原图效果，不同处理
                break;
            case 1:
                //黑白滤镜

                GLES20.glUniform3fv(uColor,1,ImageBgData.GRAY_FILTER_COLOR_DATA,0);
                break;
            case 2:
                //暖色滤镜
                GLES20.glUniform3fv(uColor,1,ImageBgData.WARM_FILTER_COLOR_DATA,0);

                break;
            case 3:
                //冷色滤镜
                GLES20.glUniform3fv(uColor,1,ImageBgData.COOL_FILTER_COLOR_DATA,0);

                break;
        }
    }
```

**8. 处理后的数据通过OpenGL ES绘制出来**



```cpp
 GLES20.glActiveTexture(GLES20.GL_TEXTURE0 + getTextureType());
        GLES20.glBindTexture(GLES11Ext.GL_TEXTURE_EXTERNAL_OES, getTextureId());
        GLES20.glUniform1i(mUTexture, getTextureType());

//设置定点数据
        GLES20.glEnableVertexAttribArray(mAPosition);
        GLES20.glVertexAttribPointer(
                mAPosition,
                2,
                GLES20.GL_FLOAT,
                false,
                0,
                mVerBuffer);
        //
        GLES20.glEnableVertexAttribArray(mACoord);
        GLES20.glVertexAttribPointer(
                mACoord,
                2,
                GLES20.GL_FLOAT,
                false,
                0,
                mTextureCoordinate);
        //绘制三角形带
        GLES20.glDrawArrays(GLES20.GL_TRIANGLE_STRIP, 0, 4);

        GLES20.glDisableVertexAttribArray(mAPosition);
        GLES20.glDisableVertexAttribArray(mACoord);
```

[代码托管在Github上](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fayyb1988%2Fmediajourney)

## 14.4 遇到的问题

1. 如何设置给Camera设置SurfaceTexure而不是SurfaceHolde
2. 画面预览被放大了，进行正交投影 矩阵变换前 把视图的宽高比反转下，因为camera的width大于height
3. 在片元着色器中定义了uniform vec3 u_Color;但是写program中绑定数据时却用了glVertexAttrib3fv（这个只用于顶点着色器，如果把color定义在顶点着色器，然后通过varying传递也可以）
    修正为  GLES20.glUniform3fv(uColor,1,ImageBgData.COOL_FILTER_COLOR_DATA,0);即可正常切换预览

## 14.5 资料

[Android视频编辑器（四）通过OpenGL给视频增加不同滤镜效果]
 [专栏：Android图像处理之实时滤镜]
 [专栏：图像处理]
 [Android Camera使用OpenGL ES 2.0和GLSurfaceView对预览进行实时二次处理（黑白滤镜）]

## 14.6 收获

1. 了解OES概念
2. 了解FBO、VBO、PBO概念
3. 通过实践实时滤镜熟悉流程

