# 17-JNI与NDK的学习和使用

从这篇开始，我们进入JNI和NDK系列的学习实践，来一起学习成长吧.

## 17.0 目录

1. 什么是JNI、NDK？
2. Java和Native交互流程
3. 通过AS创建Native CPP简单的项目
4. JNI基础知识介绍
5. 实现JAVA和Native的相互调用
6. 资料
7. 收获

## 17.1 什么是JNI、NDK？

JNI：Java Native Interface（java本地接口），使得Java与本地语言（
 C、CPP）相互调用

NDK：Native Development Kit，是Android的一个工具开发包，帮助开发者快速开发C、CPP动态库，自动将动态库打包进入APK。

通过JNI实现Java和Native的交互，在Android上通过NDK实现JNI的功能。

## 17.2 Java和Native交互流程

JNI

1. 在Java类中通过native关键字声明Native方法
2. javac命令编译Java类得到class文件
3. 通过javah命令（javah -jni class名称）导出JNI的头文件（.h文件）
4. 实现native方法
5. 编译生成动态库（.so文件）
6. 实现Java和C、CPP的相互调用

NDK

1. 配置NDK环境、创建Natvie CPP项目）
2. 在Java类中通过native关键字声明Native方法
3. 自动生成native方法，实现native方法
4. 通过ndk-build或者cmake编译产生动态库
5. 实现Java和C、CPP的相互调用

## 17.3 通过AS创建Native CPP简单的项目

**1. 如何配置NDK环境**

SDK Manager —》SDK Tools中下载选中NDK，LLDB和CMake。
 其中NDK是Native开发工具包，
 LLDB是调试Native代码用
 CMake是编译工具

![img](https:////upload-images.jianshu.io/upload_images/1791669-cb9a4a39711f33d8.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

配置好环境后，我们就可以开始创建Native CPP项目了

**2. 通过AS创建CPP项目**

AS New Project 选中 Native CPP项目。即可自动创建一个demo项目。

Java通过调用native方法stringFromJNI获取一个字符串。



```cpp
//Java 代码
public class MainActivity extends AppCompatActivity {

    ...
    // Used to load the 'native-lib' library on application startup.
    static {
        System.loadLibrary("native-lib");
    }

    /**
     * A native method that is implemented by the 'native-lib' native library,
     * which is packaged with this application.
     */
    public native String stringFromJNI();

    public native static String stringFromJNIStatic();
}

//JNI 代码
#include <jni.h>
#include <string>

extern "C" JNIEXPORT jstring JNICALL
Java_com_av_mediajourney_MainActivity_stringFromJNI(
        JNIEnv* env,
        jobject /* this */) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}

extern "C"
JNIEXPORT jstring JNICALL
Java_com_av_mediajourney_MainActivity_stringFromJNIStatic(JNIEnv *env, jclass clazz) {

}
```

代码比较简单，但是麻雀虽小，五脏俱全。 我们看到这个小小的JNI方法可能会有以下疑问。

1. stringFromJNI生成的关联的Native方法名称为什么是Java_com_av_mediajourney_MainActivity_stringFromJNI？可以是其他的吗？
2. JNI方法的参数 JNIEnv* 和 jobject代码什么意思？*
3. Java中需要的是String类型，为什么JNI返回的是一个jstring类型？
4. extern "C" 是什么意思？
5. JNIEXPORT和JNICALL又是什么意思？

这涉及到JNI的基本知识，我们通过对JNI基本知识的学习来解决上面的疑惑。

## 17.4 JNI基本知识

本小节分如下内容

1. JNIEnv和jobject jclass
2. Java 语言中的数据类型是如何映射到 c/cpp本地语言中的
3. java的属性和方法在JNI 签名

### 17.4.1 JNIEnv 和 jobject 、 jclass

JNIEnv：是指线程上下文环境，每个线程有且只有一个JNIEnv实例
 JNIEnv 结构包括 JNI 函数表



![img](https:////upload-images.jianshu.io/upload_images/1791669-4b464ee159faa717.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

图片来源于：《JNI编程指南》

第二个参数的意义取决于该方法是静态还是实例方法(static or an instance method)。当本地方法作为一个实例方法时，第二个参数相当于对象本身，即 this. 当本地方法作为一个静态方法时，指向所在类.

### 17.4.2  Java 语言中的数据类型是如何映射到 c/cpp本地语言中的

在 Java 中有两类数据类型：**基本数据类型**，如，boolean,int, float, char；另一种为**引用数据类型**，如，类，实例，数组。

Java和JNI的基本数据类型映射关系如下



![img](https:////upload-images.jianshu.io/upload_images/1791669-6fdcbc8c9a3a3506.png?imageMogr2/auto-orient/strip|imageView2/2/w/562/format/webp)

> 相比基本类型，对象类型的传递要复杂很多。 Java 层对象作为指针传递到 JNI 层，它指向 JavaVM 内部数据结构。使用这种指针的目的是：不希望 JNI 用户了解 JavaVM 内部数据结构。对引用类型指针所指结构的操作，都要通过 JNI 方法进行，比如，"java.lang.String"对象，JNI 层对应的类型为 jstring，对该 类型 的操作要通过 JNIEnv-> NewStringUTF 进行。

**针对字符串对象**
 通过字符串拼接来展示



```cpp
//Java代码新增字符串拼接的native方法

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        ...
        tv.setText(stringFromJNI()+appendString("hengheng","hahah"));
    }

    ...

    //新加字符串拼接方法
    public native String appendString(String str1,String str2);
}


//对应JNI的实现

extern "C"
JNIEXPORT jstring JNICALL
Java_com_av_mediajourney_MainActivity_appendString(JNIEnv *env, jobject thiz, jstring str1,
                                                   jstring str2) {
    //使用对应的 JNI 函数把 jstring 转成 C/C++字串
    //Unicode 以 16-bits 值编码；UTF-8 是一种以字节为单位变长格式的字符编码，并与 7-bits
    //ASCII 码兼容。UTF-8 字串与 C 字串一样，以 NULL('\0')做结束符
    //调用 GetStringUTFChars，把一个 Unicode 字串转成 UTF-8 格式字串
    const char *string1 = env->GetStringUTFChars(str1, NULL);

    //调用该函数会有内存分配操作，失败后，该函数返回 NULL，并抛 OutOfMemoryError 异常。
    if(string1 == NULL){
        return NULL;
    }
    const char *string2 = env->GetStringUTFChars(str2, NULL);
    if(string2 == NULL){
        return NULL;
    }

    //string：string是STL当中的一个容器，对其进行了封装，所以操作起来非常方便。
    //char*：char *是一个指针，可以指向一个字符串数组，至于这个数组可以在栈上分配，也可以在堆上分配，堆得话就要你手动释放了。
    std::string const cc = std::string(string1) + std::string(string2);

    //调用 ReleaseStringUTFChars 释放 GetStringUTFChars 中分配的内存(Unicode -> UTF-8转换的原因)。
    env->ReleaseStringUTFChars(str1,string1);
    env->ReleaseStringUTFChars(str2,string2);

    //使用 JNIEnv->NewStringUTF 构造 java.lang.String；
    return env->NewStringUTF(cc.c_str());

}
```

JNI String函数汇总表如下：



![img](https:////upload-images.jianshu.io/upload_images/1791669-7db8d4b926d70e5b.png?imageMogr2/auto-orient/strip|imageView2/2/w/1060/format/webp)

图片来源于：《JNI编程指南》

**针对数组**
 通过求和int类型数组来展示



```java
//java代码修改
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        ...

        int[] intarry = new int[10];
        for (int i = 0; i < 10; i++) {
            intarry[i] = i;
        }
        tv.setText(stringFromJNI()+appendString("hengheng","hahah")+sumArray(intarry));
    }

    ...

    public native int sumArray(int[] intarray);


}


//JNI实现

extern "C"
JNIEXPORT jint JNICALL
Java_com_av_mediajourney_MainActivity_sumArray(JNIEnv *env, jobject thiz, jintArray intarray) {

    jint sum;
    jsize length = env->GetArrayLength(intarray);

    //方案一 通过GetIntArrayRegion指定buf赋值范围
//    jint *buf ;
//
//    env->GetIntArrayRegion(intarray, 0, length, buf);
//    for (jint i = 0; i < length; ++i) {
//        sum += buf[i];
//    }

    //方案二：通过GetIntArrayElements和ReleaseIntArrayElements，
    //返回 Java 数组的一个拷贝(实现优良的VM，会返回指向 Java 数组的一个直接的指针，并标记该内存区域，不允许被 GC)。
    jint *pInt = env->GetIntArrayElements(intarray, NULL);
    for (jint i = 0; i < length; ++i) {
        sum += pInt[i];
    }
    env->ReleaseIntArrayElements(intarray,pInt,0);
    return sum;
}
```

JNI Array 函数汇总表如下：



![img](https:////upload-images.jianshu.io/upload_images/1791669-3e16d7bd2b6461bc.png?imageMogr2/auto-orient/strip|imageView2/2/w/1060/format/webp)

图片来源于：《JNI编程指南》

**针对对象（非String和数组的对象）和对象数组**

通过FindClass 获取到jclass，这块内容会涉及到引用的类型，比如GlobalRef 和 LocalRef，我们在下一篇会详细学习实践。而对Java对象的描述又涉及到 java的属性和方法在JNI 签名相关知识，我们来一起学习下。

### 17.4.3  java的属性和方法在JNI 签名

我们这一小节，学习Java熟悉和方法在JNI的签名，实践从本地代码访问Java对象成员、调用 Java 方法。

> 签名的作用：为了准确描述一件事物.
>  Java Vm 定义了类签名，方法签名；其中方法签名是为了支持方法重载。

Java 语言支持两种成员(field)：(static)静态成员和对象成员. 在 JNI 获取和赋值成员的方法是不同的. 同样的，方法也是两种方法: 静态方法和对象方法。

我们先看下了解下java的属性和方法在JNI 签名对应关系，然后通过通过native修改java成员值以及调用java方法为例对其进行了解熟悉。



<img src="https:////upload-images.jianshu.io/upload_images/1791669-71e9525679e85eae.png?imageMogr2/auto-orient/strip|imageView2/2/w/1084/format/webp" alt="img" style="zoom:50%;" />

其中要特别注意的是：

1. 类描述符开头的'L'与结尾的';'必须要有
2. 数组描述符，开头的'['必须有.
3. 方法描述符规则: "(各参数描述符)返回值描述符"，其中参数描述符间没有任何分隔 符号
    描述符很重要，请烂熟于心. 写 JNI，对于错误的签名一定要特别敏感，此时编译器帮不 上忙，执行 make 前仔细检查你的代码。

下面我们开始通过修改native修改java 成员变量和和调用方法修改修改java变量的值。



```php
访问对象成员分三步，
1. 通过 GetObjectClass 从 obj 对象得到 cls.
2. 通过 GetFieldID 得到对象成员 ID, 如下：
fid = (*env)->GetFieldID(env, cls, "s", "Ljava/lang/String;");
3. 通过在对象上调用下述方法获得成员的值：
jstr = (*env)->GetObjectField(env, obj, fid);
此外 JNI 还提供Get/SetIntField，Get/SetFloatField 访问不同类型成员。
```

先来看下通过JNI访问修改java属性的例子



```cpp
//Java 代码 定义两个成员变量： 静态变量和实例变量

public class MainActivity extends AppCompatActivity {

   
    private String value = "123";
    private static String value_static = "321";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        ...
        tv.setText(stringFromJNI()+appendString("hengheng","hahah")+sumArray(intarry)
        +"\n"+"value="+value+" value_static="+value_static);
    }
}

//JNI层
void accessField(JNIEnv *pEnv, jobject pJobject);

extern "C" JNIEXPORT jstring JNICALL
Java_com_av_mediajourney_MainActivity_stringFromJNI(
        JNIEnv* env,
        jobject obj) {
    ...

    accessField(env,obj);

    return tmp;

}

void accessField(JNIEnv *env, jobject obj) {
    
    //1. 获取class
    jclass jclazz = env->GetObjectClass(obj);
    
    //2. 通过GetFieldID获取fieldid
    // 签名一定要熟悉，否则在运行时直接会导致崩溃。比如String对应签名时"Ljava/lang/String;"
    jfieldID fieldId = env->GetFieldID(jclazz, "value", "Ljava/lang/String;");


    jstring jst = static_cast<jstring>(env->GetObjectField(obj, fieldId));

    jst = env->NewStringUTF("456");

    //3. 通过SetObjectField，修改fieldId的值
    env->SetObjectField(obj, fieldId, jst);

    
    
    ///下面来修改静态成员

    //1. 通过GetStaticFieldID获取静态成员fieledid
    jfieldID fielId2 = env->GetStaticFieldID(jclazz, "value_static", "Ljava/lang/String;");

    jstring jst2 = env->NewStringUTF("789");
    //2. 通过SetStaticObjectField给静态成员变量赋值
    env->SetStaticObjectField(jclazz,fielId2,jst2);
    
}
```

接着我们来看下 JNI调用Java的方法



```dart
JNI访问Java方法的步骤：
1.通过 GetMethodID 在给定类中查询方法. 查询基于方法名称和签名
2.本地方法调用 Call<Return Value Type>Method

方法签名由各参数类型签名和返回值签名构成. 参数签名在前，并用小括号括
起.
```



```csharp
public class MainActivity extends AppCompatActivity {

    ...

    //定义实例方法
    private void setValue(String value) {
        this.value = value;
    }

    //定义静态方法
    private static void setValue_static(String value_static) {
        MainActivity.value_static = value_static;
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        ...
        tv.setText(stringFromJNI()+appendString("hengheng","hahah")+sumArray(intarry)
        +"\n"+"value="+value+" value_static="+value_static);
    }
}


//JNI实现

void accessMethod(JNIEnv *env, jobject obj) {
    jclass clazz = env->GetObjectClass(obj);

    jmethodID methodId = env->GetMethodID(clazz, "setValue", "(Ljava/lang/String;)V");
    if (methodId == NULL) {
        return;
    }
    //这里一定要注意 不能直接env->CallVoidMethod(obj,methodId,"set by native method");
    //JNI中对象类型的使用一定用通过env方法来操作，比如生成string：env->NewStringUTF
    jstring jst = env->NewStringUTF("set by native method");
    env->CallVoidMethod(obj,methodId,jst);
    
    
    //调用静态方法
    jmethodID methodId1= env->GetStaticMethodID(clazz, "setValue_static", "(Ljava/lang/String;)V");
    if(methodId1== NULL){
        return;
    }
    jstring jst1 = env->NewStringUTF("\n set by native method staic");

    env->CallStaticVoidMethod(clazz,methodId1,jst1);

}
```

这里遇到了一个问题，折腾了好一阵。问题是：
 使用 env->CallVoidMethod(obj,methodId, "set by native method");运行时报错
 JNI DETECTED ERROR IN APPLICATION: use of deleted global reference 0x71749eae2e

检查签名没有错，通过java -p class 也确认了签名的正确性，但是为什么报错呐？

[How can I call Java Methods containing String parameter(s) using JNI?]: http://www.jguru.com/faq/view.jsp?EID=226786(https://links.jianshu.com/go?to=http%3A%2F%2Fwww.jguru.com%2Ffaq%2Fview.jsp%3FEID%3D226786)

看到了正确的用法，突然意识到在JNI中java的String不是基本数据类型，数据的生成要通过env对应的方法获取。
 改为



```php
jstring jst = env->NewStringUTF("set by native method");
env->CallVoidMethod(obj,methodId,jst);
```

另外还可以通过 JNI 调用 Java 类的构造方法和父类的方法

### 17.4.4 性能优化

> 执行一个 Java/native 调用要比 Java/Java 调用慢 2-3 倍. 也可能有一些 VM 实现，Java/native 调用性能与 Java/Java 相当。(此种虚拟机，Java/native 使用 Java/Java相同的调用约定)。
>  native/Java 调用效率可能与 Java/Java 有 10 倍的差距，因为 VM 一般不会做 Callback 的优化。

通过 FindClass 、GetFieldID、GetMethodID 去找到对应的信息是很耗时的，如果方法被频繁调用，那么肯定不能每次都去查找对应的信息，有必要将它们缓存起来，在下一次调用时，直接使用缓存内容就好了

> 可以在项目中加一套 Hash 表, 封装 FindClass，GetMethodID，GetFieldID等函数，查询的所有操作，都对 Hash 表操作，如首次 FindClass 一个类，这时可以把一个类的所有成员缓存到 Hash 表中，用名字+签名做键值。
>  引入了这个优化，项目的执行效率有 100 倍的提高；
>
> 1. 用一个 Hash 表，还是每个类一个 Hash 表
> 2. 首次 FindClass 类时，一次缓存所有的成员，还是用时缓存
>     最终做的选择是：为了降低冲突，每个类一个 Hash 表，并且一次缓存一个类的所有成员。

## 17.5 Java和Native的相互调用

上面两个小节中在对**Java 语言中的数据类型是如何映射到 c/cpp本地语言中的**
 以及  **java的属性和方法在JNI 签名**的学习实践已经充分的展示了相互调用。如果还有不清晰，请回看上一小节。

这里我们再来回顾下，这篇的目标 以及疑惑

**JNI方法的参数 JNIEnv 和 jobject代码什么意思**
 **Java中需要的是String类型，为什么JNI返回的是一个jstring类型？**
 —》这个两个问题，相信通过上面的学习实践，已经有很好的理解

我们再来其他几个问题
 **stringFromJNI生成的关联的Native方法名称为什么是Java_com_av_mediajourney_MainActivity_stringFromJNI？可以是其他的吗？**

JNI的方法名称是根据java的全包名+类名，并且把”.”替换为”_”,为规则生成的。这是静态注册的方式，当然也有动态注册的方式，这个我们下一篇再来详细学习实践。

**extern "C" 是什么意思？**
 extern “C”的作用是避免编译器按照CPP的方式编译C函数
 C语言不支持函数的重载，编译之后函数名称不变
 CPP支持函数的重载，编译之后函数名称会发生变化。调用的时候导致找不到JNI的实现

**JNIEXPORT和JNICALL又是什么意思？**



```cpp
在jni.h中可以看到这两个宏的定义
//JNIEXPORT表示 该函数是否可以导出
#define JNIEXPORT  __attribute__ ((visibility ("default")))
//调用规范
#define JNICALL
```

这篇就到这里了，下一篇我们来学习JNI的相关引用以及注册相关内容，欢迎关注公众号“音视频开发之旅”，一起学习

## 17.6 资料

《JNI编程指南》
 [NDK官网] : [https://developer.android.google.cn/ndk/guides?hl=zh-cn](https://links.jianshu.com/go?to=https%3A%2F%2Fdeveloper.android.google.cn%2Fndk%2Fguides%3Fhl%3Dzh-cn)
 [Android：JNI 与 NDK到底是什么？] : [https://blog.csdn.net/carson_ho/article/details/73250163](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fcarson_ho%2Farticle%2Fdetails%2F73250163)

## 17.7 收获

感谢你的阅读

通过对JNI和NDK的学习实践，

1. 了解了JNI和NDK是什么，以及两者之间的关系；
2. Android如何配置进行NDK的开发
3. JNI基本知识介绍（JNIEnv、数据类型对应关系、属性和方法签名等）
4. 实现Android中Java和Native的相互调用

 