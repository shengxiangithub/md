# 01-三种方式绘制图片

 [腾讯课堂 零声教育](https://0voice.ke.qq.com) 整理，  版权归原作者所有，如有侵权请联系删除，原文链接：https://www.jianshu.com/p/41c6f5960fea

在android开发中我们最常使用的绘制图片的方式就是ImageView，设置src。那么有没有其他方案可以实现图片的绘制呐？

### 1.1 三种方案

1. 通过Imageview设置setImageBitmap



```java
final String path = Environment.getExternalStorageDirectory().getPath() + File.separator + "Pictures" + File.separator + "tmp.jpg";

        Bitmap bitmap = BitmapFactory.decodeFile(path);
        imageView.setImageBitmap(bitmap);
```

2. 通过自定义View，定义Paint和读取Bitmap，然后在onDraw中用Canvas进行drawBitamp

```java
public class CustomView extends View {
    private Paint paint;
    private Bitmap bitmap;

    public CustomView(Context context) {
        this(context,null,0);
    }

    public CustomView(Context context, @Nullable AttributeSet attrs) {
        this(context, attrs,0);
    }

    public CustomView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        paint = new Paint();
        paint.setAntiAlias(true);
        paint.setStyle(Paint.Style.STROKE);
    String path = Environment.getExternalStorageDirectory().getPath() + File.separator + "Pictures" + File.separator + "tmp.jpg";

        bitmap = BitmapFactory.decodeFile(path);
    }

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        if(bitmap!=null&&!bitmap.isRecycled()) {
            canvas.drawBitmap(bitmap, 0, 0, paint);
        }
    }
}
```

3. 通过SurfaceView，通过surfaceView的SufaceHolder，拿到到canvas然后进行绘制。

```java
surfaceView.getHolder().addCallback(new SurfaceHolder.Callback() {
            @Override
            public void surfaceCreated(SurfaceHolder holder) {
                if(holder == null){
                    return;
                }
                Paint paint = new Paint();
                paint.setAntiAlias(true);
//                                paint.setStyle(Paint.Style.STROKE);

       String path = Environment.getExternalStorageDirectory().getPath() + File.separator + "Pictures" + File.separator + "tmp.jpg";

                Bitmap bitmap1 = BitmapFactory.decodeFile(path);

                Canvas canvas = null;
                try {
                    canvas = holder.lockCanvas();
                    canvas.drawBitmap(bitmap1,0,0,paint);
                }catch (Exception e){
                    e.printStackTrace();
                } finally {
                    if(canvas!=null){
                        holder.unlockCanvasAndPost(canvas);
                    }

                }

                Log.d(TAG, "surfaceCreated: ");
            }

            @Override
            public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {
                Log.d(TAG, "surfaceChanged: format"+format+" w="+width+" h="+height);

            }

            @Override
            public void surfaceDestroyed(SurfaceHolder holder) {
                Log.d(TAG, "surfaceDestroyed: ");

            }
        });
```

### 1.2 SurfaceView和View的优劣对比以及使用场景

通过上面的例子我们可以看到SurfaceView和View的使用的方式的区别，View可以直接可以通过ImageView封装的API或者View的onDraw中拿到Canvas进行绘制。而SurfaceView则通过SurfaceHolder也是拿到Canvas进行绘制。
 下吗我们来说下SurfaceView和view的区别

一个窗口中的所有view，共享一个window，对应一个surface（绘图表面）surface中有个Canvas，可用于绘制，在WMS中只可见顶层的DecorView，在WMS中对应WindowState，app请求刷新Surface时，会在SurfaceFlinger内部建立对应的Layer。

优点：

1. 可以在子线程绘制--》适用于被动频繁刷新的界面，比如视频播放或者游戏
2. 刷新只会刷新自己而不会刷新其他view所有在的viewHierchy，避免造成整个viewHierchy刷新，性能会更好。
3. 双缓冲机制，避免闪烁，提升性能。
    缺点：
4. view的动画对sureface无效
    关闭window中所有view的硬件加速可以做到不透明的情况。

**如何使用：**
 Sureface.CallBack.
 Sureface.getHolder
 以及Sureface.lockCavas获取到canvas进行canvas操作然后sureface.unlockCavasAndPost进行页面的更新。

**问题：**
 为什么surfaceView不可见时会调用surfaceDestroyed，重新可见时有重新create？
 因为SurfaceView的双缓冲机制非常消耗内存，Android规定SurfaceView不可见时，会销毁Surfaceview的SurfaceHolder，以达到节省系统资源的目的。

### 1.3 android10+存储权限的获取

在写该demo的时候，由于采用从外部存储中读取一种图片进行绘制，在获取外部存储权限的时候会遇到了麻烦，google对用户权限的获取越来越收敛，这是好事。权限的授予用户自己控制，在隐私安全上做了更好的保障。那么对于开发者该如何获取用户的存储权限呐？要做一下三步

1. AndroidMainfest中注册
2. Application添加        android:requestLegacyExternalStorage="true"
3. 通过AppOpsManager动态的获取

### 1.4 小结

通过本章，我们了解了绘制图片根源都是在拿到Canvas然后在上面绘制bitmap。
 可以通过ImageView提供的API；可以通过自定义View，在onDraw方法中进行canvas的drawBitmap；也可以通过SurfaceView在onSurfaceCreate的时候通过surface的lockCanvas然后进行canvas的drawbitmap，最后在unlockCanvasAndPost到进行绘制。

SurfaceView和View最大的差异点在于，Surfaceview有自己单独的window，对应WMS中有自己独立的layer层，可以在子线程进行绘制，刷新时不会影响所在的View树结构中的其他View，适用于被动频繁刷新的场景。相应的劣势是View的动画对Surfaceview无效，比如移动缩放等，最明显的表现就是有动画进入或者滑动退出有SurfaceView页面时导致页面透明。但是事实上在关闭View的硬件加速后是可以的。
 关于Surfaceview的SurfaceHolder的生命周期，SurfaceView不可见时即进行销毁，可见时再重新创建，原因在于SurfaceView采用了双缓冲机制，做到了刷新是不闪烁。但是双缓冲是比消耗内存的，所以android做了上述的策略。

Android10以上读取sd卡的内容等权限被google收敛，首先要在AndroidManifest中注册，然后是使用前要动态的获取，在AndroidManifest的Application中也要加上




来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。