---
title: Android面试抱佛脚四：JNI 了解一下？
---

![标题图](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6f235c9e748644cfac1f76622dba2e76~tplv-k3u1fbpfcp-zoom-1.image)


> android面试中老是会问jni，但是我在小厂搬砖多年，可还没咋用过啊
> 哭~~~~
没用过那就了解一下吧。
```
编写：guuguo  校对：guuguo
```
# 名词解释
- `c++头文件`: 头文件用来放置对应`c++`方法的声明，其实它的内容跟 .cpp 文件中的内容是一样的，都是 C++ 的源代码。但头文件不用被编译。头文件可以通过`#include`被包含到.cpp文件中。include仅仅是复制头文件的定义代码到.cpp文件中。所以头文件用来放置声明，而不是定义。因为多个源文件直接包含定义的话会有定义冲突，而声明就不会。(头文件也可以包含定义，但是尽量不要，如果 需要，通过`#ifndef...#endif `让编译器判断个名字是否被定义，再决定要不要继续编译后续的内容)
- `JNI （Java Native Interface，Java本地接口）`是一种编程框架，使得Java虚拟机中的Java程序可以调用本地应用/或库，也可以被其他程序调用。 
- `CMake` 是一个跨平台构建工具，支持C/C++/Java等语言的工程构建。本文中用来编译c++代码。
# 这篇文章讲什么?
Android 系统中有大量的实现都是native实现的，中间通过JNI进行java层调用。学会JNI的使用，不光是能为我们开发和面试提供助力，还能为我们理解android 系统源码的基础多加两块砖。
说明一下这篇文章的内容和目的：
1. 了解JNI 在开发中的基础使用
2. Java 代码和 c++ 的native 方法链接原理
3. JNI 框架是啥，都有哪些东西
4. Ndk 是什么东西？

弄明白这四个小点，对于JNI也就有了初步的理解，在要利用其进行开发的时候也能信手拈来。
# JNI 使用的小栗子（静态注册）
jni注册方式分静态注册和动态注册，
- 静态注册：根据函数名找到对应的JNI函数,样式为`Java_包名_类名_方法名`
- 动态注册：当我们使用`System#loadLibarary`方法加载so库的时候，Java虚拟机会找到`JNI_OnLoad`函数并主动调用。所以我们可以在`JNI_OnLoad` 调用 `jniRegisterNativeMethods`进行方法的动态注册。(先不学习该方式，欲了解可google)

下面我们就讲一下静态注册先：
1. ## 创建demo jni sdk模块
我们创建一个`sdk`模块，承载native和jni代码，目录结构如下：
![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/49e61f5da6ad466ba618a7e8244a455d~tplv-k3u1fbpfcp-zoom-1.image)

图中展示的主要目录如下：
- `src/main/java`    java源码
- `src/main/jni`    native源码
- `src/main/jni/CMakeLists.txt`    cmake的配置文件

并且在build.gradle 中配置好`jni`源码路径:
```groovy
sourceSets {
    main {
        jni.srcDirs = ['src/main/jni']
    }
}
```
1. ## 定义native java 方法
在kotlin 中，使用关键字external标识该方法是JNI方法。在调用该方法的时候，`Java_包名_类名_方法名`的c++函数。
我们先来创建JNI入口java类 `JNI.java`，定义好java的native方法。方法如下：
```kotlin
package top.guuguo.myapplication
class JNI {
    /**返回签名后的字符串*/
    external fun signString(str: String): String
    companion object {
        ///实例的创建一定要在native代码加载之后，如本例的 
        ///System.loadLibrary("jni-test")
        val instance by lazy { JNI() }
    }
}
```
我们定义了一个简单的native方法`signString`，模拟对字符串进行签名的方法。
1. ## 生成对对应的头文件
java中提供了javah 工具。通过他可以自动生成native方法对应c++的头文件。通过javah -h 看看该工具的使用说明：
```shell
javah -h
用法: 
  javah [options] <classes>
其中, [options] 包括:
  -o <file>                输出文件 (只能使用 -d 或 -o 之一)
  -d <dir>                 输出目录
  -v  -verbose             启用详细输出
  -h  --help  -?           输出此消息
  -version                 输出版本信息
  -jni                     生成 JNI 样式的标头文件 (默认值)
  -force                   始终写入输出文件
  -classpath <path>        从中加载类的路径
  -cp <path>               从中加载类的路径
  -bootclasspath <path>    从中加载引导类的路径
<classes> 是使用其全限定名称指定的
(例如, java.lang.Object)。
```
使用方式如下： `-cp`  等同于`-classpath`，用来指定要生成头文件的`class`文件路径
```
javah -d app/src/main/cpp/header -cp "./app/build/tmp/kotlin-classes/debug/"  top.guuguo.myapplication.JNI
```
可以看到命令执行过后，.h文件被成功生成了
![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b4e20aa1ac664855b06fe3ca135412e6~tplv-k3u1fbpfcp-zoom-1.image)
有了`.h` jni 声明文件后，我们在 jni.cpp中完成对应方法的实现，代码如下：
```c++
#include <stdio.h>
#include <stdlib.h>
#include <string>
#include "header/top_guuguo_myapplication_JNI.h"

JNIEXPORT jstring JNICALL Java_top_guuguo_myapplication_JNI_signString(JNIEnv *env, jobject obj, jstring jStr) {
    const char *cstr = env->GetStringUTFChars(jStr, NULL);
    std::string str = std::string(cstr);
    env->ReleaseStringUTFChars(jStr, cstr);
    std::string cres = "signed：" + str;
    jstring jres = env->NewStringUTF(cres.c_str());
    return jres;
}
```
方法的定义实现很简单，只是对传入的字符串前面拼接了`signed:`字符串。
1. ## 完善CmakeList.txt 和 build.gradle 编译.so产物
对于native源码的编译，当前有两种方案：cmake 和 ndk-build。CMake会更加流行一些，现在介绍一下CMake。
`CMake` 是一个跨平台构建工具，支持C/C++/Java等语言的工程构建。通过配置`CMake` 构建脚本`CMakeLists.txt`，我们可以利用`CMake`命令做好自定义的编译工作。
这是cmake使用的主要指令
- `set(all_src "./src")`：该指令可以定义名为`all_src`的变量值
- `add_library`：该指令的主要作用就是将指定的源文件生成链接文件，然后添加到工程中去
#### `CMakeLists.txt`
我们编辑一下该配置文件，使用如下内容
```shell
# Copyright (c) 2019 - 2020 The Alibaba DingTalk Authors. All rights reserved.

PROJECT(jni-test)
cmake_minimum_required(VERSION 3.4.1)

# 对一些c++编译期标识 赋值
#set(CMAKE_CXX_COMPILER      "clang++" )         # 显示指定使用的C++编译器
#set(CMAKE_CXX_FLAGS   "-std=c++11 -O2")             # c++11
#set(CMAKE_CXX_FLAGS   "-g")                     # 调试信息
#set(CMAKE_CXX_FLAGS   "-Wall")                  # 开启所有警告
#set(CMAKE_CXX_FLAGS_DEBUG   "-O0" )             # 调试包不优化
#set(CMAKE_CXX_FLAGS_RELEASE "-O2 -DNDEBUG " )   # release包优化
set(CMAKE_CXX_FLAGS_RELEASE "-std=c++11 -O2 ")
set(CMAKE_CXX_FLAGS_DEBUG "-std=c++11 -O2 ")

# 对变量 SRC_ROOT 赋值
set(SRC_ROOT "./")

# 遍历目录下直属的所有.cpp文件保存到变量中
file(GLOB all_src
        "${SRC_ROOT}/*.hpp"
        "${SRC_ROOT}/*.cpp"
        "${SRC_ROOT}/src/*.h"
        "${SRC_ROOT}/src/*.hpp"
        "${SRC_ROOT}/header/*.h"
        "${SRC_ROOT}/header/*.hpp"
        )
# 将源码文件添加到编译动态库中
add_library(jni-test SHARED ${all_src})
```
#### build.gradle 添加native配置：
```groovy
defaultConfig {
    /**...*/
    externalNativeBuild {
        cmake {
            ///编译目标名
            targets 'jni-test'
            //预编译行为配置 :-fexceptions 启用异常处理
            cppFlags "-std=c++11 -fexceptions -frtti"
            arguments "-DANDROID_STL=c++_shared"
        }
    }
}
externalNativeBuild {
    cmake {
        version '3.6.0'
        path 'src/main/jni/CMakeLists.txt'
    }
}
```
在以上代码中指定好一些必要参数，以及`cmake`版本和配置文件路径
#### 编译：
接下来的编译中会自动 编译出相关类库，也可以通过以下的gradle命令直接打包出对应的so库和aar包
```shell
./gradlew :sdk:aR
```
也就是使用`aR(assembleRelease)`命令编译release包，在`build/intermediates/cmake/release`中能找到对应产物。
1. ## 简单c++方法调用
完成了定义，我们简单实现一下调用：
```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main2)
        System.loadLibrary("jni-test")
        findViewById<Button>(R.id.button).setOnClickListener {
           Toast.makeText(this,JNI.instance.signString("hello world"),Toast.LENGTH_LONG).show()
        }
    }
}
```
我们在点击按钮之后，直接弹出吐司展示签名后的字符串。
#### **这一块有一点需要注意！！**
获取JNI实例的步骤，需要在`System.loadLibrary`之后。
这样才能正确调用到对应的`native`方法。
## 小结：
至此，最小化实现的一个jni样例就完成了，实现了native方法定义以及java对其的调用。
以此为基础，我们在未来能深入很多
- 我们能够慢慢了解跨平台`native` sdk 如何在安卓中使用。
- 能够为阅读aosp源码增加自己的基础功

# Java 代码和 c++ 的native 方法如何连接起来
java调用native方法的时候，由art虚拟机对应做特殊处理。
参考[Android ART执行类方法的过程](https://www.jianshu.com/p/2ff1b63f686b)，虚拟机在执行方法的时候判断是否native方法，执行。
客户端的实现很简单，就是上面提到的静态注册和动态注册方式。

# JNI 框架是啥，都有哪些东西?
JNIEnv 表示 Java 调用 native 语言的环境，是一个封装了几乎全部 JNI 方法的指针。
我们查看 `jni.h `的源码(aosp源码路径`source/libnativehelper/include_jni/jni.h`)。
找到`JNIEnv`的定义：`typedef _JNIEnv JNIEnv;`
可以看到其实是`_JNIEnv`类型的别名。看看_JNIEnv结构的源码：
```c++
truct _JNIEnv {
    /* do not rename this; it does not seem to be entirely opaque */
    const struct JNINativeInterface* functions;
    #if defined(__cplusplus)
    jint GetVersion()
    { return functions->GetVersion(this); }
    jclass DefineClass(const char *name, jobject loader, const jbyte* buf,
        jsize bufLen)
    { return functions->DefineClass(this, name, loader, buf, bufLen); }
   // ...
    }
```
可以看出所有的`JNIEnv`方法都是间接调用的`JNINativeInterface`的方法，只是对`JNINativeInterface`结构体的一层封装。
我们JNI的大多数操作都是通过其进行。

# NDK是啥，和jni什么关系？
```
ndk:Native Development Kit
```
Android NDK 支持使用 [CMake](https://cmake.org/) 编译应用的 C 和 C++ 代码。
NDK是一系列工具的集合。
- NDK提供了一系列的工具，帮助开发者快速开发C（或C++）的动态库，并能自动将so和java应用一起打包成apk。这些工具对开发者的帮助是巨大的。
- NDK集成了交叉编译器，并提供了相应的mk文件隔离CPU、平台、ABI等差异，开发人员只需要简单修改mk文件（指出“哪些文件需要编译”、“编译特性要求”等），就可以创建出so。
- NDK可以自动地将so和Java应用一起打包，极大地减轻了开发人员的打包工作。

NDK提供了一份稳定、功能有限的API头文件声明。包含有：C11标准库（libc）、标准数学库（libm）、c++17库、Log库（liblog）、压缩库（libz）、Vulkan渲染库（libvulkan）、openGl库（libGLESv3）等。
NDK可以为我们生成C/C++动态链接库。 我们对于native的开发是基于ndk的开发。

**ndk和jni没什么关系，只是基于ndk开发的动态库，需要通过jni和java进行沟通。**

# 最后
经过这一节的学习，接下来面试中碰到`jni`问题的话，总算可以说个123了：
1. jni的native代码怎么关联？通过静态注册和动态注册方式。
2. 加载so库需要注意什么？System.loadLibrary之后再获取实例调用native方法才能调用到对应实现。
3. 怎么构建so库？ndk支持通过cmake实现代码编译构建。
4. ndk和jdk的区别？

> 只有学习才能是我成长，只有学习才能是我进步，我要好好学习，为建设祖国贡献一份力量~~~

# 参考文章：
- [Android JNI介绍（八）- CMakeLists的使用](https://juejin.cn/post/6844904064820477960)
- [JNI方法注册及加载原理分析](https://www.jianshu.com/p/4ab830aa935e)
- [JNI实现源码分析【四 函数调用】](https://cloud.tencent.com/developer/article/1193430)
- [Android ART执行类方法的过程](https://www.jianshu.com/p/2ff1b63f686b)