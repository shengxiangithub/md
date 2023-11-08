# 15-OpenGL ES粒子系统 - 喷泉

## 15.0 目录

1. 粒子和粒子系统
2. 实践：喷泉效果
3. 遇到的问题
4. 资料
5. 收获

通过该篇的实践实现如下效果



![img](https:////upload-images.jianshu.io/upload_images/1791669-fd620a208dae1316.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)

## 15.1 什么是粒子和粒子系统

如何定义粒子？
 一个粒子有位置信息（x,y,z）、运动方向、颜色、生命值（开始和结束的时间）等属性

什么粒子系统？
 通过渲染绘制出大量 位置、形状、方向、颜色不同的物体（粒子），从而形成大量粒子运动的效果。

明确了概念我们来逐步实现烟花效果

## 15.2 实践：喷泉效果

面对一个比较大或者没有尝试过的项目或内容，害怕、怯懦时常出现，这时要理清目标，抓住主线，运用结构化思维，拆解流程，然后逐步实现每个环节，解决每个环节以中的问题，这也是打怪升级的过程，下面一起来享受这个过程吧。

目标：了解和运用粒子系统，实现比较炫酷的效果，从粒子喷泉着手。

**流程拆解**

1. 梳理粒子特性（坐标、颜色、运动矢量、开始时间）以及  重力和阻力的影响、持续时间
2. 粒子发生器如何发射粒子（反射点、方向、数量）
3. 编写着色器glsl代码
4. 编写Program
5. 编写GLSurfaceView的Render进行着色器加载、编译、链接、使用、渲染

**下面开始我们的具体实践**

### 15.2.1. 定义粒子系统对象

定义粒子的属性以及提供添加粒子的方法



```java
public class ParticleSystem {
    //位置 xyz
    private final int POSITION_COMPONENT_COUNT = 3;
    //颜色 rgb
    private final int COLOR_COMPONENT_COUNT = 3;
    //运动矢量 xyz
    private final int VECTOR_COMPONENT_COUNT = 3;
    //开始时间
    private final int PARTICLE_START_TIME_COMPONENT_COUNT = 1;


    private final int TOTAL_COMPONENT_COUNT = POSITION_COMPONENT_COUNT + COLOR_COMPONENT_COUNT
        + VECTOR_COMPONENT_COUNT + PARTICLE_START_TIME_COMPONENT_COUNT;

    //步长
    private final int STRIDE = TOTAL_COMPONENT_COUNT * VertexArray.BYTES_PER_FLOAT;


    //粒子游标
    private int nextParticle;
    //粒子计数
    private int curParticleCount;
    //粒子数组
    private final float[] particles;
    //最大粒子数量
    private final int maxParticleCount;
    //VBO
    private final VertexArray vertexArray;

    public ParticleSystem(int maxParticleCount) {
        this.particles = new float[maxParticleCount * TOTAL_COMPONENT_COUNT];
        this.maxParticleCount = maxParticleCount;
        this.vertexArray = new VertexArray(particles);
    }

    /**
     * 添加粒子到FloatBuffer
     *
     * @param position        位置
     * @param color           颜色
     * @param direction       运动矢量
     * @param particStartTime 开始时间
     */
    public void addParticle(Geometry.Point position, int color, Geometry.Vector direction, float particStartTime) {
        final int particleOffset = nextParticle * TOTAL_COMPONENT_COUNT;
        int currentOffset = particleOffset;
        nextParticle++;
        if (curParticleCount < maxParticleCount) {
            curParticleCount++;
        }
        //重复使用，避免内存过大
        if (nextParticle == maxParticleCount) {
            nextParticle = 0;
        }
        //填充 位置坐标 xyz
        particles[currentOffset++] = position.x;
        particles[currentOffset++] = position.y;
        particles[currentOffset++] = position.z;

        //填充 颜色 rgb
        particles[currentOffset++] = Color.red(color) / 255;
        particles[currentOffset++] = Color.green(color) / 255;
        particles[currentOffset++] = Color.blue(color) / 255;

        //填充 运动矢量
        particles[currentOffset++] = direction.x;
        particles[currentOffset++] = direction.y;
        particles[currentOffset++] = direction.z;

        //填充粒子开始时间
        particles[currentOffset++] = particStartTime;

        //把新增的粒子添加到顶点数组FloatBuffer中
        vertexArray.updateBuffer(particles, particleOffset, TOTAL_COMPONENT_COUNT);
    }

}
```

### 15.2.2. 定义粒子发射器对象



```java
public class ParticleShooter {

    //发射粒子的位置
    private final Geometry.Point position;
    //发射粒子的颜色
    private final int color;
    //发射粒子的方法
    private final Geometry.Vector direction;

    public ParticleShooter(Geometry.Point position, int color, Geometry.Vector direction) {
        this.position = position;
        this.color = color;
        this.direction = direction;
    }

    /**
     * 调用粒子系统对象添加粒子
     *
     * @param particleSystem
     * @param currentTime
     */
    public void addParticles(ParticleSystem particleSystem, float currentTime) {
        particleSystem.addParticle(position, color, direction, currentTime);
    }
}
```

### 15.2.3. 编写顶点和片元着色器



```cpp
uniform float u_Time;

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
    
    //把当前位置通过内置变量传给片元着色器
    gl_Position =  vec4(curPossition,1.0);
    gl_PointSize = 10.0;
}
```



```cpp
precision mediump float;

varying vec3 v_Color;
varying float v_ElapsedTime;

void main(){
    //粒子颜色随着颜色的推移变化
    gl_FragColor = vec4(v_Color/v_ElapsedTime, 1.0);
}
```

### 15.2.4. 编写Program封装着色器



```java
public class ParticleShaderProgram {

    private final String U_TIME ="u_Time";

    private final String A_POSITION="a_Position";
    private final String A_COLOR="a_Color";
    private final String A_DIRECTION="a_Direction";
    private final String A_PATRICLE_START_TIME="a_PatricleStartTime";

    private final int program;

    private final int uTimeLocation;
    private final int aPositionLocation;
    private final int aColorLocation;
    private final int aDirectionLocation;
    private final int aPatricleStartTimeLocation;



    public ParticleShaderProgram(Context context) {
        //生成program
        String vertexShaderCoder = ShaderHelper.loadAsset(context.getResources(), "particle_vertex_shader.glsl");
        String fragmentShaderCoder = ShaderHelper.loadAsset(context.getResources(), "particle_fragment_shader.glsl");
        this.program = ShaderHelper.loadProgram(vertexShaderCoder,fragmentShaderCoder);

        //获取uniform 和attribute的location

        uTimeLocation = GLES20.glGetUniformLocation(program,U_TIME);

        aPositionLocation = GLES20.glGetAttribLocation(program,A_POSITION);
        aColorLocation = GLES20.glGetAttribLocation(program,A_COLOR);
        aDirectionLocation = GLES20.glGetAttribLocation(program,A_DIRECTION);
        aPatricleStartTimeLocation = GLES20.glGetAttribLocation(program,A_PATRICLE_START_TIME);
    }

    /**
     * 设置 始终如一的Uniform变量
     * @param curTime
     */
    public void setUniforms(float curTime){
        GLES20.glUniform1f(uTimeLocation,curTime);
    }

    public int getProgram() {
        return program;
    }

    public int getaPositionLocation() {
        return aPositionLocation;
    }

    public int getaColorLocation() {
        return aColorLocation;
    }

    public int getaDirectionLocation() {
        return aDirectionLocation;
    }

    public int getaPatricleStartTimeLocation() {
        return aPatricleStartTimeLocation;
    }

    public void useProgram(){
        GLES20.glUseProgram(program);
    }
}
```

### 15.2.5. 编写Render加载、渲染



```java
public class ParticlesRender implements GLSurfaceView.Renderer {
    
    private final Context mContext;
    
    private ParticleShaderProgram mProgram;
    private ParticleSystem mParticleSystem;
    private long mSystemStartTimeNS;
    private ParticleShooter mParticleShooter;

    public ParticlesRender(Context context) {
        this.mContext = context;
    }

    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        GLES20.glClearColor(0f,0f,0f,0f);

        mProgram = new ParticleShaderProgram(mContext);

        //定义粒子系统 最大包含1w个粒子，超过最大之后复用最前面的
        mParticleSystem = new ParticleSystem(10000);

        //粒子系统开始时间
        mSystemStartTimeNS = System.nanoTime();
        
        //定义粒子发射器
        mParticleShooter = new ParticleShooter(new Geometry.Point(0f, -0.9f, 0f),
                Color.rgb(255, 50, 5),
                new Geometry.Vector(0f, 0.3f, 0f));
    }

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        GLES20.glViewport(0,0,width,height);
    }

    @Override
    public void onDrawFrame(GL10 gl) {
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT);
        
        //当前（相对）时间 单位秒
        float curTime = (System.nanoTime() - mSystemStartTimeNS)/1000000000f;
        //粒子发生器添加粒子
        mParticleShooter.addParticles(mParticleSystem,curTime);
        //使用Program
        mProgram.useProgram();
        //设置Uniform变量
        mProgram.setUniforms(curTime);
        //设置attribute变量
        mParticleSystem.bindData(mProgram);
        //开始绘制粒子    
        mParticleSystem.draw();
    }
}



ParticleSystem添加bindData和draw方法，如下

public class ParticleSystem {

...

public void bindData(ParticleShaderProgram program) {
        int dataOffset = 0;
        vertexArray.setVertexAttributePointer(dataOffset,
                program.getaPositionLocation(),
                POSITION_COMPONENT_COUNT, STRIDE);
        dataOffset +=POSITION_COMPONENT_COUNT;

        vertexArray.setVertexAttributePointer(dataOffset,
                program.getaColorLocation(),
                COLOR_COMPONENT_COUNT, STRIDE);
        dataOffset +=COLOR_COMPONENT_COUNT;

        vertexArray.setVertexAttributePointer(dataOffset,
                program.getaDirectionLocation(),
                VECTOR_COMPONENT_COUNT, STRIDE);
        dataOffset +=VECTOR_COMPONENT_COUNT;

        vertexArray.setVertexAttributePointer(dataOffset,
                program.getaPatricleStartTimeLocation(),
                PARTICLE_START_TIME_COMPONENT_COUNT, STRIDE);
    }

    public void draw() {
        GLES20.glDrawArrays(GLES20.GL_POINTS,0,curParticleCount);
    }

}

VertexArray添加方法 setVertexAttributePointer 用于给顶点着色器的Attribute属性的变量赋值

public class VertexArray {
   
   ...

    public void setVertexAttributePointer(int dataOffset, int location, int count, int stride) {
        floatBuffer.position(dataOffset);
        GLES20.glVertexAttribPointer(location,count,GLES20.GL_FLOAT,false,stride,floatBuffer);
        GLES20.glEnableVertexAttribArray(location);
        floatBuffer.position(0);
    }
}   
```

### 15.2.6. 在GLSurfaceView中使用Render



```java
public class ParticleActivity extends Activity{

    private GLSurfaceView glSurfaceView;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_particle_layout);
        glSurfaceView = findViewById(R.id.glSurfaceView);

        glSurfaceView.setEGLContextClientVersion(2);
        glSurfaceView.setRenderer(new ParticlesRender(this));
    }

    @Override
    protected void onResume() {
        super.onResume();
        glSurfaceView.onResume();
    }

    @Override
    protected void onPause() {
        super.onPause();
        glSurfaceView.onPause();
    }
}
```

**效果如下**

![img](https:////upload-images.jianshu.io/upload_images/1791669-d576fc5ebbdb526d.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)



是不是和预期的效果差别很大，别着急，只要在正确的路上，遇到问题就分析解决，最重要的是坚持前行。

## 15.3 问题

可以看到有如下几个问题

1. 粒子发射没有方向变化
2. 新发射的和下落的重叠

下面我们来一一解决。

### 问题1. 随机改变发射粒子的方向

粒子的发射方向是有发射器决定的，我们现在是一个固定的向上发送 Vector(0f, 0.3f, 0f)。

```cpp
mParticleShooter = new ParticleShooter(new Geometry.Point(0f, -0.9f, 0f),
                                       Color.rgb(255, 50, 5),
                                       new Geometry.Vector(0f, 0.3f, 0f));


public class ParticleShooter {
    ...
        public void addParticles(ParticleSystem particleSystem, float currentTime) {
        particleSystem.addParticle(position, color, direction, currentTime);
    }
    ...
}
```

如果想改变这个为随机方向，我们需要修改这个这个direction
 我们通过随机数以及矩阵变换来实现方向的随机



```java
public class ParticleShooter {
    ...
 	private float[] rotationMatrix = new float[16];
    private final Random random = new Random();
    final float angleVarianceInDegrees = 20f;

    public void addParticles(ParticleSystem particleSystem, float currentTime) {

        Matrix.setRotateEulerM(rotationMatrix, 0,
                               (random.nextFloat() -0.5f) * angleVarianceInDegrees,
                               (random.nextFloat()-0.5f) * angleVarianceInDegrees,
                               (random.nextFloat()-0.5f) * angleVarianceInDegrees);

        float[] tmpDirectionFloat = new float[4];

        Matrix.multiplyMV(tmpDirectionFloat,0,
                          rotationMatrix,0,
                          new float[]{direction.x,direction.y,direction.z,1f},0);

        Geometry.Vector newDirection = new Geometry.Vector(tmpDirectionFloat[0], tmpDirectionFloat[1], tmpDirectionFloat[2]);

        particleSystem.addParticle(position, color, newDirection, currentTime);
    }
    ...
}
```

效果如下：



![img](https:////upload-images.jianshu.io/upload_images/1791669-9e21737cc70f8c16.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)

### 问题2. 重叠覆盖

通过修改粒子的发射方向，看起来重叠效果也没有了，真的是这样吗，还是不容易观察到了？

为了验证问题我们同一时刻多加些粒子，即同时发送多批粒子粒子系统。
 修改也比较简单，加个for循环即可。



```cpp
public void addParticles(ParticleSystem particleSystem, float currentTime, int count) {

    for (int i = 0; i < count; i++) {
        Matrix.setRotateEulerM(rotationMatrix, 0,
                               (random.nextFloat() - 0.5f) * angleVarianceInDegrees,
                               (random.nextFloat() - 0.5f) * angleVarianceInDegrees,
                               (random.nextFloat() - 0.5f) * angleVarianceInDegrees);

        float[] tmpDirectionFloat = new float[4];

        Matrix.multiplyMV(tmpDirectionFloat, 0,
                          rotationMatrix, 0,
                          new float[]{direction.x, direction.y, direction.z, 1f}, 0);

        Geometry.Vector newDirection = new Geometry.Vector(tmpDirectionFloat[0], tmpDirectionFloat[1], tmpDirectionFloat[2]);

        particleSystem.addParticle(position, color, newDirection, currentTime);

    }
}
```

我们给这个count传20，效果如下



![img](https:////upload-images.jianshu.io/upload_images/1791669-41aefaeb7ef74a2c.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)

可以看到重叠覆盖又出现了，**为什么会出现这种情况？**

同一时刻，同一位置，既有下落的粒子，又有上升的粒子，如果下落的粒子后渲染，就会覆盖上升的粒子。

**那么该如何解决呐？**

OpenGL提供了累加混合技术 GL_BLEND_，公式如下



```undefined
输出 = （源因子 * 源片段）+（目标因子 * 目标片段）
```

具体实现为，在Render的onSurfaceCrate中设置



```java
   @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        GLES20.glClearColor(0f,0f,0f,0f);

        GLES20.glEnable(GLES20.GL_BLEND);
//采用累加混合 输出= （GLES20.GL_ONE * 源片段）+（GLES20.GL_ONE * 目标片段）
        GLES20.glBlendFunc(GLES20.GL_ONE, GLES20.GL_ONE);

....
}
```

效果如下：



![img](https:////upload-images.jianshu.io/upload_images/1791669-7d9d2576c52d4838.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)

可以看到没有再出现重叠覆盖的情况了。

还有一个问题是，每个粒子都是一个点，大小通过gl_PointSize设置为10，看到的是正方形粒子。能不能修改为圆形或者指定样式呐？_

### 问题3. 把点修改为纹理图片

下面我们就通过纹理图片来把每个点绘制为一个点精灵
 关于纹理的使用如果不熟悉，请先阅读[音视频开发之旅（12） OpenGL ES之纹理]
 **首先 修改片元着色器，添加2D纹理**



```cpp
precision mediump float;

uniform sampler2D u_TextureUnit;

varying vec3 v_Color;
varying float v_ElapsedTime;

void main(){
    //粒子颜色随着颜色的推移变化
    //gl_FragColor = vec4(v_Color/v_ElapsedTime, 1.0);

    //改为：通过内置函数texture2D和原来的fragcolor相乘
    gl_FragColor = vec4(v_Color/v_ElapsedTime, 1.0) * texture2D(u_TextureUnit, gl_PointCoord);

}
```

**然后修改Program，解析sample2D以及赋值**



```java
public class ParticleShaderProgram {

...
    private final String U_TEXTURE_UNIT ="u_TextureUnit";
    private final int uTextureUnit;

    public ParticleShaderProgram(Context context) {
     
        ...
        uTextureUnit = GLES20.glGetUniformLocation(program,U_TEXTURE_UNIT);

        ...
    }


   public void setUniforms(float curTime, int textureId){
        GLES20.glUniform1f(uTimeLocation,curTime);

        //激活纹理
        GLES20.glActiveTexture(GLES20.GL_TEXTURE0);
        //绑定纹理id
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D,textureId);
        //赋值
        GLES20.glUniform1i(uTextureUnit,0);

    }
....


}
```

**最后在Render中定义textureId以及在onDrawFrame中传值**



```java
public class ParticlesRender {
    
    ...
    private int mTextureId;
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
    ...
        mTextureId = TextureHelper.loadTexture(mContext, R.drawable.particle_texture);
    ...

   @Override
    public void onDrawFrame(GL10 gl) {
        ...
        //设置Uniform变量
        mProgram.setUniforms(curTime,mTextureId);
        ...
    }

}
    
    ...

}
```

效果如下：



![img](https:////upload-images.jianshu.io/upload_images/1791669-21266f872c302a8a.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)

最后我们修改顶点着色器中定义的点的大小以及运动矢量



```undefined
    gl_PointSize = 25.0;
```



```cpp
//定义粒子发射器
        mParticleShooter = new ParticleShooter(new Geometry.Point(0f, -0.9f, 0f),
                Color.rgb(255, 50, 5),
                new Geometry.Vector(0f, 0.8f, 0f));
```

就是开篇展示的效果



![img](https:////upload-images.jianshu.io/upload_images/1791669-4dfd8b85f9c668dc.gif?imageMogr2/auto-orient/strip|imageView2/2/w/320/format/webp)

这篇就到这里了，通过实践，一步步实现最终的喷泉效果。
 下一篇我们继续来学习实践粒子系统，实现烟花空中爆炸的效果。

## 15.4 资料

《OpenGL ES 3.0 编程指南》
 《OpenGL ES应用开发实践指南》

[粒子系统--烟花 [OpenGL-Transformfeedback]]
 [Android制作粒子爆炸特效]
 [OpenGL进阶(六)-粒子系统]
 [【OpenGL】Shader实例分析（七）- 雪花飘落效果]

## 15.5 收获

1. 了解粒子属性和粒子系统
2. 通过任务拆解逐步实现喷泉效果
3. 解决遇到的问题（重力、发射方向、重叠覆盖、点精灵）

