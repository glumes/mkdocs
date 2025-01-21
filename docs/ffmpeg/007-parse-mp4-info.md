---
title: "007 | 播放器系列专栏-解析 MP4 文件读取信息"
date: 2022-06-18T13:02:00+08:00
slug: "007-parse-mp4-info"
subtitle: ""
tags: ["Player"]
categories: ["FFmpeg"]
toc: true
 
draft: false
original: true
image: ""
---

在之前文章中已经介绍过了 MP4 标准的来源以及它的格式定义，基本上就是由一个个 Box 组成的，大致的结构如下：

```cpp
ftyp
moov
    mvhd
    trak
        tkhd
        mdia
    trak
        tkhd
        mdia
mdat
```

接下来我们就要去手动解析 MP4 文件，**注意这可不是用 FFmpeg 来解封装**，而是从 MP4 文件中一个一个字节读取信息并解析它的含义获得想要的内容。

<!--more-->

平常一看到后缀是 `.mp4` 的文件，脑海里一想到的就是视频，但其实不管后缀如何，它也还是一个二进制文件，可以按照二进制的方式进行读取和写入。

## 解析 MP4 文件获取信息

举个例子，在 Mac 上用 `010 Editor` 软件去查看一个 MP4 文件，以 16 进制显示，效果如下：

![](https://image.glumes.com/blog_image/mp4-010-editor111.png)

从图中箭头指示处可以看到 `ftyp` 和 `mvhd` 两个 Box 类型，另外也还有 `moov` 和 `tkhd` 这些 Box 类型。

之所以能够显示出 Box 类型的字符串，是因为把十六进制数据转换成 ASCII 码了，比如 61 对应就是字母 a ，这应该在计算机基础书中都有讲过的。

在 `mvhd` Box中存储着视频文件的时长信息，想要获取到这个信息，直接从 Box 中读取就好，至于为什么会这样，见下图：

![](https://image.glumes.com/blog_image/20220612231019.png)

上图展示了 mvhd Box 的数据结构，它继承自 FullBox，在读取时先读取 FullBox 的字段，然后在读取 mvhd Box 自己的（取 version == 0 时的数据结构排布）。

![](https://image.glumes.com/blog_image/20220613191456.png)

> 关于 mvhd box 和 full box 的数据结构文档以及 MP4 中所有 Box 的类型资料，已经在知识星球中给出了，可以加入星球在资料中找到。

FullBox 的数据结构如上所示，由字节位数可以算出，在 box type 之后偏移 12 字节可以得到 timescale 字段内容，偏移 16 字节可以得到 duration 字段内容。

其中 timescale 为 0x000003E8，对应十进制数据 1000 。

duration 为 0x000086E6，对应十进制数据 34534。

用 duration 除以 timescale 就是视频的时长了，十六进制相除后的结果是 0x22，转换成十进制就是 34，和用十进制数据相除得到的 34.534 基本一致了（时长单位是秒）。

以上只是个简单例子，说明完全可以去手动解析 MP4 文件获取它的格式信息。

而且在这个层面上还有一些独特的用法：比如我们要想提取视频中的某个 Box 信息，或者想要填充自定义的 Box 格式，携带一些私有数据，在播放时再把它解析出来做处理。


在接下来的文章，我们就会去实践手动解析 MP4 文件，逐一拆解每个 Box 格式，发掘其背后的另一种用法，加强对音视频的处理能力。
