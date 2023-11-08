# 04-Camera视频采集

 [腾讯课堂 零声教育](https://0voice.ke.qq.com) 整理，  版权归原作者所有，如有侵权请联系删除，原文链接：[音视频开发之旅（四）Camera视频采集 - 简书 (jianshu.com)](https://www.jianshu.com/p/01e8ed63a4c8)

## 4.0 目录

1. Camera基础知识
2. 视频采集的流程
3. 遇到的问题和常见的坑（重点）
4. 收获

## 4.1  Camera基础知识

Camera 有几个重要的基础概念。

1. facing相机的方向，一般后置摄像头和前置摄像头。
2. Orientation：相机采集图片的角度，摄像头的传感器在手机中是横向的，在预览的时候根据Camera的预览方向进行顺时针旋转对应角度进行设置即可正常预览。如果不正确设置会导致预览时出现倒立、镜像等问题。把预览的图片保存为相册也要单独设置方向，注意这个方向和预览方向互不相干。
3. 预览图片的大小   预览容器的大小和摄像头支持的图片预览的图片大小，如果设置了Camera不支持的预览大小，会导致黑屏。
4. 可以设置帧回调然后，在每一帧中进行业务处理，比如，人脸识别等功能
5. Camera预览的图片格式有NV21 YUV420sp等
6. Camera需要一个容器把的Surface显示在屏幕上，一般SurfaceView，TextureView等。

## 4.2 视频采集的流程

1. 通过SurfaceView拿到SurfaceHolder，然后设置addCallback回调，当Surface创建、销毁、改变时触发对应的回调，在其中可以进行Camera的初始化以及参数设置
2. 通过new Camera（cameralId)生成一个对象。然后Camera.getParams获取到相关的参数，可以把重要的或者说比较关系的parmas打印出来，比如说支持多少个摄像头、支持的预览图片的大小、每个摄像头的方向等信息。可以根据需要设置对应的参数，比如图片的格式、图片的预览大小等。当然有一个必须要设置的就是Camera的预览展示方向，否则预览的到图片和正常的方向不一致。
3. 可以设置Camera的setPreviewCallback获取每一帧的回调，根据需要设置处理，开始预览startPreview以及帧回调的处理
4. 摄像头的切换

如果出发Camera的切换，需要把前一个Camera释放，重新生成和设置Camera切换预览
 和Activity生命周期想的关系，这个是有SurfaceView决定的，当页面可见时（onResume）创建或重新创建，页面不可见时（onPause）销毁释放

**具体实现如下**

1. SurfaceView的设置

```java
private SurfaceHolder mSurfaceHolder;

    private void initSurfaceView() {
        mSurfaceHolder = surfaceview.getHolder();
        mSurfaceHolder.addCallback(new SurfaceHolder.Callback() {
            @Override
            public void surfaceCreated(SurfaceHolder holder) {
                Log.d(TAG, "surfaceCreated: ");
                handleSurfaceCreated();
            }

            @Override
            public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {
            }

            @Override
            public void surfaceDestroyed(SurfaceHolder holder) {
                Log.d(TAG, "surfaceDestroyed: ");
                handleSurfaceDestroyed();
            }
        });
    }

    private void handleSurfaceDestroyed() {
        releaseCamera();
        mSurfaceHolder = null;
        Log.i(TAG, "handleSurfaceDestroyed: ");
    }

    private void handleSurfaceCreated() {
        Log.i(TAG, "handleSurfaceCreated: start");
        if (mSurfaceHolder == null) {
            mSurfaceHolder = surfaceview.getHolder();
        }
        if (mCamera == null) {
            initCamera(curCameraId);
        }
        try {
            //问题2：页面重新打开后SurfaceView的内容黑屏
            //Camera is being used after Camera.release() was called
            //在surfaceDestroyed时调用了Camera的release 但是没有设置为null,
            //--》如何解耦合，把生命周期相关的方法和Camera的生命周期绑定而不时在回调中处理，方便业务实现
            //onResume--》surfaceCreated
            //onPause--》surfaceDestroyed
            mCamera.setPreviewDisplay(mSurfaceHolder);
        } catch (IOException e) {
            e.printStackTrace();
            Log.e(TAG, "handleSurfaceCreated: " + e.getMessage());
        }
        startPreview();
        Log.i(TAG, "handleSurfaceCreated: end");
    }

    private void startPreview() {
//        mCamera.setPreviewCallback(new Camera.PreviewCallback() {
//            @Override
//            public void onPreviewFrame(byte[] data, Camera camera) {
//                Log.i(TAG, "onPreviewFrame: setPreviewCallback");
//            }
//        });
        //问题：很多时候，不仅仅要预览，在预览视频的时候，希望能做一些检测，比如人脸检测等。这就需要获得预览帧视频，该如何做呐？
        mCamera.setOneShotPreviewCallback(new Camera.PreviewCallback() {
            @Override
            public void onPreviewFrame(byte[] data, Camera camera) {
                Log.i(TAG, "onPreviewFrame: setOneShotPreviewCallback");
                Camera.Size previewSize = mCamera.getParameters().getPreviewSize();
                YuvImage yuvImage = new YuvImage(data, ImageFormat.NV21, previewSize.width, previewSize.height, null);
                ByteArrayOutputStream os = new ByteArrayOutputStream(data.length);
                if(!yuvImage.compressToJpeg(new Rect(0,0,previewSize.width,previewSize.height),100,os)){
                    Log.e(TAG, "onPreviewFrame: compressToJpeg error" );
                    return;
                }
                byte[] bytes = os.toByteArray();
                Bitmap bitmap = BitmapFactory.decodeByteArray(bytes, 0, bytes.length);

                //这里的处理方式是简单的把预览的一帧图保存下。如果需要做人脸设别或者其他操作，可以拿到这个bitmap进行分析处理
                //我们可以通过找出这张图片发现预览保存的图片的方向是不对的，还是Camera的原始方向，需要旋转一定角度才可以。
                if(curCameraId == Camera.CameraInfo.CAMERA_FACING_BACK){
                    bitmap = BitmapUtils.rotate(bitmap,90);
                }else {
                    bitmap = BitmapUtils.mirror(BitmapUtils.rotate(bitmap,270));
                }
                FileUtils.saveBitmapToFile(bitmap,"oneShot.jpg");
            }
        });

//        mCamera.setPreviewCallbackWithBuffer(new Camera.PreviewCallback() {
//            @Override
//            public void onPreviewFrame(byte[] data, Camera camera) {
//                Log.i(TAG, "onPreviewFrame: setPreviewCallbackWithBuffer");
//            }
//        });
        mCamera.startPreview();

    }
```

2. Camera的初始化和Params设置

```java
private void initCamera(int cameraId) {
    curCameraId = cameraId;
    mCamera = Camera.open(curCameraId);
    Log.d(TAG, "initCamera: Camera Open ");

    setCamerDisplayOrientation(this, curCameraId, mCamera);

    if (!hadPrinted) {
        printCameraInfo();
        hadPrinted = true;
    }
    Camera.Parameters parameters = mCamera.getParameters();
    Camera.Size closelyPreSize = CameraUtil.getCloselyPreSize(true, SystemUtils.getDisplayWidth(), SystemUtils.getDisplayHeight(), parameters.getSupportedPreviewSizes());
    Log.i(TAG, "initCamera: closelyPreSizeW="+closelyPreSize.width+" closelyPreSizeH="+closelyPreSize.height);
    parameters.setPreviewSize(closelyPreSize.width, closelyPreSize.height);
    mCamera.setParameters(parameters);
}

private void printCameraInfo() {
    //1. 调用getParameters获取Parameters
    Camera.Parameters parameters = mCamera.getParameters();

    //2. 获取Camera预览支持的图片格式(常见的是NV21和YUV420sp)
    int previewFormat = parameters.getPreviewFormat();
    Log.d(TAG, "initCamera: previewFormat=" + previewFormat); // NV21

    //3. 获取Camera预览支持的W和H的大小，
    // 手动设置Camera的W和H时，要检测camera是否支持，如果设置了Camera不支持的预览大小，会出现黑屏。
    // 那么这里有一个问，由于Camera不同厂商支持的预览大小不同，如果做到兼容呐？
    // 需要使用方采用一定策略进行选择（比如：选择和预设置的最接近的支持的WH）
    //通过输出信息，我们可以看到Camera是横向的即 W>H
    List<Camera.Size> supportedPreviewSizes = parameters.getSupportedPreviewSizes();
    for (Camera.Size item : supportedPreviewSizes
        ) {
        Log.d(TAG, "initCamera: supportedPreviewSizes w= " + item.width + " h=" + item.height);
    }

    //可以看到Camera的宽高是屏幕的宽高是不一致的，手机屏幕是竖屏的H>W，而Camera的宽高是横向的W>H
    Camera.Size previewSize = parameters.getPreviewSize();
    int[] physicalSS = SystemUtils.getPhysicalSS(this);
    Log.i(TAG, "initCamera: w=" + previewSize.width + " h=" + previewSize.height
          + " screenW=" + SystemUtils.getDisplayWidth() + " screenH=" + SystemUtils.getDisplayHeight()
          + " physicalW=" + physicalSS[0] + " physicalH=" + physicalSS[1]);

    //4. 获取Camera支持的帧率 一般是10～30
    List<Integer> supportedPreviewFrameRates = parameters.getSupportedPreviewFrameRates();
    for (Integer item : supportedPreviewFrameRates
        ) {
        Log.i(TAG, "initCamera: supportedPreviewFrameRates frameRate=" + item);
    }

    //5. 获取Camera的个数信息，以及每一个Camera的orientation，这个很关键，如果根据Camera的orientation正确的设置Camera的DisplayOrientation可能会导致预览倒止或者出现镜像的情况
    int numberOfCameras = Camera.getNumberOfCameras();
    Camera.CameraInfo cameraInfo = new Camera.CameraInfo();
    for (int i = 0; i < numberOfCameras; i++) {
        Camera.getCameraInfo(i, cameraInfo);
        Log.i(TAG, "initCamera: facing=" + cameraInfo.facing
              + " orientation=" + cameraInfo.orientation);
    }
}

/**
     *
     * @param activity
     * @param cameraId
     * @param camera
     */
public static void setCamerDisplayOrientation(Activity activity, int cameraId, Camera camera) {
    Camera.CameraInfo cameraInfo = new Camera.CameraInfo();
    Camera.getCameraInfo(cameraId, cameraInfo);
    int rotation = activity.getWindowManager().getDefaultDisplay().getRotation();
    Log.i(TAG, "setCamerDisplayOrientation: rotation=" + rotation + " cameraId=" + cameraId);
    int degress = 0;

    switch (rotation) {
        case Surface.ROTATION_0:
            degress = 0;
            break;
        case Surface.ROTATION_90:
            degress = 90;
            break;
        case Surface.ROTATION_180:
            degress = 180;
            break;
        case Surface.ROTATION_270:
            degress = 270;
            break;
    }
    int result = 0;
    if (cameraInfo.facing == Camera.CameraInfo.CAMERA_FACING_FRONT) {
        result = (cameraInfo.orientation + degress) % 360;
        result = (360 - result) % 360;

    } else if (cameraInfo.facing == Camera.CameraInfo.CAMERA_FACING_BACK) {
        result = (cameraInfo.orientation - degress + 360) % 360;
    }
    Log.i(TAG, "setCamerDisplayOrientation: result=" + result + " cameraId=" + cameraId + " facing=" + cameraInfo.facing + " cameraInfo.orientation=" + cameraInfo.orientation);

    camera.setDisplayOrientation(result);
}
```

3. Camera的预览、帧回调处理、保存图片旋转和镜像处理

```java
private void startPreview() {

    //问题六：很多时候，不仅仅要预览，在预览视频的时候，希望能做一些检测，比如人脸检测等。这就需要获得预览帧视频
    mCamera.setOneShotPreviewCallback(new Camera.PreviewCallback() {
        @Override
        public void onPreviewFrame(byte[] data, Camera camera) {
            Log.i(TAG, "onPreviewFrame: setOneShotPreviewCallback");
            Camera.Size previewSize = mCamera.getParameters().getPreviewSize();
            YuvImage yuvImage = new YuvImage(data, ImageFormat.NV21, previewSize.width, previewSize.height, null);
            ByteArrayOutputStream os = new ByteArrayOutputStream(data.length);
            if(!yuvImage.compressToJpeg(new Rect(0,0,previewSize.width,previewSize.height),100,os)){
                Log.e(TAG, "onPreviewFrame: compressToJpeg error" );
                return;
            }
            byte[] bytes = os.toByteArray();
            Bitmap bitmap = BitmapFactory.decodeByteArray(bytes, 0, bytes.length);

            //这里的处理方式是简单的把预览的一帧图保存下。如果需要做人脸设别或者其他操作，可以拿到这个bitmap进行分析处理
            //我们可以通过找出这张图片发现预览保存的图片的方向是不对的，还是Camera的原始方向，需要旋转一定角度才可以。
            if(curCameraId == Camera.CameraInfo.CAMERA_FACING_BACK){
                bitmap = rotate(bitmap,90);
            }else {
                bitmap = mirror(rotate(bitmap,270));
            }
            saveBitmapToFile(bitmap,"oneShot.jpg");
        }
    });


    mCamera.startPreview();

}

void saveBitmapToFile(Bitmap bitmap, String fileName) {
    File file = new File(MyApplication.getContext().getExternalFilesDir(Environment.DIRECTORY_PICTURES), fileName);
    if (file.exists()) {
        file.delete();
    }
    try {
        file.createNewFile();
        FileOutputStream fos = new FileOutputStream(file);
        bitmap.compress(Bitmap.CompressFormat.JPEG, 100, fos);
        fos.flush();
        fos.close();
    } catch (IOException e) {
        e.printStackTrace();
    }
}

//水平镜像翻转
public Bitmap mirror(Bitmap rawBitmap) {
    Matrix matrix = new Matrix();
    matrix.postScale(-1f, 1f);
    return Bitmap.createBitmap(rawBitmap, 0, 0, rawBitmap.getWidth(), rawBitmap.getHeight(), matrix, true);
}

//旋转
public Bitmap rotate(Bitmap rawBitmap, float degree) {
    Matrix matrix = new Matrix();
    matrix.postRotate(degree);
    return Bitmap.createBitmap(rawBitmap, 0, 0, rawBitmap.getWidth(), rawBitmap.getHeight(), matrix, true);
}
```

4. Camera的切换

```csharp
private void switchCamera() {
    if (mCamera != null) {
        releaseCamera();
        initCamera((curCameraId == Camera.CameraInfo.CAMERA_FACING_FRONT) ? Camera.CameraInfo.CAMERA_FACING_BACK : Camera.CameraInfo.CAMERA_FACING_FRONT);
        try {
            mCamera.setPreviewDisplay(mSurfaceHolder);
        } catch (IOException e) {
            e.printStackTrace();
        }
        startPreview();
    }
}

private void releaseCamera() {
    if (mCamera != null) {
        mCamera.setPreviewCallback(null);
        mCamera.stopPreview();
        mCamera.release();
        mCamera = null;
    }
}
```

## 4.3 遇到的问题  (页面卡住、黑屏、倒立等)

### 问题一 ：切换摄像头后画面卡住

解决：需要先关闭Camera释放资源，然后重新打开切换后的Camera，重新设置PreviewDisplay 然后开始预览

### 问题二： 页面重新打开后（在预览页按Home键推到后台，然后再回到前台）SurfaceView的内容黑屏

解决：通过查看log看到有一个异常信息：“Camera is being used after Camera.release() was called” 原来是在surfaceDestroyed时调用了Camera的release 但是没有设置为null,    在surfaceCreated的时候是根据camera是否为空来判断是否需要重新初始化。

### 问题三：前摄像头预览出现倒立并且是镜像状态



```cpp
public static void setCamerDisplayOrientation(Activity activity, int cameraId, Camera camera) {
    Camera.CameraInfo cameraInfo = new Camera.CameraInfo();
    Camera.getCameraInfo(cameraId, cameraInfo);
    int result = 0;
    if (cameraInfo.facing == Camera.CameraInfo.CAMERA_FACING_FRONT) {
        result = (cameraInfo.orientation) % 360;
    } 
    camera.setDisplayOrientation(result);
```

解决：

```cpp
`public static void setCamerDisplayOrientation(Activity activity, int cameraId, Camera camera) {
    Camera.CameraInfo cameraInfo = new Camera.CameraInfo();
    Camera.getCameraInfo(cameraId, cameraInfo);
    int result = 0;
    if (cameraInfo.facing == Camera.CameraInfo.CAMERA_FACING_FRONT) {
        result = (cameraInfo.orientation) % 360;
        result = (360 - result) % 360;
    } 
    camera.setDisplayOrientation(result);
```



![img](https:////upload-images.jianshu.io/upload_images/1791669-72135f3ffee72a26.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)


 图片来自[[Android: Camera相机开发详解(上)\]](https://www.jianshu.com/p/f8d0d1467584)

### 问题四：很多时候，不仅仅要预览，在预览视频的时候，希望能做一些检测，比如人脸检测等。这就需要获得预览帧视频，该如何做呐？

Camera提供了setPreviewCallback、setOneShotPreviewCallback以及setPreviewCallbackWithBuffer三个方法供使用者进行帧回调处理。比如下面的处理时，通过setOneShotPreviewCallback获取一帧的bitmap，然后进行保存到文件

```cpp
mCamera.setOneShotPreviewCallback(new Camera.PreviewCallback() {
    @Override
        public void onPreviewFrame(byte[] data, Camera camera) {
        Log.i(TAG, "onPreviewFrame: setOneShotPreviewCallback");
        Camera.Size previewSize = mCamera.getParameters().getPreviewSize();
        YuvImage yuvImage = new YuvImage(data, ImageFormat.NV21, previewSize.width, previewSize.height, null);
        ByteArrayOutputStream os = new ByteArrayOutputStream(data.length);
        if(!yuvImage.compressToJpeg(new Rect(0,0,previewSize.width,previewSize.height),100,os)){
            Log.e(TAG, "onPreviewFrame: compressToJpeg error" );
            return;
        }
        byte[] bytes = os.toByteArray();
        Bitmap bitmap = BitmapFactory.decodeByteArray(bytes, 0, bytes.length);           
        //这里的处理方式是简单的把预览的一帧图保存下。如果需要做人脸设别或者其他操作，可以拿到这个bitmap进行分析处理
        FileUtils.saveBitmapToFile(bitmap,"oneShot.jpg");
    }
});
```

### 问题五：发现保存的图片和预览的图片的方向不一致

解决： 预览通过Camera的setDisplay Orientation根据前后摄像头的需旋转的角度进行了处理，但是保存为图片是和预览时的设置时不相关的，需要单独处理



![img](https:////upload-images.jianshu.io/upload_images/1791669-1322f3cc792bd6f2.png?imageMogr2/auto-orient/strip|imageView2/2/w/1188/format/webp)


 图片来自[[Android: Camera相机开发详解(上)\]](https://www.jianshu.com/p/f8d0d1467584)



//我们可以通过找出这张图片发现预览保存的图片的方向是不对的，还是Camera的原始方向，需要旋转一定角度才可以。

```cpp
if(curCameraId == Camera.CameraInfo.CAMERA_FACING_BACK){
    bitmap = BitmapUtils.rotate(bitmap,90);
}else {
    bitmap = BitmapUtils.mirror(BitmapUtils.rotate(bitmap,270));
}


public class BitmapUtils {
    //水平镜像翻转
    public static Bitmap mirror(Bitmap rawBitmap) {
        Matrix matrix = new Matrix();
        matrix.postScale(-1f, 1f);
        return Bitmap.createBitmap(rawBitmap, 0, 0, rawBitmap.getWidth(), rawBitmap.getHeight(), matrix, true);
    }

    //旋转
    public static Bitmap rotate(Bitmap rawBitmap, float degree) {
        Matrix matrix = new Matrix();
        matrix.postRotate(degree);
        return Bitmap.createBitmap(rawBitmap, 0, 0, rawBitmap.getWidth(), rawBitmap.getHeight(), matrix, true);
    }
}
```

## 4.4 参考资料

《音视频开发进阶指南》
 [[Android: Camera相机开发详解(上)\]](https://www.jianshu.com/p/f8d0d1467584)    
 [Android camera2 实现相机预览及获取预览帧数据流](https://www.jianshu.com/p/e20a2ad6ad9a)
 [Activity启动后View何时开始绘制（onCreate中还是onResume之后？）](https://www.jianshu.com/p/c5d200dde486)

## 4.5 收获

1. Camera的Facing、Orientation 、Size、PreviewCallback等基础知识的实践和了解
2. camera预览的流程熟悉
3. 黑屏、卡顿、倒立、镜像等问题的分析和处理处理

 