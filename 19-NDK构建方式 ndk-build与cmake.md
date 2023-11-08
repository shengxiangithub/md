# 19-NDK构建方式 ndk-build与cmake

## 19.0 目录

1. ndk-build和makefile
2. cmake和cMakeLists.txt
3. 资料
4. 收获

AS 2.2 +默认使用CMake进行 NDK 编译，我们这篇主要学习实践也是CMake，那么为什么要带ndk-build呐？

1. CMake对编辑构建过程做了高级的封装，方便调用者使用，但是Cmake并不直接建构出最终的so，而是产生标准的建构文档Makefile，然后再用一般的建构方式使用。
2. 早期的项目有些Makefile和cmakelists.txt都存在。我们至少有能看懂Makefile吧。

**为什么需要makefile？**
 要回答这个问题，就要了解so是如何生成的？步骤如下

1. 预处理：宏展开，宏替换，展开include                     gcc -E -o xxx.i xxx.c
2. 预编译：gcc检查代码的规范性，把代码编译成汇编  gcc -S -o xxx.s xxx.i
3. 汇编: 把.s汇编文件编译成.o二进制文件                      gcc -c -o xxx.o xxx.s
4. 链接：链接其他so，合并数据段                                   gcc  -o xxx  xxx.o

涉及到多个文件多个目录的编译时，要对所有的文件执行上述操作，如果某个文件发生变化，要做到相关联的文件重新编译链接，更是十分麻烦，而makefile的出现就是解决这一痛点，通过makefile配置和规则，实现编译的自动化，提升效率。在AndroidStudio上可以通过配置Android.mk以及Application.mk，使用ndk-build进行编译成动态或者静态库。

## 19.1 Makefile文件解析

### 19.1.1 Makefile规则介绍

一个完整的 Makefile 中，包含了 5 个东西：显式规则、隐含规则、变量定义、指示符和注释。

**显示规则**



```bash
target… : prerequisites… 
【Tab】command
```

target:
 规则的目标，通常是最后需要生成的文件名或者中间产物。可以是是.o文件，也可以是最后的可执行文件。
 也可以是“伪目标”，一个make执行的动作的名称，比如 “clean”，伪规则需要用.PHONY 申明

prerequisites:
 规则的依赖。生成target所需的文件名列表。

command:
 规则的命令行。 任意的shell或者可在shell下执行的程序。
 一个规则可以有多个命令行，但每一条命令占一行。
 注意： 每一个命令行必须以【Tab】字符开始，它告诉make此行是一个命令行

默认的情况下，make执行的是Makefile中的第一个规则，此规则的第一个目标称
 之为“最终目的”

1. 目标文件不存在，使用其描述规则创建它；
2. 目标文件存在，目标文件所依赖的.c 源文件、.h 文件中的任何一个比目标
    文件“更新”（在上一次 make 之后被修改）。则根据规则重新编译生成它；
3. 目标.o 文件存在，目标.o 文件比它的任何一个依赖文件（的.c 源文件、.h 文件）“更新”（它的依赖文件在上一次 make 之后没有被修改），则什么也不做。

**伪目标**



```css
.PHONY:xxx
```

使用伪目标有两点原因：

1. 此目标的目的为了执行执行一些列命令，而不需要创建这个目标。
2. 提高执行 make 时的效率

**变量与函数**



```jsx
“$(objects)”
$(xx）：取变量的值
```

**模式规则的自动化变量**



```ruby
$@:表示规则中的目标
$^:表示规则中所有的依赖文件，组成一个列表,以空格隔开，如果这个列表有重复项则消除重复
$<:表示规则中第一个依赖文件，如果运行在模式套用中，相当于依次取出依赖条件，套用该模式规则。
```

**两个常用函数**



```jsx
$(wildcard PATTERN) 列出当前目录下所有符合模式“PATTERN”格式的文件名

wildcard ： 把当前文件加下的所有*.c文件都赋值为src
src = $(wildcard *.c)
```



```jsx
模式替换函数—patsubst
obj = $(patsubst %c,%o,$(src))
```

**指示符**
 include: 在一个 Makefile 中包含其它的 makefile 文件

“-include”：忽略由于包含文件不存在或者无法创建时的错误提示

### 19.1.2 Android.mk的用法

基本格式如下



```ruby
LOCAL_PATH := $(call my-dir)  
include $(CLEAR_VARS) 
include $(LOCAL_PATH)/xxx/xxx.mk
 
LOCAL_MODULE    := xxx 
LOCAL_C_INCLUDES := xxx
LOCAL_SRC_FILES := xxx.c  
LOCAL_SHARED_LIBRARIES := xxx  
LOCAL_LDLIBS := xxx
 
include $(BUILD_SHARED_LIBRARY) 
```

下面逐一对其解析



```go
LOCAL_PATH := $(call my-dir)  
```

my-dir是一个宏， 返回Android.mk的所在的文件夹路径。
 call 使用makefile的一个函数，LOCAL_PATH := $(call my-dir) 的意义是，把当前文件所在的文件夹路径赋值给LOCAL_PATH。



```ruby
include $(CLEAR_VARS)
```

CLEAR_VARS 负责清理除上面刚定义的LOCAL_PATH 外的其他LOCAL_xxx.



```ruby
include $(LOCAL_PATH)/xxx/xxx.mk
```

如果遇到比较大的项目，会在自文件夹下建立对应的makefile，方便开发维护，在编译时，痛殴include把他们引用进来一起编译。

定义一些LOCAL_变量，含义如下



```css
LOCAL_MODULE    : 生成so的名称，不带前缀lib和后缀.so
LOCAL_MODULE_PATH : 生成so的输出目标地址
LOCAL_C_INCLUDES：包含的头文件，如果多个用\来进行换行
LOCAL_SRC_FILES ：需要编译的源文件
LOCAL_SHARED_LIBRARIES : 会生成依赖关系，当库不存在时会去编译这个库
LOCAL_LDLIBS  : 链接的库不产生依赖关系，用于不需要重新编译的库
```



```ruby
include $(BUILD_SHARED_LIBRARY) 
```

BUILD_SHARED_LIBRARY：是Build System提供的一个变量，指向一个GNU Makefile Script。

### 19.1.3 Application.mk

基本格式如下



```go
APP_PLATFORM = android-16
APP_ABI := armeabi-v7a arm64-v8a x86
APP_STL := c++_static
APP_CPPFLAGS := -exceptions -fno-rtti
APP_OPTIM := release
```

下面对其解析

APP_PLATFORM :声明构建此应用所面向的 Android API 级别，并对应于应用的 minSdkVersion

使用 Gradle 和 externalNativeBuild 时，不应直接设置此参数,有minSdkVersion确定可支持的下限。

APP_ABI : Application Binary Interface 设置为特定cpu架构生成对应的so
 注意： Gradle 的 externalNativeBuild 会忽略该APP_ABI

APP_STL : 用于此应用的 C++ 标准库。
 默认情况下使用 system STL。其他选项包括 c++_shared、c++_static 和 none

APP_CPPFLAGS :为项目中的所有 C++ 编译传递的标记

APP_OPTIM: release 或 debug.

有了ndk-build构建工具以及makefile编译配置文件为什么又引入了Cmake构建工具呐？我们来一起继续学习。

## 19.2 Cmake构建

### **什么是Cmake**

 “CMake”：”cross platform make”，是个开源的跨平台自动化建构系统，它用配置文件CMakeLists.txt控制建构过程（build process）。

如果工程很大，相关性比较强，每个文件夹下都有对应的makefile，就会变得相对繁琐，cmake的出现就是为了解决这样的问题。cmake是为了生成makefile而存在，这样我们就不需要再去写makefile了，只需要写简单的CMakeLists.txt即可

### **常用变量**

 PROJECT_SOURCE_DIR：工程的根目录

PROJECT_BINARY_DIR：运行 cmake 命令的目录，通常是 ${PROJECT_SOURCE_DIR}/build

PROJECT_NAME：返回通过 project 命令定义的项目名称

CMAKE_CURRENT_SOURCE_DIR：当前处理的 CMakeLists.txt 所在的路径

CMAKE_CURRENT_BINARY_DIR：target 编译目录

CMAKE_CURRENT_LIST_DIR：CMakeLists.txt 的完整路径

CMAKE_CURRENT_LIST_LINE：当前所在的行

CMAKE_MODULE_PATH：定义自己的 cmake 模块所在的路径，SET(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)，然后可以用INCLUDE命令来调用自己的模块

EXECUTABLE_OUTPUT_PATH：重新定义目标二进制可执行文件的存放位置

LIBRARY_OUTPUT_PATH：重新定义目标链接库文件的存放位置

### **常用指令**

1. 赋值指令

```css
set(SRC_LIST main.cpp test.cpp)
```

2. 打印指令
    message(${PROJECT_SOURCE_DIR})
    message("build with debug mode")

3. 指定 cmake 的最小版本

```css
cmake_minimum_required(VERSION 3.10.2)
```

4. 设置项目名称

```undefined
project(xxx)
```

它会引入两个变量 xxx_BINARY_DIR 和 xxx_SOURCE_DIR，同时，cmake 自动定义了两个等价的变量 PROJECT_BINARY_DIR 和 PROJECT_SOURCE_DIR。

5. 生成动态库

```css
add_library(xxx SHARED xxx.cpp) 
```

6. 指定源文件

```undefined
aux_source_directory(. SRC_LIST) 
```

搜索当前目录下所有的cpp源代码文件并将列表存储在一个变量中。

7. 指定一定规则的源文件

```bash
file(GLOB SRC_LIST "*.cpp" "xxx/*.cpp")
```

8. 查找指定的库文件

```bash
find_library( log-lib
              log )
```

9. 设置 target 需要链接的库

```bash
target_link_libraries(native-lib
                       ${log-lib} )
```

### **示例如下**

```bash
cmake_minimum_required(VERSION 3.10.2)

project("mediajourney")

message(${PROJECT_SOURCE_DIR})
message(${PROJECT_NAME} "PROJECT_BINARY_DIR :"${PROJECT_BINARY_DIR})

message(${CMAKE_CURRENT_SOURCE_DIR} \n ${CMAKE_CURRENT_BINARY_DIR})
message("--->")

message(${mediajourney_SOURCE_DIR})


add_library(
             native-lib
             SHARED
             native-lib.cpp )

find_library(
              log-lib
              log )
target_link_libraries(
                       native-lib
                       ${log-lib} )
```

运行后可以如下路径中看到运行结果
 .cxx/cmake/debug/${ABI}/build_output.txt

```csharp
CMake Warning (dev) at /Users/yabin/work/av/mediaJourneyNative/app/src/main/cpp/CMakeLists.txt:7:
  Syntax Warning in cmake code at column 47

  Argument not separated from preceding token by whitespace.
This warning is for project developers.  Use -Wno-dev to suppress it.

/Users/xxx/work/av/mediaJourneyNative/app/src/main/cpp
mediajourneyPROJECT_BINARY_DIR :/Users/xxx/work/av/mediaJourneyNative/app/.cxx/cmake/debug/x86
/Users/xxx/work/av/mediaJourneyNative/app/src/main/cpp
/Users/xxxx/work/av/mediaJourneyNative/app/.cxx/cmake/debug/x86
--->
/Users/xxx/work/av/mediaJourneyNative/app/src/main/cpp
--->
Configuring done
```

## 19.3 资料

《GNU_makefile中文手册》
 《cmake实战》

[CMakeLists.txt 语法介绍与实例演练]: https://blog.csdn.net/afei__/article/details/81201039(https://links.jianshu.com/go?to=https%3A%2F%2Fblog.csdn.net%2Fafei__%2Farticle%2Fdetails%2F81201039)

## 19.4 收获

1. 了解了ndk-build和cmake编译构建工具的流程
2. 了解makefile的规范和基本使用
3. 了解cmaklists.txt的基本使用