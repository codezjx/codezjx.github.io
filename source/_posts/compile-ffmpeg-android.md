---
title: 在Mac上编译基于Android平台的FFmpeg源码
date: 2019-06-21 20:23:46
categories: FFmpeg
tags: [Android, FFmpeg, NDK]
---

这段时间开始研究音视频编解码的相关知识，自然少不了学习FFmpeg这个开源项目。网上编译FFmpeg源码有很多教程，但是大部分都过时了，编译的时候还会遇到一大堆错误，踩了不少坑。因此总结了此文章，方便大家后续查阅。

## 下载NDK和FFmpeg

编译Android平台的FFmpeg需要下载NDK和FFmpeg源码：

首先下载NDK，目前官方最新稳定版是r20的版本，但是建议不要下最新的。这里我们为了顺利编译，可以下载r17及以下的版本，这里我们下载了[r17c][1]版本，为什么？请看后面的报错处理环节。

然后去[FFmepg官网][2]下载最新的源码，目前最新版是[ffmpeg-4.1.3.tar.bz2][3]。

## 编写configure配置脚本

编译FFmpeg源码需要通过configure脚本来进行配置，后期根据项目需求可对FFmpeg进行各种裁剪，因此我们可以通过配置脚本来实现。通过指令`./configure --help`我们可以查看所支持的配置项，网上很多文章有介绍这里就不展开了。

新建`build_android.sh`文件，并输入以下脚本内容来帮助我们编译FFmpeg。注意更新下第一行的NDK路径修改为你本地下载的r17c路径即可：

```shell
#!/bin/bash
NDK=/Users/codezjx/Android/android-ndk-r17c
SYSROOT=$NDK/platforms/android-21/arch-arm
ISYSROOT=$NDK/sysroot
ASM=$ISYSROOT/usr/include/arm-linux-androideabi
TOOLCHAIN=$NDK/toolchains/arm-linux-androideabi-4.9/prebuilt/darwin-x86_64
PREFIX=$(pwd)/android/armv7-a
CROSS_PREFIX=$TOOLCHAIN/bin/arm-linux-androideabi-

build_android()
{
    ./configure \
    --prefix=$PREFIX \
    --enable-shared \
    --disable-static \
    --disable-doc \
    --disable-ffmpeg \
    --disable-ffplay \
    --disable-ffprobe \
    --disable-avdevice \
    --disable-symver \
    --cross-prefix=$CROSS_PREFIX \
    --target-os=android \
    --arch=arm \
    --enable-cross-compile \
    --sysroot=$SYSROOT \
    --extra-cflags="-I$ASM -isysroot $ISYSROOT -D__ANDROID_API__=21 -Os -fpic -marm -march=armv7-a"
    make clean
    make
    make install
}

build_android
```
## 执行编译并生成.so文件

执行以下指令开始编译FFmpeg
```
$ chmod +x build_android.sh
$ ./build_android.sh
```

若最后没有报错，显示以下log，则证明编译成功，会在`android/armv7-a`下生生成我们需要的.so库和相关的头文件，这个路径也正是我们在`--prefix`中配置的路径
```
...
INSTALL libavutil/twofish.h
INSTALL libavutil/version.h
INSTALL libavutil/xtea.h
INSTALL libavutil/tea.h
INSTALL libavutil/lzo.h
INSTALL libavutil/avconfig.h
INSTALL libavutil/ffversion.h
INSTALL libavutil/libavutil.pc
```

最终`android/armv7-a`的目录结构大概是这样的：
```
├── include
│   ├── libavcodec
│   ├── libavfilter
│   ├── libavformat
│   ├── libavutil
│   ├── libswresample
│   └── libswscale
├── lib
│   ├── libavcodec.so
│   ├── libavfilter.so
│   ├── libavformat.so
│   ├── libavutil.so
│   ├── libswresample.so
│   ├── libswscale.so
│   └── pkgconfig
└── share
    └── ffmpeg
```

## 踩坑环节
由于我们这里编译的是最新的FFmpeg源码，网上的脚本很多都过时了，要不就是跟NDK版本不搭，编译的时候会遇到很多问题，这里列出我编译时遇到的一些问题，这样大家也能更清晰的知道为什么上面的`build_android.sh`要这么配置。

>Tips：在编译FFmpeg的时候难免会遇到很多问题，控制台的错误信息可能不够详细，这个时候我们可以打开`ffbuild/config.log`这个log文件查看更细致的日志信息，可帮助我们更快的定位问题。

### 错误1：C compiler test failed.
```
/Users/codezjx/Android/android-sdk-macosx/ndk-bundle/toolchains/arm-linux-androideabi-4.9/
prebuilt/darwin-x86_64/bin/arm-linux-androideabi-gcc is unable to create an executable file.
C compiler test failed.
```

在NDK升级到r18及以后，官方移除了GCC采用了Clang作为默认的交叉编译器，具体可看这里[Changelog-r18][4]。而FFmpeg的编译默认选择的是GCC来进行编译，所以当configure脚本根据路径去查找`arm-linux-androideabi-gcc`这个可执行文件的时候，发现找不到了，这也是为啥上面我们选择r17c版本的NDK来编译的原因。

### 错误2：Unknown option "--disable-ffserver"
```
Unknown option "--disable-ffserver".
See ./configure --help for available options.
```

在FFmpeg4.0.x版本后已经移除掉`--disable-ffserver`这个配置项了，如果用的是网上的旧脚本，就会报这个错误，移除掉就好。

### 错误3：error: request for member 's_addr' in something not a structure or union
```
libavformat/udp.c: In function 'udp_set_multicast_sources':
libavformat/udp.c:290:28: error: request for member 's_addr' in something not a structure or union
         mreqs.imr_multiaddr.s_addr = ((struct sockaddr_in *)addr)->sin_addr.s_addr;
                            ^
libavformat/udp.c:292:32: error: incompatible types when assigning to type '__be32' from type 'struct in_addr'
             mreqs.imr_interface= ((struct sockaddr_in *)local_addr)->sin_addr;
                                ^
libavformat/udp.c:294:32: error: request for member 's_addr' in something not a structure or union
             mreqs.imr_interface.s_addr= INADDR_ANY;
                                ^
libavformat/udp.c:295:29: error: request for member 's_addr' in something not a structure or union
         mreqs.imr_sourceaddr.s_addr = ((struct sockaddr_in *)&sources[i])->sin_addr.s_addr;
                             ^
make: *** [libavformat/udp.o] Error 1
```

如果在`build_android.sh`脚本中使用的NDK版本是r15c或者r16b，就会报这个error，所以解决方法就是升级到r17及以上的版本就能解决。

### 错误4：No such file or directory
```
CC  libavfilter/aeval.o
In file included from libavfilter/aeval.c:26:0:
./libavutil/avassert.h:30:20: fatal error: stdlib.h: No such file or directory
 #include <stdlib.h>
                    ^
compilation terminated.
make: *** [libavfilter/aeval.o] Error 1
```

这个错误是因为新版本NDK的机制引起的，因为NDK将头文件和库文件进行了分离，指定的`--sysroot`只有库文件，头文件放在NDK目录下的`sysroot`内，只需在`--extra-cflags`中添加 `-isysroot $NDK/sysroot`即可。除此之外有关汇编的头文件也进行了分离，需要根据目标平台进行指定`-I$NDK/sysroot/usr/include/arm-linux-androideabi`，将`arm-linux-androideabi`改为需要的平台就可以，后面这条也是添加到`--extra-cflags`中即可。

### 错误5：expected identifier or '(' before numeric constant
```
libavcodec/aaccoder.c: In function 'search_for_ms':
libavcodec/aaccoder.c:803:25: error: expected identifier or '(' before numeric constant
                     int B0 = 0, B1 = 0;
                         ^
libavcodec/aaccoder.c:865:28: error: lvalue required as left operand of assignment
                         B0 += b1+b2;
                            ^
libavcodec/aaccoder.c:866:25: error: 'B1' undeclared (first use in this function)
                         B1 += b3+b4;
                         ^
libavcodec/aaccoder.c:866:25: note: each undeclared identifier is reported only once for each function it appears in
make: *** [libavcodec/aaccoder.o] Error 1
```

这个错误就比较悬了，跟NDK版本的变量名定义产生了冲突，在低版本的NDK是不会出现这个错误的。网上看了有一些解决方法是直接修改变量的名字-_-|||，这种方式过于暴力，以后更新FFmpeg源码的时候极有可能会产生代码冲突，感觉不是靠谱的解决方案。后面在FFmpeg官方的Reports中找到了这个错误的描述以及解决方法，其实也很简单，就是将`--target-os=linux`修改为`--target-os=android`即可，原因还有待考量，详见这里[#7103][5]。

## 导入.so文件到Android项目

编译成功后，我们将拿到.so库和相关的头文件，这个时候就可以导入到Android项目中使用了。简单来说有以下几个步骤：

### AS新建一个Native C++类型的Project
这里为了方便测试，我们直接新建一个C++类型的Project，Android Studio会帮我们生成好Native项目的目录结构和`CMakeLists.txt`文件。

### 导入.so和include头文件
将.so和include头文件分别拷贝到src对应的目录中，最后app的目录结构大致如下：
```
├── CMakeLists.txt
├── app.iml
├── build.gradle
├── proguard-rules.pro
└── src
    └── main
        ├── AndroidManifest.xml
        ├── cpp
        │   ├── include
        │   │   ├── libavcodec
        │   │   ├── libavfilter
        │   │   ├── libavformat
        │   │   ├── libavutil
        │   │   ├── libswresample
        │   │   └── libswscale
        │   └── native-lib.cpp
        ├── java
        ├── jniLibs
        │   └── armeabi-v7a
        │       ├── libavcodec.so
        │       ├── libavfilter.so
        │       ├── libavformat.so
        │       ├── libavutil.so
        │       ├── libswresample.so
        │       └── libswscale.so
        └── res
```

### 修改CMakeLists.txt
首先通过`include_directories`导入FFmpeg相关的头文件，然后通过`add_library()`和`set_target_properties()`方法添加需要导入的.so库，指定其具体位置。最后在`target_link_libraries()`中还要声明需要链接的库。
```shell
add_library(...)

include_directories(${CMAKE_SOURCE_DIR}/src/main/cpp/include)

add_library(avcodec SHARED IMPORTED)
set_target_properties( avcodec 
        PROPERTIES
        IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/src/main/jniLibs/armeabi-v7a/libavcodec.so)

add_library(avformat SHARED IMPORTED)
set_target_properties( avformat
        PROPERTIES
        IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/src/main/jniLibs/armeabi-v7a/libavformat.so)

add_library(avfilter SHARED IMPORTED)
set_target_properties( avfilter
        PROPERTIES
        IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/src/main/jniLibs/armeabi-v7a/libavfilter.so )

add_library(avutil SHARED IMPORTED)
set_target_properties( avutil
        PROPERTIES
        IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/src/main/jniLibs/armeabi-v7a/libavutil.so )

add_library(swresample SHARED IMPORTED)
set_target_properties( swresample
        PROPERTIES
        IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/src/main/jniLibs/armeabi-v7a/libswresample.so )

add_library(swscale SHARED IMPORTED)
set_target_properties( swscale
        PROPERTIES
        IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/src/main/jniLibs/armeabi-v7a/libswscale.so )
...

target_link_libraries( # Specifies the target library.
        native-lib

        # Links the target library to the log library
        # included in the NDK.
        ${log-lib}
        avcodec
        avformat
        avfilter
        avutil
        swresample
        swscale
        )
```

>Tips：注意下`${CMAKE_SOURCE_DIR}`代表了CMakeLists.txt当前所在的路径，不要设置错了。

最后不要忘了在`build.gradle`中声明`abiFilters`，目前我们只编译了`armeabi-v7a`架构的.so库。
```gradle
defaultConfig {
    ...
    ndk {
        abiFilters 'armeabi-v7a'
    }
}
```

### 修改native-lib.cpp，调用FFmpeg相关函数
在demo代码中，我们随便导入一个FFmpeg的函数，看看是否能正常的运行。这里直接在原基础上增加一个`avcodec_configuration()`函数调用，看看是否能打印出config信息，这里一定要记得include相应的头文件。

```c++
#include <jni.h>
#include <string>
extern "C" {
#include "libavcodec/avcodec.h"
}

extern "C" JNIEXPORT jstring JNICALL
Java_com_codezjx_ffmpegproject_MainActivity_stringFromJNI(
        JNIEnv *env,
        jobject /* this */) {
    std::string hello = avcodec_configuration();
    return env->NewStringUTF(hello.c_str());
}
```

运行项目，最终在`MainActivity`界面中应该会输出以下信息，整个FFmpeg的编译及导入流程就结束了：
```
--prefix=/Users/codezjx/AndroidProjects/ffmpeg-4.1.3/android/armv7-a 
--enable-shared --disable-static --disable-doc --disable-ffmpeg 
--disable-ffplay --disable-ffprobe --disable-avdevice --disable-symver 
--cross-prefix=/Users/codezjx/Android/android-ndk-r17c/toolchains/
arm-linux-androideabi-4.9/prebuilt/darwin-x86_64/bin/arm-linux-androideabi- 
--target-os=android --arch=arm --enable-cross-compile --sysroot=/Users/codezjx/
Android/android-ndk-r17c/platforms/android-21/arch-arm --extra-cflags='-I/Users
/codezjx/Android/android-ndk-r17c/sysroot/usr/include/arm-linux-androideabi 
-isysroot /Users/codezjx/Android/android-ndk-r17c/sysroot -D__ANDROID_API__=21 
-Os -fpic -marm -march=armv7-a'
```

[1]: https://dl.google.com/android/repository/android-ndk-r17c-darwin-x86_64.zip
[2]: https://ffmpeg.org/download.html
[3]: https://ffmpeg.org/releases/ffmpeg-4.1.3.tar.bz2
[4]: https://github.com/android-ndk/ndk/wiki/Changelog-r18
[5]: https://trac.ffmpeg.org/ticket/7130