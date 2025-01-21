---
title: "002 | 播放器系列专栏-FFmpeg依赖库的配置"
date: 2022-03-18T21:10:54+08:00
slug: "planet-player-project-configure-ffmpeg"
subtitle: ""
tags: ["Player"]
categories: ["FFmpeg"]
toc: true
 
draft: false
original: true
image: ""
author: "glumes"
---


上回书说道：[星球专享 | 关于播放器的一次项目实践](https://mp.weixin.qq.com/s?__biz=MzA4MjU1MDk3Ng==&mid=2451531599&idx=1&sn=708c717b2a74d6be5a54c150abbac4ea&scene=21#wechat_redirect)

目前已经完成了项目的创建，是怎样一个项目呢？

<!--more-->

首先是播放器 SDK ，也是项目最核心的模块，然后是对 SDK 进行单元测试的模块，最后是使用 SDK 做播放器的可视化项目模块。

项目工程的每个目录介绍已经在上篇文章中讲过了，这里会说一下如何打开项目。

![](https://image.glumes.com/blog_image20220326201628.png)

如图所示，1 和 2 代表两个 CMakeLists.txt 文件，其中 1 代表的是 SDK 工程 CMake 配置文件，2 代表的是播放器工程 CMake 配置文件。

图标 3 作为新增的库文件目录，后面会介绍。

项目根目录是作为 SDK 的目录，而 demo 是在根目录下的子目录中，同时 demo 依赖根目录 SDK 的编译结果，这种项目配置在一些开源项目中还是很常见的。

当用 CLion 打开工程时如果选择了根目录下的 CMakeLists.txt 就是 SDK 工程了，选择了 demo 目录下的就是播放器项目了，差别就是在 CLion 中能否有 PlanetPlayerDemo 这个构建，如下图所示：

![](https://image.glumes.com/blog_image20220326201651.png)

选择 SDK 工程打开方式时就只有 2 和 3 的选项了，其中 2 是 SDK 的构建，3 是单测的构建，而 1 是播放器打开方式才有的，前期很多时候都只要 SDK 打开方式就行了。

----

打开工程之后，接下来就要添加 FFmpeg 的依赖了。

这里并不打算讲要如何编译 FFmpeg ，因为一开始就被编译困住了，很难接下来的学习，反而有一种简单的方式直接拿编译好的库就行了。

如果是 Mac 电脑的话，使用 brew 安装 ffmepg ，电脑上就已经有编译好的库了，而且还很全面。

```cpp
brew install ffmpeg
```

众所周知，FFmpeg 是有很多编译选项和依赖选项的，那么上面的命令到底指定了哪些依赖呢？

![](https://image.glumes.com/blog_image20220326201710.png)

如上图，✅ 和 ❎ 表示的意思很明确了。

另外箭头所指的 url 地址其实就是 brew 安装 ffmpeg 的编译脚本了，里面指定了哪些依赖内容，比如涉及的 x264、x265 就包含在内了。

我们的播放器项目就是在 Mac 上运行的，所以完全可以直接用 brew 安装好的 ffmpeg 库。

![](https://image.glumes.com/blog_image20220326201726.png)

如上命令，在 finder 中打开 ffmpeg 的安装目录。

![](https://image.glumes.com/blog_image20220326201742.png)

其中 include 目录就是头文件目录，lib 目录里面放着 ffmpeg 的动态库和静态库。

我们要的就是这两个目录里的东西，直接拷出来用，为此我建立了一个仓库，单独存放这些编译好的库文件（只用静态库就行）。

> https://github.com/glumes/lib

这个仓库正好对应之前提的图标 3 的目录。 

温馨提示：由于我在家用的 M1 Pro 对应 arm64 架构，所以拿出来的库也是 arm64 架构的，如果你用的非 M1 对应的就是 x86_64 架构，这块等我回公司了补上，也可以自己补上。


接下来就是要在工程中链接 FFmpeg 库了。

首先新建了一个 vendor.cmake 作辅助，判断当前系统是什么平台和架构的：

```cpp
if (CMAKE_SYSTEM_PROCESSOR STREQUAL "arm64")
    set(arch arm64)
elseif (CMAKE_SYSTEM_PROCESSOR STREQUAL "x86_64")
    set(arch x64)
endif ()

if (WIN32)
    set(platform win)
elseif (APPLE)
    set(platform mac)
else ()
    message(FATAL_ERROR "not support current platform")
endif ()
```

然后添加链接 FFmpeg 库的方法：

![](https://image.glumes.com/blog_image20220326201805.png)

可以看到链接库时用到了上面指定的平台和架构信息，这和我们的目录结构是相互依赖的。

![](https://image.glumes.com/blog_image20220326201820.png)

有了这两个方法，在 SDK 工程和播放器工程都可以复用了。

接下来在 SDK 工程中的配置就和平常配置一样了，依赖好 ffmpeg 的库。

```cmake
set(path ${CMAKE_CURRENT_SOURCE_DIR})

# SDK 的头文件
set(PLANET_INCLUDES ./ include src)

include(${CMAKE_CURRENT_SOURCE_DIR}/vendor.cmake)

# 模拟第三方库依赖
add_subdirectory(3rdparty/test1)
list(APPEND PLANET_INCLUDES 3rdparty/test1/src)

add_subdirectory(3rdparty/test2)
list(APPEND PLANET_INCLUDES 3rdparty/test2/src)

# 添加 FFmpeg 头文件的依赖
list(APPEND PLANET_INCLUDES ${path}/lib/ffmpeg/${platform}/${arch}/include)
# 自定义方法 链接 ffmpeg 库目录
link_ffmpeg_directory(${path})

# SDK 的源文件
file(GLOB_RECURSE PLANET_FILES
        src/*.*)

# 编译 SDK 的静态库
add_library(PlanetPlayer STATIC ${PLANET_FILES})

# 包含头文件内容
target_include_directories(PlanetPlayer PUBLIC ${PLANET_INCLUDES})

# 链接三方库
target_link_libraries(PlanetPlayer  gtest TEST1 TEST2)

# 自定义方法 链接 ffmpeg
link_ffmpeg_library(PlanetPlayer ${path})
```


同样在播放器项目中也要做配置，依赖 SDK 以及 ffmpeg 的库。

```cmake
# 设置 SDK 的根目录
set(ProjectPath ${CMAKE_CURRENT_SOURCE_DIR}/../../PlanetPlayer)

# 自定义方法 链接 ffmpeg 库目录
link_ffmpeg_directory(${path})

# 播放器项目的头文件
set(DEMO_INCLUDES ${CMAKE_CURRENT_SOURCE_DIR}/src)

# SDK 提供的头文件
list(APPEND DEMO_INCLUDES ${ProjectPath}/include)

# 播放器项目的源文件
file(GLOB_RECURSE DEMO_SOURCE_FILES ${CMAKE_CURRENT_SOURCE_DIR}/src/*.*)

# 添加 SDK 的目录
add_subdirectory(${ProjectPath}/ PlanetPlayer)

# 播放器项目
add_executable(PlanetPlayerDemo ${DEMO_SOURCE_FILES})

# 包含头文件内容
target_include_directories(PlanetPlayerDemo PUBLIC ${DEMO_INCLUDES})

# 链接三方库
target_link_libraries(PlanetPlayerDemo PlanetPlayer)

# 自定义方法 链接 ffmpeg
link_ffmpeg_library(PlanetPlayerDemo ${path})
```

这里有个问题，就是既然 SDK 依赖了 ffmepg ，播放器依赖了 SDK ，为什么播放器还有依赖 ffmpeg ？

这是因为编译的 SDK 是个静态库，但是并没有把 ffmpeg 的静态库合并进来，导致播放器仅链接了 SDK 的库会找不到 ffmpeg 函数的符号表，后续再把这个功能补上。

另外也说明了，音视频做工程搭建也是有很多学问的。

以上就是本篇文章的内容了，搞定了库依赖就可以开始撸代码啦！！！

关于播放器实践的专栏，后续大部分进展都会放在知识星球里面了，尤其是源码会在星球内同步更新，当然也会挑一些干货在公众号同步。

目前 音视频开发进阶知识星球 还在让利中，非常低的价格就可以获得业内一线开发人员的答疑解惑。

与其在群里面提问石沉大海，不如来星球有问必答，而且这个价格还是管一年的哦，一年的时间可以说是相当划算了。

同时星球内非常欢迎大家提问，尤其是我不会的问题，我会去找业内好朋友请教，既回答了你的问题又帮助了我提高。

想要加入的可以通过扫如下二维码进星球哦，iPhone 用户如果不能访问小程序的话，也可以加我微信 ezglumes 拉你进星球。