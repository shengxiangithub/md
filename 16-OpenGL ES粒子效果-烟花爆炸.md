# 16-OpenGL ES粒子效果-烟花爆炸

## 16.0 目录

1. 烟花爆竹场景和属性
2. 实践以及遇到的问题
3. 资料
4. 收获

通过该篇的实践实现如下效果



![img](https:////upload-images.jianshu.io/upload_images/1791669-16fdd9e8cb32901d.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)

## 16.1 烟花爆竹场景和属性

在上一篇 [音视频开发之旅（15） OpenGL ES粒子系统 - 喷泉](https://links.jianshu.com/go?to=https%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzU5NjkxMjE5Mg%3D%3D%26mid%3D2247483869%26idx%3D1%26sn%3Dae260fc2ee8524dc96f6667cc6d70045%26chksm%3Dfe5a30f2c92db9e4e68cf09f14e223325a5406a4cad372c96b855c892a253537f19a13c50258%26token%3D1731064216%26lang%3Dzh_CN%23rd) 的基础上 实现烟花爆炸效果。

在具体实践之前，先来想一想，烟花爆炸的场景和属性
 场景：从中心点开始爆炸，然后烟花粒子向各个方向炸开，整体形状也各有不同，比如 北京奥运会的大脚印，但是大部分还是圆形（因为设计实现相对简单），今天我们也实现一个圆形的烟花爆炸效果。
 属性：颜色、角度、运动矢量、形状。

实现流程和上一篇基本一致，不清晰整理的流程，建议先回看下

## 16.2 实践：烟花效果

在上篇的基础上通过修改Render的onDrawFrame中的粒子发射器来逐步实现烟花爆炸效果。

### **16.2.1  首先定义烟花爆炸对象**

 粒子的添加角度采用360度随机的方式添加



```java
public class ParticleFireworksExplosion {

    private float[] mDirectionVector = {0f, 0f, 0.5f, 1f};
    private float[] mResultVector = new float[4];
    private final Random mRandom = new Random();
    private float[] mRotationMatrix = new float[16];

    public void addExplosion(ParticleSystem particleSystem, Geometry.Point position, int color, float curTime) {

        Matrix.setRotateEulerM(mRotationMatrix, 0, mRandom.nextFloat() * 360, mRandom.nextFloat() * 360, mRandom.nextFloat() * 360);
        Matrix.multiplyMV(mResultVector, 0, mRotationMatrix, 0, mDirectionVector, 0);


        particleSystem.addParticle(
            position,
            color,
            new Geometry.Vector(mResultVector[0],
                                mResultVector[1],
                                mResultVector[2]),
            curTime);
    }
}
```

### **16.2.2 然后在Render中使用ParticleFireworksExplosion作为粒子发射器**



```java
public class ParticlesRender implements GLSurfaceView.Renderer {

    ...
        private ParticleFireworksExplosion particleFireworksExplosion;

    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {

        ...
            particleFireworksExplosion = new ParticleFireworksExplosion();
    }


    @Override
    public void onDrawFrame(GL10 gl) {
        ...

            //粒发射器添加粒子
            //        mParticleShooter.addParticles(mParticleSystem,curTime,20);
            int color = Color.rgb(255, 50, 5);

        //烟花爆炸粒子发生器 添加粒子
        particleFireworksExplosion.addExplosion(
            mParticleSystem,
            new Geometry.Point(
                0,
                0f ,
                0),
            color,
            curTime);
        ...

    }
}
```

效果如下：



![img](https:////upload-images.jianshu.io/upload_images/1791669-d72851cbaa648bbc.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)

### 问题1:   这明显不是烟花爆炸的效果

烟花爆炸是一个时间段只有一个爆炸，然后烟花颗粒向外圆形扩展开。而目前实现的效果是不断的发射新的粒子。

为此需要在onDrawFrame中间隔一段时间才发射粒子，这样让“烟花飞一会”从而形成烟花爆炸效果；同时，为了同一时间段粒子的数量保持一致，我们多同一个时间点我们同发射写粒子。具体实现如下：



```java
public class ParticleFireworksExplosion {

    ...

        private final int mPreAddParticleCount = 100;

    public void addExplosion(ParticleSystem particleSystem, Geometry.Point position, int color, float curTime) {

        //不是OnDrawFrame就添加烟花爆炸粒子，而是采用1/100的采样率 ，让粒子飞一会，从而产生烟花爆炸效果
        if (mRandom.nextFloat() < 1.0f / mPreAddParticleCount) {

            //同一时刻添加100个方向360随机的粒子
            for (int i = 0; i < mPreAddParticleCount; i++) {
                Matrix.setRotateEulerM(mRotationMatrix, 0, mRandom.nextFloat() * 360, mRandom.nextFloat() * 360, mRandom.nextFloat() * 360);
                Matrix.multiplyMV(mResultVector, 0, mRotationMatrix, 0, mDirectionVector, 0);


                particleSystem.addParticle(
                    position,
                    color,
                    new Geometry.Vector(mResultVector[0],
                                        mResultVector[1],
                                        mResultVector[2]),
                    curTime);
            }
        }
    }
}
```

效果如下：



![img](https:////upload-images.jianshu.io/upload_images/1791669-8b218e4fc96ffbdf.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)

### 问题2:  烟花扩散的效果不是圆形而是椭圆

这需要采用投影矩阵和视图矩阵变换 把椭圆形状投影到屏幕。

为此我们需要做如此处理

```undefined
1.  顶点着色器中添加unifrom mat4，在gl_Position的赋值需要和改矩阵相乘。
2.  Program中解析该location以及进行数据的输入
3.  Render中进行透视矩阵和试图矩阵的定义和计算，生成matrix数据
```

具体实现如下：

**1. 顶点着色器**

```cpp
uniform float u_Time;
uniform mat4 u_Matrix; //定义矩阵数据类型变量

attribute vec3 a_Position;
attribute vec3 a_Color;
attribute vec3 a_Direction;
attribute float a_PatricleStartTime;

varying vec3 v_Color;
varying float v_ElapsedTime;

void main(){
    v_Color = a_Color;
    //粒子已经持续时间  当前时间-开始时间
    v_ElapsedTime = u_Time - a_PatricleStartTime;
    //重力或者阻力因子，随着时间的推移越来越大
    float gravityFactor = v_ElapsedTime * v_ElapsedTime / 9.8;
    //当前的运动到的位置 粒子起始位置+（运动矢量*持续时间）
    vec3 curPossition = a_Position + (a_Direction * v_ElapsedTime);
    //减去重力或阻力的影响
    curPossition.y -= gravityFactor;

    //把当前位置通过内置变量传给片元着色器。
    //    gl_Position =  vec4(curPossition,1.0); //注释掉该实现，修改为下面的实现。

    //把当前位置和MVP矩阵相乘后，通过内置变量传给片元着色器
    gl_Position = u_Matrix * vec4(curPossition, 1.0);

    gl_PointSize = 25.0;
}
```

**2. Program中解析和赋值matrix**

```java
public class ParticleShaderProgram {

    ...
        private final String U_MATRIX = "u_Matrix";

    private final int uMatrixLocation;


    public ParticleShaderProgram(Context context) {
        ...

            //获取uniform 和attribute的location

            uMatrixLocation = glGetUniformLocation(program, U_MATRIX);

        ...
    }

    /**
     * 设置Uniform变量
     * @param matrix
     * @param curTime
     */
    public void setUniforms(float[] matrix, float curTime, int textureId){
        GLES20.glUniformMatrix4fv(uMatrixLocation, 1, false, matrix, 0);

        ...

    }
}
```

**3. Render中进行透视矩阵和试图矩阵的定义和计算，生成matrix数据**

```java
public class ParticlesRender implements GLSurfaceView.Renderer {

    ...
        private final float[] projectionMatrix = new float[16];
    private final float[] viewMatrix = new float[16];
    private final float[] viewProjectionMatrix = new float[16];

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        GLES20.glViewport(0,0,width,height);

        Matrix.perspectiveM(projectionMatrix, 0,45, (float) width
                            / (float) height, 1f, 10f);

        setIdentityM(viewMatrix, 0);
        translateM(viewMatrix, 0, 0f, -1.5f, -5f);
        multiplyMM(viewProjectionMatrix, 0, projectionMatrix, 0,
                   viewMatrix, 0);
    }


    @Override
    public void onDrawFrame(GL10 gl) {
        ...
            //设置Uniform变量 
            mProgram.setUniforms(viewProjectionMatrix,curTime,mTextureId);
    }
}
```

效果如下：



![img](https:////upload-images.jianshu.io/upload_images/1791669-999e224b68359855.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)

### 问题3、目前的烟花都是红色的，如何实现多彩的烟花？

只需要修改粒子的颜色即可，我们通过hsv颜色空间，随机调节色调，然后转为rgb颜色空间的颜色值，赋值给粒子，具体实现如下

```java
public class ParticleFireworksExplosion {

    private float[] mDirectionVector = {0f, 0f, 0.3f, 1f};
    private float[] mResultVector = new float[4];
    private final Random mRandom = new Random();
    private float[] mRotationMatrix = new float[16];

    private final int mPreAddParticleCount = 100;
    // 定义hsv色彩空间值，色调值默认为0（对应角度范围为0-360），饱和度和亮度默认为1（范围为0-2）
    private final float[] hsv = {0f, 1f, 1f};


    public void addExplosion(ParticleSystem particleSystem, Geometry.Point position, float curTime) {


        //不是OnDrawFrame就添加烟花爆炸粒子，而是采用1/100的采样率 ，让粒子飞一会，从而产生烟花爆炸效果
        if (mRandom.nextFloat() < 1.0f / mPreAddParticleCount) {

            //随机生成颜色的色调，
            hsv[0] = random.nextInt(360);
            //把hsv颜色空间转位rgb颜色空间值
            int color = Color.HSVToColor(hsv);

            //同一时刻添加100*3个方向360随机的粒子
            for (int i = 0; i < mPreAddParticleCount *3 ; i++) {
                Matrix.setRotateEulerM(mRotationMatrix, 0, mRandom.nextFloat() * 360, mRandom.nextFloat() * 360, mRandom.nextFloat() * 360);
                Matrix.multiplyMV(mResultVector, 0, mRotationMatrix, 0, mDirectionVector, 0);
                particleSystem.addParticle(
                    position,
                    color,
                    new Geometry.Vector(mResultVector[0],
                                        mResultVector[1]+0.3f,//由于烟花弹向上的惯性，爆炸时添加向上的偏移，效果看起来更加逼真。
                                        mResultVector[2]),
                    curTime);
            }
        }
    }
}
```

效果正如开篇展示



![img](https:////upload-images.jianshu.io/upload_images/1791669-f5cc9733ad060c73.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)

[完整代码已上传至github](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fayyb1988%2Fmediajourney)

## 16.3 参考

《OpenGL ES 3.0 编程指南》
 《OpenGL ES应用开发实践指南》

[粒子系统--烟花 [OpenGL-Transformfeedback]]
 [Android制作粒子爆炸特效]
 [Android超强大的粒子动画库，流星雨，爆炸，满屏飞花，等粒子特效快速实现]
 [OpenGL进阶(六)-粒子系统]
 [【OpenGL】Shader实例分析（七）- 雪花飘落效果]

## 16.4 收获

通过对OpenGL ES粒子系统的学习实践，发现通过粒子可以制作很多绚丽的效果。也在学习实践的过程中越来越发现路还很长，要不断持续学习实践才行。

具体到本篇收获
 分析烟花爆炸的场景与特性

通过实践逐步实现多彩的烟花效果
 遇到的问题解决

