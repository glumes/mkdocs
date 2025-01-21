---
title: "006 | 播放器系列专栏-在 Mac 上查看 MP4 格式信息"
date: 2022-05-05T21:39:07+08:00
slug: "mac-mp4-info"
subtitle: ""
tags: ["Player"]
categories: ["FFmpeg"]
toc: true
 
draft: false
original: true
image: ""
---

之前介绍了在 Windows 上查看 MP4 格式信息，使用的是 **Mp4 Explorer** 软件，具体使用如下：

[005 | 播放器系列专栏-在 Windows 上查看 MP4 格式](https://glumes.com/post/ffmpeg/windows-mp4-info/)

现在该介绍一下 MAC 上用的软件了，它就是 MediaInfo 软件，官网地址如下：

<!--more-->

https://mediaarea.net/MediaInfo

使用起来也很简单，打开视频文件就可以看到信息了。

如下图：

![](https://image.glumes.com/blog_image20220501215721.png)

现在 MediaInfo 也要收费使用了，而且还不能显示 Box 信息，其内容和 FFmpeg 命令显示差不多的，所以在 Mac 上还是建议直接用 FFmpeg 好。


## Bento4 介绍

那么如何在 Mac 上显示 MP4 的 Box 信息呢？答案就是用 **Bento4** 软件。

Bento4 的官网地址如下：

https://www.bento4.com/

它的 Github 地址如下：

https://github.com/axiomatic-systems/Bento4

Bento4 是一款开源软件，有 C++、Java、Python 三种对外接口，可以用来对 MP4 视频进行各种操作。

比如显示 Box 结构、加解密视频、抽取音视频流等，还可以将 MP4 文件转换成 TS 文件或者 HLS 文件，或者将 MP4 转换成 Fragment MP4 格式。

详细功能在 Github 中介绍的很全面了，如下图：

![](https://image.glumes.com/blog_image20220501222822.png)

中文翻译版本：

```cpp
mp4info         显示有关MP4文件的高级信息，包括所有tracks和codec的详细信息。
mp4dump         显示MP4文件的整个atom/box结构。
mp4edit         添加/插入/移除/替换 MP4文件的atom/box项。
mp4extract      从MP4文件中提取atom/box
mp4encrypt      加密MP4文件（支持多种加密方案）
mp4decrypt      解密MP4文件（支持多种加密方案）
mp4dcfpackager  将媒体文件加密为OMA DCF文件
mp4compact      将“stsz”表转换为“stz2”表以创建更紧凑的MP4文件
mp4fragment     从非碎片化的MP4文件创建碎片化MP4文件。
mp4split        将支离破碎的MP4文件拆分为离散的文件
mp4tag          显示/编辑MP4元数据（iTunes样式和其他样式）
mp4mux          将一个或多个基本流（H264/AVC、H265/HEVC、AAC）多路复用到MP4文件中
mp42aac         从MP4文件中提取原始AAC基本流
mp42avc         从MP4文件中提取原始AVC/H.264基本流
mp42hls         将MP4文件转换为HLS（HTTP Live Streaming）演示文稿，包括生成片段和.m3u8播放列表以及AES-128和SAMPLE-AES（用于公平播放）加密。这可以作为苹果mediafilesegmenter工具的替代品。
mp42ts          将MP4文件转换为MPEG2-TS文件。
mp4dash         从一个或多个MP4文件（包括加密）创建MPEG破折号输出。作为一种选择，还可以同时生成带有MP4片段的HLS播放列表，允许将单个流用作DASH和HLS。这是一个功能齐全的MPEG DASH/HLS打包机。
mp4dashclone    创建远程或本地MPEG破折号演示文稿的本地克隆，并在克隆片段时对其进行加密。
mp4hls          从一个或多个MP4文件创建多比特率HLS主播放列表，包括对加密和仅I帧播放列表的支持。此工具在内部使用“mp42hls”低级工具，因此该低级工具支持的所有选项也都可用。这可以作为苹果variantplaylistcreator工具的替代品。
```

## 安装 Bento4

安装很简单，一行命令搞定：

```cpp
brew install bento4
```

然后就拥有了上面图片中的所有命令行工具。

## Bento4 工具使用

别看上面命令行工具很多，使用起来非常简单。

![](https://image.glumes.com/blog_image20220501223716.png)

直接输入命令，就可以看到使用提示。

想要查看 MP4 的 Box 信息，使用如下命令：

```cpp
// 换成你的路径
mp4dump video-640x360.mp4
```

显示如下：

![](https://image.glumes.com/blog_image20220501224049.png)


除了显示 Box 信息，还有一些其他的操作，比如将 MP4 转成 TS 格式文件

```cpp
mp42ts video-640x360.mp4 ideo-640x360.ts
```

另外还可以将 MP4 文件进行切片支持 HLS 流格式，从而支持 m3u8 的形式进行播放。

```cpp
mp42hls video-640x360.mp4
```

生成的文件如下：

![](https://image.glumes.com/blog_image20220501225440.png)

总之，Bento4 的功能还是很强大的，非常值得学习和使用。