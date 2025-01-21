---
title: "【音视频连载-001】基础学习篇- SDL 介绍以及工程配置"
date: 2020-02-26T23:33:35+08:00
subtitle: ""
tags: ["SDL"]
categories: ["FFmpeg"]
slug: "av-beginner-001"
toc: true
 
draft: false
original: true
---

这是音视频基础学习系列的第一篇文章，主要讲解 SDL 是什么以及为什么要用到它，看似和音视频没啥卵关系，其实必不可少。

<!--more-->

### SDL 简介

SDL 是 “Simple DirectMedia Layer” 的缩写，它是一个跨平台的多媒体库，可以在 Mac、Windows、Linux 以及更多的系统上运行。

SDL 提供了统一的针对音频、视频、键盘、鼠标、控制杆以及 3D 硬件的低级别访问接口，我们利用这些接口就能在不同系统上播放出音频、视频内容，而无需懂得系统特定的音视频接口。

这种跨平台特性和 OpenGL 是一样的，差别在与 OpenGL 是真·跨平台，它是 Khronos Group 组织开发维护的一个接口规范，具体的实现是由驱动厂商完成。而 SDL 是把要兼容平台的相关接口都给封装好了，然后才对外提供统一的接口。

由此可见，高下立判。一个跨平台是写了接口就行，不管实现；另一个是写好了实现，才能跨平台（貌似跑题了）。

由于 SDL 的跨平台特性，在后续学习 FFmpeg 时就可以利用 SDL 进行音视频的播放操作，而不用像在 Android 平台上搞 FFmpeg 还得编译 so、写 JNI 、写界面那么麻烦，另外 ffplay 源码里面也是用的 SDL 进行播放的，可以从中进行借鉴。




### SDL 下载安装

在 Mac 上下载 SDL 很简单，直接

```sh
brew install sdl2
```

注意，这里下载的是 SDL 2.0 版本，如果用如下的命令

```sh
brew install sdl
```

下载的就是 SDL 1.0 版本了，区别就是版本后面数字 `2` 的后缀。

采用最新的 2.0 ，我当前使用的版本号就是 **2.0.10** 。

如果是 Windows 系统，参考下其他文章的下载配置吧，没有电脑也没办法了。

SDL 下载之后位于 MAC 系统的如下目录，这个目录后续会用到的。

> /usr/local/Cellar/sdl2/2.0.10

### CLion 新建工程

接下来就开始打开 CLion ，新建一个 C++ 工程。

这里用到 CLion 是因为它确实好用，自动补全、代码提示、断点调试等功能非常好用，只是没有社区免费版的，有 30 天的免费试用期，之后就得靠激活码激活了。

好在是用 CMake 进行编译的，如果下载了工程源码，并且配置好了 CMake 的关联库和头文件，直接用 CMake 命令行也可以进行编译的，这个后面会讲到。

### C++ 工程关联 SDL 库

接下来就是在 C++ 工程中关联 SDL 库，便于在工程中引用 SDL 相关头文件。

之前提到 SDL 的安装路径如下：

```sh
/usr/local/Cellar/sdl2/2.0.10
```

该目录如下图：

![](https://images.xiaozhuanlan.com/photo/2020/650c10feacc111f9a09d873711bdf0cb.png)

其中 include 就是头文件的路径，lib 就是库的路径。

这里要用到 `include_directories` 和 `link_directories` 两个命令。

其中：

> `include_directories` 是将头文件所在文件夹添加在搜索路径中，这样就能通过 `include` 去加载头文件了。


> `link_directories` 是将库所在的文件夹添加在路径中去，这样在编译时就能链接到这个库。

具体代码如下：

```cpp
# 声明一个变量 SDL_DIR 为 SDL 安装路径
set(SDL_DIR "/usr/local/Cellar/sdl2/2.0.10")
# 设置要包含的头文件的路径
include_directories(${SDL_DIR}/include)
# 设置要关联的库的路径
link_directories(${SDL_DIR}/lib)
```

代码中声明了一个变量 `SDL_DIR` 作为安装路径，如果你的系统上路径有所不同，只需要修改路径就行了。

> 在 MAC 上也可以把路径设置成 `/usr/local`，所有的库安装时在这个目录的 `lib` 和 `include` 目录下也有一份索引。


最后将我们要编译的程序关联上 SDL 这个库。

你可以通过 `link_directories` 命令将很多库所在文件夹都添加到路径中，但是只有 `target_link_libraries` 命令才会最终决定关联什么库，如果你添加的文件夹路径没有对应库的话，反而要报错的。


实现代码如下：

```cpp
target_link_libraries(av-beginner SDL2)
```

`target_link_libraries` 方法会优先链接动态库，也可以显示指定动态库或者静态库。MAC 上动态库的后缀是 `dylib` 。在上面的图片可以看到 `libSDL2.dylib` 其实是一个索引，真正的库是 `libSDL2-2.0.0.dylib`，索引忽略了它的版本号。

完成了 SDL 库的关联，就可以开始真正编写代码了。


### 代码实践

代码实践主要是验证我们的环境配置有没有问题，运行一个 SDL 函数来试试。

```cpp
#include <iostream>
#include <SDL2/SDL.h>
using namespace std;

int main(){
    cout << "hello av-beginner" << endl;
    SDL_Init(SDL_INIT_EVERYTHING);
    return 0;
}
```

`SDL_Init` 是 SDL 的初始化函数，可以根据所需功能选择性的初始化也可以全部初始化。

如果程序正常输出并且正常退出，那么说明环境配置 OK 了，后面就可以进行功能开发了。

### 总结

以上就是音视频基础学习连载的 `001` 篇。

具体代码见仓库：

> https://github.com/glumes/av-beginner

本篇文章对应的提交 `tag` 为 `av-beginner-001`，可切换至对应源码查看。

