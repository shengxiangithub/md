# 18-JNI - 引用类型、异常处理、函数注册

我们来继续学习JNI的一些知识，引用类型、异常处理以及函数注册。

## 18.0 目录

1. 引用类型的介绍与使用
2. JNI异常检测和处理的方式
3. 函数的静态注册和动态注册

## 18.1  引用

1. 局部引用
2. 全局引用
3. 全局弱引用

**LocalRef（局部引用）**

> 有两种方式让 LocalRef 无效，
>  一，native method 返回(指回到 Java 层，如果从一个本地函数返回到另一个本地函数，LocalRef 是有效的。)，JavaVM 自动释放 LocalRef；

> 二，用 DeleteLocalRef 主动释放。
>  既然 LocalRef 会被 JavaVM 自动释放，为什么还要有 DeleteLocalRef？
>  因为 LocalRef 是阻止引用被 GC，LocalRefTable 是有限的，当在本地代码中操作大量对象时，及时调用  DeleteLocalRef，会释放 LocalRef 在 LocalRefTable 中所占位置并使对象及时得到回收。否则，当超出限制，VM会报 LocalRef Overflow Exception，这个问题 是 JNI 编程中经常碰到的问题，请引起高度警惕

下面我们通过一个示例来看下 LocalRef的自动和手动释放的使用。



```cpp
// LocalRef  

public class MainActivity extends AppCompatActivity {
    ...
    protected void onCreate(Bundle savedInstanceState) {
        ...
        String s = this.NewJavaString();
        Log.i("MainActivity", "onCreate: s=" + s);
        //第二次调用时崩溃，原因是由于env->FindClass存在在localRef中，在方法调用有JNI返回到Java时释放。

        String s2 = this.NewJavaString();
        Log.i("MainActivity", "onCreate: s2=" + s2);
    }
    public native String newJavaString();
}


//对应JNI代码

jstring getJstring(JNIEnv *env) {
    static jclass clazz = NULL;
    if(clazz == NULL){
        clazz = env->FindClass("java/lang/String");
        if(clazz == NULL){
            return NULL;
        }
    }
    /**
     * 对应构造方法： String(char[] value)
     */
    jmethodID methodId = env->GetMethodID(clazz, "<init>", "([C)V");

    jcharArray charArray = env->NewCharArray(10);

    //通过NewObject创建对象
    jobject result = env->NewObject(clazz, methodId, charArray);

    // 手动释放LocalRef。
    //当在本地代码中操作大量对象时，而 LocalRefTable 又是有限的，不及时回收可能会引起LocalRef OverflowException 崩溃
    env->DeleteLocalRef(charArray);

    return static_cast<jstring>(result);
}

extern "C"
JNIEXPORT jstring JNICALL
Java_com_av_mediajourney_MainActivity_newJavaString(JNIEnv *env, jobject thiz) {
    return getJstring(env);

}


//-----》如果不在Java中二次调用localRef的方法，而是改为JNI的其他方法中调用，是不会崩溃的
extern "C" JNIEXPORT jstring JNICALL
Java_com_av_mediajourney_MainActivity_stringFromJNI(
        JNIEnv *env,
        jobject obj/* this */) {
    ...
    getJstring(env);

    getJstring(env);


    return tmp;

}
```

--》结论：如果从一个本地函数返回到另一个本地函数，LocalRef 是有效的。
 而如果从JNI返回到Java，JavaVM 自动释放 LocalRef；

针对LocalRef的Delete除了手动Delete或者等带系统删除处理外，还有一种更加高效的方式，即Push/PopLocalFrame 。

> Push/PopLocalFrame 常被用来管理 LocalRef. 在进入本地方法时，调用一次PushLocalFrame，并在本地方法结束时调用 PopLocalFrame. 此对方法执行效率非常高，不必调用 DeleteLocalRef，只要该上下文结尾处调用 PopLocalFrame 会一次性释放所有LocalRef。一定保证该上下文出口只有一个，或每个 return 语句都做严格检查是否调用了PopLocalFrame。

LocalRef: 每个被创建的 Java 对象，首先会被加入一个 LocalRef Table，这个 Table 大 小是有限的，当超出限制，VM会报 LocalRef Overflow Exception，然后崩溃. 这个问题 是 JNI 编程中经常碰到的问题，请引起高度警惕，在 JNI 中及时通过 DeleteLocalRef 释放 对象的 LocalRef. 又，JNI 中提供了一套函数：Push/PopLocalFrame，因为 LocalRef Table 大小是固定的，这套函数只是执行类似函数调用时，执行的压栈操作，在 LocalRef Table 中预留一部分供当前函数使用，当你在 JNI 中产生大量对象时，虚拟机仍然会因 LocalRef Overflow Exception 崩溃

** GlobalRef（全局引用）**
 LocalRef 一般自动创建(返回值为 jobject/jclass 等JNI 函数)，而 GlobalRef 必须通过 NewGlobalRef 由程序员主动创建



```cpp
jstring getJstring(JNIEnv *env) {
    static jclass clazz = NULL;
    if(clazz == NULL){
        jclass localRefClazz = env->FindClass("java/lang/String");
        if(localRefClazz == NULL){
            return NULL;
        }
        //通过localRefObject 生成一个全局的静态引用，方法返回到Java层后，也不会释放，保证可服用
         clazz = static_cast<jclass>(env->NewGlobalRef(localRefClazz));
        //LocalRef可以释放，避免LocalRefTable过大
        env->DeleteLocalRef(localRefClazz);
    }
   ...
}
```

** GlobalWeakRef（全局弱引用）**

> Weak Global Ref 用 NewGlobalWeakRef 于 DeleteGlobalWeakRef 进行创建和删除，多个本地方法调用过程中和多线程上下文中使用的特性与 GlobalRef 相同，但该类型的引用不保证不被 GC(如内存紧张)。
>  对于 Weak Global Ref 来说，需要使用下述代码判定：env->IsSameObject(wobj, NULL)

## 18.2 JNI 调用时的异常处理

在 JNI 中产生的异常(通过调用 ThrowNew)，与 Java 语言中异常发生的行为不同，JNI 中当前代码路径不会立即改变。在 Java 中发生异常，VM 自动把控制权转向 try/catch 中匹配的异常类型处理块。VM 首先清空异常队列，然后执行异常处理块。而JNI 中必须显式处理 VM 的处理方式。Native 提供了 ExceptionOccurred 和 ExceptionCheck 方法来检测是否有异常发生，前者返回的是 jthrowable 类型，后者返回的是 jboolean 类型。



```php
    jthrowable exc = env->ExceptionOccurred();
    jboolean result = env->ExceptionCheck();
    if (exc) {
       // 打印异常日志
       env->ExceptionDescribe();
       // 这行代码才是关键不让应用崩溃的代码，
       env->ExceptionClear();
       // 发生异常了要记得释放资源
       env->DeleteLocalRef(cls);
       env->DeleteLocalRef(obj);
    }
```

如果有异常，会通过 ExceptionDescribe 方法来打印异常信息，方便我们在 LogCat 中看到对应的信息。而 ExceptionClear 方法则是关键的不会让应用直接崩溃的方法，类似于 Java 的 catch 捕获异常处理，它会消除这次异常。

> 有两种方式检查是否有异常发生。
>
> 1. 大多数 JNI 函数用显式方式表明当前线程是否有异常发生。
> 2. 如果返回值不能表明是否有异常发生，需要用 JNI 提供的 ExceptionOccurred 检查当前线程是否有未处理异常。

## 18.3 函数注册

JNI的函数有有两种注册方式，我们从上一篇到目前为止用的都是静态注册。即：javac javah或者AS快捷方式，根据java的native方法名称生成对应的JNI方法名，生成的规则如下：    Java + 包名 + 类名 + 方法名

这种方式的好处是一键生成，比较方便。
 但是如果涉及到一些修改比如新增参数或者该改变签名可能后要重新生成对应的JNI方法。另外JNI中方法名过长也不太符合我们的阅读习惯，而动态注册可以很好的解决这个问题。

**什么是动态注册？**
 在so加载时，JNI_OnLoad函数中，_通过 env->RegisterNatives 方法手动对 JNI函数名称和 so 中的函数名进行绑定，虚拟机可以通过这个函数映射表直接找到相应的方法了。

**动态注册的流程**



```undefined
1. 定义JNINativeMethod 进行绑定
 2. RegisterNatives进行注册
3. 实现JNI_OnLoad_函数
```

每次java层加载System.loadLibrary之后，自动会查找改库一个叫JNI_OnLoad的函数。

JNINativeMethod结构体如下：



```cpp
typedef struct {
    const char* name;           //java层的native方法名称
    const char* signature;      //java方法签名
    void*       fnPtr;          //JNI中的函数指针
} JNINativeMethod;
```

RegisterNatives函数如下：



```cpp
/**jclass：java类
     JNINativeMethod* ：映射表*
    jint 个数
**/
  jint  (*RegisterNatives)(JNIEnv*, jclass, const JNINativeMethod*,
                        jint);
```

**实践**



```cpp
int JNI_OnLoad(JavaVM* vm, void* reserved){

    JNIEnv *env = NULL;
    if(vm->GetEnv((void **) &env, JNI_VERSION_1_6) != JNI_OK){
        return JNI_ERR;
    };

    jclass clazz = env->FindClass("com/av/mediajourney/MainActivity");
    if(clazz == NULL){
        return JNI_ERR;
    }
    //1. 获取注册表
    JNINativeMethod mMethods[] = {
            {"newJavaString","()Ljava/lang/String;",(void *)getJstring},
    };
    //2. 调用RegisterNatives进行注册
    if (mMethods != NULL) {
        env->RegisterNatives(clazz, mMethods, sizeof(mMethods) / sizeof(mMethods[0]));
    }

    return JNI_VERSION_1_6;
}
```

**System.loadLibrary加载流程**

对于动态注册：

```rust
System.loadLibrary->
   Runtime.loadLibrary->(Java)
     nativeLoad->(C: java_lang_Runtime.cpp)
       Dalvik_java_lang_Runtime_nativeLoad->
          dvmLoadNativeCode-> (dalvik/vm/Native.cpp)
              1) dlopen(pathName, RTLD_LAZY) (把.so mmap到进程空间，并把func等相关信息填充到soinfo中)
              2) dlsym(handle, "JNI_OnLoad")
              3) JNI_OnLoad->
                      RegisterNatives->
                         dvmRegisterJNIMethod(ClassObject* clazz, const char* methodName,
                                                const char* signature, void* fnPtr)->
                            dvmUseJNIBridge(method, fnPtr)->  (method->nativeFunc = func)

JNI函数在进程空间中的起始地址被保存在ClassObject->directMethods中。
struct ClassObject : Object {  
    /* static, private, and <init> methods */  
    int             directMethodCount;  
    Method*         directMethods;  
  
    /* virtual methods defined in this class; invoked through vtable */  
    int             virtualMethodCount;  
    Method*         virtualMethods;  
}  
此ClassObject通过gDvm.jniGlobalRefTable或gDvm.jniWeakGlobalRefLock获取。
```

对于静态注册：



```rust
在执行System.loadLibrary时，
无法把此JNI Lib实现的函数在进程中的地址增加到ClassObject->directMethods。
则直到需要调用的时候才会解析这些javah风格的函数 。
通过函数dvmResolveNativeMethod(dalvik/vm/Native.cpp)来进行解析

其执行流程如下所示：
void dvmResolveNativeMethod(const u4* args, JValue* pResult,
          const Method* method, Thread* self)  --> (Resolve a native method and invoke it.)
      1) void* func = lookupSharedLibMethod(method)(根据signature在所有已经打开的.so中寻找此函数实现)
              dvmHashForeach(gDvm.nativeLibs, findMethodInLib,(void*) method)->
                   findMethodInLib(void* vlib, void* vmethod)->
                      dlsym(pLib->handle, mangleCM)

     2) dvmUseJNIBridge((Method*) method, func);
     3) (*method->nativeFunc)(args, pResult, method, self);  
```

## 18.4 资料

《JNI编程指南》

[JNI_OnLoad简介]: https://blog.csdn.net/zerokkqq/article/details/79143834(https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fzerokkqq%2Farticle%2Fdetails%2F79143834)

## 18.5 收获

1. 了解了三种引用类型的使用场景和以及释放方式
2. 了解JNI异常处理和java的trycatch的区别以及异常检测和处理
3. 了解函数动态注册的流程



 