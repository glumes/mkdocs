---
title: "Windows 下 FFmpeg 和 LibX264 的编译和配置"
date: 2021-12-19T17:37:37+08:00
slug: "windows-compile-ffmpeg-with-libx264"
subtitle: ""
categories: ["FFmpeg"]
tags: ["FFmpeg"]
toc: true
 
draft: false
original: true
toc: true
author: "glumes"
---


周末在家折腾 Windows 平台下 FFmepg 和 LibX264 库的编译，长期以来都是在 Mac 平台下做开发，切换到 Windows 平台下还是踩了不少坑。

<!--more-->

参考了网上很多编译文章，质量也是参差不齐，版本也是五花八门，但归根到底还是 Window 下编译环境太坑爹了。

由于 Windows 上的命令行工具不好用，所以需要安装 MSYS 或者 Cygwin 这样的软件，它们的作用就是模拟 Linux 环境，其中 MSYS 还分 1.0 和 2.0 版本，有的博客文章比较久远，还使用的 1.0 版本了。

机智地没有选择走 Cygwin 这条路线，节省了不少时间，但还是踩了 MSYS 1.0 版本的坑。

如果你看到的文章是安装  MSYS 1.0 版本，并且还需要额外安装 MinGW 软件，那么请退出来，重新找个 MSYS 2.0 版本的文章吧，这样还能绕过 MinGW 单独下载太慢的问题（别问为什么我知道，你懂的）。

使用 MSYS 2.0 版本，就不需要额外安装 MinGW 软件了，它提供了 **pacman** 软件管理器，通过它来安装依赖的软件。

官网地址：[https://www.msys2.org/](https://www.msys2.org/)

> MSYS 2.0 安装软件的时候，如果网速很慢，可以考虑更新镜像源，使用国内的源。

搞定软件之后，先编译 libx264 ，在编译 FFmpeg 。

## MinGW 和 MSVC 的作用

在实际编译的时候，我们也是用不上 MinGW 的，看了一些文章用 MinGW 来编译，最后编译出来的静态库是个 .a 的形式。

一开始还没反应过来，Windows 下的静态库不是 .lib 嘛，直接用 CMake 去链接 .a 库肯定不行啊。

还看到一些文章说先把 .a 库转成 .def 文件，然后再把 .def 文件转成 .lib 文件，甚至再把 .lib 文件转成 .dll 的动态库，这么来回折腾一下又是大坑，还好没跳进去。

转念一想，我要用 CLion 开发工程，编辑器直接用 MSVC 就好了，也用不上 gcc 来编译代码，干嘛用 MinGW 去编译个 .a 库呢，直接编译出 .lib 不好嘛。

瞬间思路就打开了，调整方向，谷歌直接搜索 compile ffmpeg with msvc ，很快就找到了答案（谷歌搜英文会过滤掉很多网上各种抄袭复制的无效文章）。

## LibX264 编译

首先下载好 LibX264 源码。

然后在开始菜单中找到并打开 x64 Native Tools Command Prompt for VS 2019 ：

![](https://image.glumes.com/blog_image/vs2019-tools.png)

在打开的命令行终端中，进入到 MSYS 安装目录，打开 msys2_shell.cmd ，如下命令：

![](https://image.glumes.com/blog_image/open-msys2.png)

注意后缀有个 -use-full-path 。

这时会打开 MSYS 的新窗口，先把一些汇编依赖安装好：

```cpp
pacman -Syu
pacman -S make
pacman -S diffutils
pacman -S yasm
pacman -S nasm
```

然后，在该窗口中进入到 LibX264 的源码目录下，把如下代码保存成 .sh 文件并执行：

```cpp
OPTIONS="--enable-shared"

CC=cl ./configure $OPTIONS --enable-shared --prefix=$BUILD_DIR/

make -j 16
make install
make clean
```

执行后就开始编译了，注意 configure 命令前缀有个 CC=cl ，代表使用 MSVC 来编译了。

编译后内容如下：

![](https://image.glumes.com/blog_image/libx264_compile.png)

将编译后的 libx264.dll.lib 改成 libx264.lib，这就是静态库了。

## FFmpeg 编译

继续在 MSYS 2.0 窗口中进入到下载好 FFmpeg 的源码目录，将如下代码保存成 .sh 文件并执行：

```cpp
OPTIONS="--toolchain=msvc \
         --arch=x86_64 \
         --enable-yasm  \
         --enable-asm \
         --enable-shared \
         --disable-static \
         --disable-programs \
         --enable-swresample \
         --enable-swscale \
         --enable-libx264 \
         --enable-gpl \
         "
X264_INCLUDE=$libx264_path/include
X264_LIB=$libx264_path/lib

CC=cl ./configure $OPTIONS --extra-cflags="-I$X264_INCLUDE" --extra-ldflags="-LIBPATH:$X264_LIB" --prefix=$BUILD_DIR/
make -j 16
make install
make clean
```

要将代码中的 libx264_path 路径改成上面编译的 libx264 路径，FFmpeg 的编译需要依赖 libx264 的库。

一番等待后，就编译出了动态库：

![](https://image.glumes.com/blog_image/ffmpeg_share_library.png)


## CMake 依赖 FFmpeg 和 LibX264

最后就是在 Clion 中使用 CMake 去依赖 FFmpeg 和 LibX264 了。

定义了两个宏函数去链接头文件和库的目录：

```cmake
macro(link_ffmpeg)
    include_directories(${ffmpeg}/${platform}/${arch}/include)
    link_directories(${ffmpeg}/${platform}/${arch}/bin)
endmacro()

macro(link_libx264)
    include_directories(${libx264}/${platform}/${arch}/include)
    link_directories(${libx264}/${platform}/${arch}/lib)
endmacro()
```

> 注意，FFmpeg 链接库用的是 bin 目录下的，libx264 用的是 lib 目录下的。

在最后这一步反而卡主了：

```cmake
target_link_libraries(demo libx264 avcodec avformat)
```

要么提示找不到 libx264，要么找不到 avcodec-59，这个时候还需**把 ffmpeg 编译结果的 bin 目录添加到系统环境变量中**，为了保险起见，把 libx264 的 bin 目录也添加了。


加完之后，跑一段代码测试一下：

```cpp
#include <iostream>

extern "C"{
#include "libavformat/avformat.h"
#include "libavcodec/avcodec.h"
#include "x264.h"
}

int main() {
    x264_param_t param;
    x264_param_default(&param);
    auto codec = avcodec_find_encoder(AV_CODEC_ID_H264);
    if (codec){
        std::cout << "success!!!" << std::endl;
    }
    return 0;
}
```

果然就成功了，这下可以在 Windows 上开发学习 FFmpeg 了。

## 参考

1. https://www.cnblogs.com/wswind/p/10650126.html
2. https://blog.csdn.net/qq_18453581/article/details/120005712
2. https://www.roxlu.com/2016/057/compiling-x264-on-windows-with-msvc
3. https://www.roxlu.com/2019/062/compiling-ffmpeg-with-x264-on-windows-10-using-msvc

