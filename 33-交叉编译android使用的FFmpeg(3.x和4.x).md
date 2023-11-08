# 33-交叉编译android使用的FFmpeg(3.x和4.x)

## 33.0 目录

1. 配置安装android交叉编译工具链
2. 手写FFmpeg编译脚本 进行编译（针对ffmpeg3.x和ffmpeg4.x版本）
3. androidStudio中引用使用ffmpeg
4. 遇到的问题
5. 资料
6. 收获

![img](https:////upload-images.jianshu.io/upload_images/1791669-38b205ab2fd23317.png?imageMogr2/auto-orient/strip|imageView2/2/w/552/format/webp)

这篇我们来学习实践ffmpeg的交叉编译，其中会涉及到**ffmpeg的版本**、**NDK的版本**、**编译脚本的编写**、**Gradler ABI处理** 以及 **CMakeLists.txt的针对不同ndk版本脚步的编写**
 在交叉编译的时候由于平台差异性大，需要工具来解决这一问题，就出现了各种工具链，即Toolchains。而NDK提供了交叉编译的一整套工具的集合，不同版本和配置会有不同差异，很容易就会出现各种问题，是一个不断折腾的过程，），本文记录学习实践的过程，**ffmpeg3.3.9+ndk16**，以及**ffmpeg4.3.1或者ffmpeg4.2.4+ndk21** 欢迎交流讨论。

## 33.1 配置安装android交叉编译工具链

1. [下载NDK](https://links.jianshu.com/go?to=https%3A%2F%2Fdeveloper.android.google.cn%2Fndk%2Fdownloads%2F)
    NDK有很多版本 14，16，21等等，我们该选择什么版本比较好呐？
    现在一般都选用16以上的，16时编译工具链做了不少变化。而不同的ffmpeg版本支持的NDK版本也不同，比如，ffmpeg3.x的版本使用android-ndk-r16b，但是ffmpeg4.x的版本使用16就编译不过，这里选用的是android-ndk-r21e.
2. 配置NDK环境
    android-ndk-r21e



```bash
配置环境变量
vim ~/.bash_profile

export NDK_PATH=/Users/xxx/tools/android-ndk-r21e
export PATH=${PATH}:$NDK_PATH

编辑保存好之后执行下下面语句使之生效
. ~/.bash_profile
```

1. 安装android交叉编译工具链（也可省略）
    /Users/xxx/tools/android-ndk-r 21e/build/tools路径下有make-standalone-toolchain.sh和make_standalone_toolchain.py，我们使用py版本来配置android交叉编译工具链

用到make_standalone_toolchain.py



```bash
先创建交叉编译工具链输出的路径
/Users/xxx/tools/android-ndk-r21e/android-toolchains/android-19

mkdir android-toolchains
cd android-toolchains
mkdir android-19
cd android-19

with open(src, 'rb') as fsrc:

到android-ndk-r16b/build/tools
下执行
sudo ./make_standalone_toolchain.py --arch arm --api 19 --install-dir ../../android-toolchains/android-19/arm-arch
```

这个步骤相当于把配置的arch和api等相关信息在指定文件夹下面做个copy（个人理解），方便后续编写ffmpeg编译脚本时用指定toolchain。

之所以说这个步骤可以省略，是因为，也可以直接在后续编写ffmpeg编译脚本中指定不同ABI和API对应的toolchain路径。

## 33.2 手写FFmpeg编译脚本 进行编译（针对ffmpeg3.x和ffmpeg4.x版本）

[下载ffmpeg](https://links.jianshu.com/go?to=https%3A%2F%2Fffmpeg.org%2Fdownload.html%23releases)

不同的ffmpeg版本使用编译时需要的ndk版本也会有不同，如果在上一步你下载配置的NDK是16，则下载3.x的ffmpeg。如果NDK是21，则下载4.x的ffmpeg。

这个编译的过程折腾了自己两三天的时间，为了搞清楚原因，避免后续踩坑，针对ffmpeg3.x和ffmpeg4.x都进行记录。

我们先来看下编译第三方库的一般通用步骤

1. 看READMME.md
2. 编译项目需要Makefile，如果有尝试使用make编译，如果没有，需要写makefile或者cmake构建
3. 如果报错需要解决，一般都是Makefile的一些配置文件没有生成，运行下configure
4. 生成配置文件后，再次运行make，但是编译后的文件(elf,so,a)只能在当前系统下运行。
5. 如果需要需要跑到android或者ios需要交叉编译
6. 需要给configure传一些脚本编译参数(关键！)

**不要去copy，一定要手写**，原因有三：

1. 通过手写，搞清楚每个配置的含义
2. 避免路径不同导致编译失败。
3. 避免TAB和空格的转换导致编译失败（这个很坑！！）

下面我们分别编写下ffmpeg3.x和ffmpeg4.x对应的配置编译脚本。

### 33.2.1 针对ffmpeg3.x的编译脚本**

ffmpeg3.x_build.sh



```bash
#用于编译android平台的脚本
#!/bin/bash

#定义几个变量

ARCH=arm
CPU=armv7-a

#so生成的对应路径
PREFIX = $(pwd)/android 

#注意，这里就是上一小节1.3中说的 安装android交叉编译工具链对应的路径
ANDROID_TOOLCHAINS_PATH=$NDK_PATH/android-toolchains/android-19/arch-arm
CROSS_PREFIX=$ANDROID_TOOLCHAINS_PATH/bin/arm-linux-androideabi-
SYSROOT=$NDROID_TOOLCHAINS_PATH/sysroot


#执行.configure文件 （./configure）
#运行./configure --help查看配置 裁剪
#换行后不要用空格，可以使用tab

echo "start build ffmpeg for $ARCH"
./configure --prefix =${PREFIX} \
            #编译动态库
            --disable-static \
            --enable-shared \
            #根据需要进行裁剪
            --enable-samall \
            --disable-programs \
            --disable-ffmpeg \
            --disable-ffplay \
            --disable-ffprobe \
            --disable-doc \
            --disable-ffserver \
            #指定cpu架构
            --arch=$ARCH \
            指定cpu类型
            --cpu=$CPU \
            #指定交叉编译工具目录
            --cross-prefix=${CROSS_PREFIX} \
            #开启交叉编译
            --enable-crosss-compile \
            #指定目标平台为linux（android内核也是基于linux的ffmpeg早期版本目标平台还不可以指定为android）
            --target-os=linux \
            #配置编译环境c语言的头文件环境
            --sysroot=$SYSROOT \
            --extra-cflags="-Os -fpic" \ #生成与位置无关的代码

make clean
make
make install

echo "end build ffmpeg for $ARCH"
```

编译生成的so的名称版本号在后面，这不符合在android中使用so的命名规范，为此修改configure脚本，替换以下四行内容，重新编译生成对应的so。

修改生成so的文件名称，andorid方便使用

在configure文件中，搜索build set找到对应的位置



```bash
# SLIBNAME_WITH_MAJOR='$(SLIBNAME).$(LIBMAJOR)'
# LIB_INSTALL_EXTRA_CMD='$$(RANLIB) "$(LIBDIR)/$(LIBNAME)"'
# SLIB_INSTALL_NAME='$(SLIBNAME_WITH_VERSION)'
# SLIB_INSTALL_LINKS='$(SLIBNAME_WITH_MAJOR) $(SLIBNAME)'
# 替换成如下
SLIBNAME_WITH_MAJOR='$(SLIBPREF)$(FULLNAME)-$(LIBMAJOR)$(SLIBSUF)'
LIB_INSTALL_EXTRA_CMD='$$(RANLIB)"$(LIBDIR)/$(LIBNAME)"'
SLIB_INSTALL_NAME='$(SLIBNAME_WITH_MAJOR)'
SLIB_INSTALL_LINKS='$(SLIBNAME)'
```

### 33.2.2  针对ffmpeg4.x的版本



```bash
#!/bin/bash

#这里每个使用上一小节1.3中安装android交叉编译工具链对应的路径，我们采用手动指定的toolchain，注意要根据不同的API和不同的ABI进行不同的配置。
#另外注意 darwin-x86_64是mac上对应的文件夹名称,linux上对应的是linux-x86_64，需要根据自己编译的环境具体确认。

API=21
TOOLCHAIN=$NDK_PATH/toolchains/llvm/prebuilt/darwin-x86_64/

#armv8-a
ARCH=arm64
CPU=armv8-a
#r21版本的ndk中所有的编译器都在/android-ndk-r21e/toolchains/llvm/prebuilt/darwin-x86_64/目录下（clang）
#配置.c文件的编译工具
CC=$TOOLCHAIN/bin/aarch64-linux-android$API-clang

#.cpp文件的编译工具
CXX=$TOOLCHAIN/bin/aarch64-linux-android$API-clang++

#头文件环境用的不是/android-ndk-r21e/sysroot,而是编译器//android-ndk-r21e/toolchains/llvm/prebuilt/darwin-x86_64/sysroot

SYSROOT=$NDK_PATH/toolchains/llvm/prebuilt/darwin-x86_64/sysroot

#交叉编译工具目录,对应关系如下(我们这里使用的是armv8a，根据自己的需要进行配置)
# armv8a -> arm64 -> aarch64-linux-android-
# armv7a -> arm -> arm-linux-androideabi-
# x86 -> x86 -> i686-linux-android-
# x86_64 -> x86_64 -> x86_64-linux-android-
CROSS_PREFIX=$TOOLCHAIN/bin/aarch64-linux-android-

#输出目录
PREFIX=$(pwd)/android/$CPU
OPTIMIZE_CFLAGS="-march=$CPU"

# 把configure，make写到一个function中（方便编译不同ABI时复用）
# 运行./configure --help查看配置 裁剪
# 换行后不要用空格，可以使用tab

function build_android
{
echo "CC is $CC"
echo "CROSS_PREFIX is $CROSS_PREFIX"
./configure \
    #指定输出路径
    --prefix=$PREFIX \
    #编译动态库
    --enable-shared \
    --disable-static \
    #根据需要进行裁剪
    --disable-ffmpeg \
    --disable-ffplay \
    --disable-ffprobe \
    --disable-avdevice \
    --disable-doc \
    --disable-symver \
    #指定交叉编译工具目录
    --cross-prefix=$CROSS_PREFIX \
    #指定目标平台为android
    --target-os=android \
    #指定cpu架构
    --arch=$ARCH \
    #指定cpu类型
    --cpu=$CPU \
    #指定c、cpp语言编译器
    --cc=$CC \
    --cxx=$CXX \
    #开启交叉编译
    --enable-cross-compile \
    #配置编译环境c语言的头文件环境
    --sysroot=$SYSROOT \
    --extra-cflags="-Os -fpic $OPTIMIZE_CFLAGS" \

make clean
make
make install

}

build_android
```

Ffmpeg4.x的版本编译出来命名直接时正确的（但不带版本号），可以不用修改configure直接使用生成的so。

## 33.3 AS中引入使用ffmpeg动态库

首先在local.properties中指定对应的ndk.dir，之所以没有用AS自动配置的ndk版本，是因为通过自己手动配置更加灵活，指定不同的版本。

不同的camkelist版本以及gradle版本有不同的API

**针对ffmpeg3.x和android-ndk-r16b**



```bash
cmake_minimum_required(VERSION 3.4.1)

project("myffmpegdemo")
MESSAGE(STATUS "PROJECT_SOURCE_DIR =" ${PROJECT_SOURCE_DIR})
MESSAGE(STATUS "CMAKE_SOURCE_DIR =" ${CMAKE_SOURCE_DIR})

include_directories(${PROJECT_SOURCE_DIR}/ffmpeg/include)

link_directories(${CMAKE_SOURCE_DIR}/../jniLibs/armeabi-v7a)

add_library(
             native-lib
             SHARED
             native-lib.cpp )

find_library(
        log-lib
        log)

target_link_libraries(
        native-lib
        avcodec-57
        avfilter-6
        avformat-57
        avutil-55
        swresample-2
        swscale-4
        avdevice-57
        postproc-54

        ${log-lib})
```

**针对ffmpeg4.x和 android-ndk-r21e**



```bash
cmake_minimum_required(VERSION 3.4.2)

project("myffmpeg42demo")

#指定ffmpeg对应的头文件，根据自己的具体情况进行设置
include_directories(${PROJECT_SOURCE_DIR}/ffmpeg/include)



message("CMAKE_CXX_FLAGS:" + ${CMAKE_CXX_FLAGS})
message("CMAKE_SOURCE_DIR:" + ${CMAKE_SOURCE_DIR})

message("CMAKE_ANDROID_ARCH_ABI版本是:" + ${CMAKE_ANDROID_ARCH_ABI})

#4.x采用这种方式引入ffmpeg库不行，改下下面的方式
#link_directories(${CMAKE_SOURCE_DIR}/../jniLibs/armeabi)

# 引入FFmpeg的库文件，设置内部的方式引入，指定库的目录是 -L  指定具体的库-l
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -L${CMAKE_SOURCE_DIR}/../jniLibs/${CMAKE_ANDROID_ARCH_ABI}")


add_library( # Sets the name of the library.
        native-lib
        SHARED
        native-lib.cpp)


find_library( # Sets the name of the path variable.
        log-lib

        log)


target_link_libraries( # Specifies the target library.
        native-lib

        avformat avcodec avfilter avutil swresample swscale

        ${log-lib})
```

**如何适配so架构**
 每一种架构对应一种ABI，是可以向下兼容的

比如：项目里面没有armeabi-v7a的文件夹，如果你有两个文件夹armeabi和armeabi-v7a两个文件夹，armeabi里面有a.so 和 b.so, armeabi-v7a里面只有a.so，那么arm64-v8a的手机在用到b的时候发现有armeabi-v7a的文件夹，发现里面没有b.so，就报错了，如果没有armeabi-v7a文件夹，运行时发现没有适配armeabi-v7a，就会直接去找armeabi的so库，所以要么你别加armeabi-v7a,要么armeabi里面有的so库，armeabi-v7a里面也必须有。

## 33.4 遇到的问题

**1. ffbuild/config.mak: No such file or directory**



```go
./ffmpeg_build3.sh 
Unknown option "--disable-ffserver".
See ./configure --help for available options.
Makefile:2: ffbuild/config.mak: No such file or directory
Makefile:40: /tools/Makefile: No such file or directory
Makefile:41: /ffbuild/common.mak: No such file or directory
Makefile:94: /libavutil/Makefile: No such file or directory
Makefile:94: /ffbuild/library.mak: No such file or directory
Makefile:96: /fftools/Makefile: No such file or directory
Makefile:97: /doc/Makefile: No such file or directory
Makefile:98: /doc/examples/Makefile: No such file or directory
Makefile:163: /tests/Makefile: No such file or directory
make: *** No rule to make target `/tests/Makefile'.  Stop.
Makefile:2: ffbuild/config.mak: No such file or directory
Makefile:40: /tools/Makefile: No such file or directory
Makefile:41: /ffbuild/common.mak: No such file or directory
Makefile:94: /libavutil/Makefile: No such file or directory
Makefile:94: /ffbuild/library.mak: No such file or directory
Makefile:96: /fftools/Makefile: No such file or directory
Makefile:97: /doc/Makefile: No such file or directory
Makefile:98: /doc/examples/Makefile: No such file or directory
Makefile:163: /tests/Makefile: No such file or directory
make: *** No rule to make target `/tests/Makefile'.  Stop.
Makefile:2: ffbuild/config.mak: No such file or directory
Makefile:40: /tools/Makefile: No such file or directory
Makefile:41: /ffbuild/common.mak: No such file or directory
Makefile:94: /libavutil/Makefile: No such file or directory
Makefile:94: /ffbuild/library.mak: No such file or directory
Makefile:96: /fftools/Makefile: No such file or directory
Makefile:97: /doc/Makefile: No such file or directory
Makefile:98: /doc/examples/Makefile: No such file or directory
Makefile:163: /tests/Makefile: No such file or directory
make: *** No rule to make target `/tests/Makefile'.  Stop.
```

解决方案：一般是configure配置出错了，或者换行带有空格或者copy的tab被转为了空格导致，解决方案搞清楚ffmpeg编译脚本的意义，按照规范手写。

**2.  4.x的版本编译的so，在编译时报错 incompatible target**



```jsx
/Users/yabin/tools/android-ndk-r21e/toolchains/llvm/prebuilt/darwin-x86_64/lib/gcc/arm-linux-androideabi/4.9.x/../../../../arm-linux-androideabi/bin/ld: error: /Users/yabin/work/tmp/myffmpegDemo/app/src/main/cpp/../jniLibs/armeabi-v7a/libswscale.so: incompatible target
```

把ndk的版本从21换成16也一样的错误

# ndk.dir=/Users/yabin/tools/android-ndk-r21e

ndk.dir=/Users/yabin/tools/android-ndk-r16b



```jsx
/Users/yabin/tools/android-ndk-r16b/toolchains/arm-linux-androideabi-4.9/prebuilt/darwin-x86_64/lib/gcc/arm-linux-androideabi/4.9.x/../../../../arm-linux-androideabi/bin/ld: error: /Users/yabin/work/tmp/myffmpegDemo/app/src/main/cpp/../jniLibs/armeabi-v7a/libswscale.so: incompatible target
```

解决方案：
 换成了3.3.9版本的ffmpeg的编译产物，可以了
 根本原因在于不同NDK版本对应的Cmakelist中引入so的方式不同。

**3. AS引入so后在armeabi中加入so，正确配置后，编译报错“armeabi is no longer supported.  Use armeabi-v7a”**



```tsx
CMake Error at /Users/yabin/tools/android-ndk-r21e/build/cmake/android.toolchain.cmake:174 (message):
armeabi is no longer supported.  Use armeabi-v7a.


Error while executing process cmake/3.10.2.4988404/bin/cmake with arguments 
  Unknown argument -


  abiFilters Collection contains no element matching the predicate.
  No valid Native abi found to request!
```

解决方案：使用ndk21不支持armeabi，改为armeabi-v7a

**4. 【ffmpeg】编译时报错：error: undefined reference to 'av_codec_next(AVCodec const\*)’***

解决方案：ffmpeg库的接口都是c函数，其头文件也没有extern "C"的声明，所以在cpp文件里调用ffmpeg函数要加extern “C” 。

## 33.5 资料

1. [1.0-FFMPEG-Android利用ndk(r20)编译最新版本ffmpeg4.2.1] ([https://juejin.cn/post/6844903945496690696](https://links.jianshu.com/go?to=https%3A%2F%2Fjuejin.cn%2Fpost%2F6844903945496690696))
2. [Shell 脚本 - 自己动手编译 FFmpeg](https://www.jianshu.com/p/617740ce5b78)
3. [AndroidStudio NDK的接入FFmpeg填坑记](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fxv545262342%2Farticle%2Fdetails%2F70174587)
4. [雷神-最简单的基于FFmpeg的移动端例子：Android HelloWorld](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fleixiaohua1020%2Farticle%2Fdetails%2F47008825)
5. [【ffmpeg】编译时报错：error: undefined reference to `av...](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fu010168781%2Farticle%2Fdetails%2F104775295%2F)
6. [重新编译使用CMAKE的旧项目的问题处理](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.freesion.com%2Farticle%2F9096748286%2F)
7. [Android 关于arm64-v8a、armeabi-v7a、armeabi、x86下的so文件兼容问题](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.cnblogs.com%2Fjanehlp%2Fp%2F7473240.html)

## 33.6 收获

1. 针对NDK16和NDK21以及ffmpeg3.x和ffmpeg4.x配置编译
2. 手写ffmpeg配置和编译脚本
3. AS中引用使用ffmpeg动态库以及不同NDK版本的引入方式
4. 了解cpu架构的兼容性
5. 遇到的问题解决
    ffmpeg编译是比较折腾的过程，不同的NDK版本、ffmpeg版本有不同的配置和编译方式，还记得去年年初的时候折腾过一两天没搞定，这次有花费了一两天终于把ffmpeg3.x和ffmpeg4.x的都编译生成对应的so，并在AS中引入使用。