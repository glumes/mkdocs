---
title: "005 | 播放器系列专栏-在 Windows 上查看 MP4 格式信息"
date: 2022-04-16T18:29:38+08:00
slug: "windows-mp4-info"
subtitle: ""
tags: ["Player"]
categories: ["FFmpeg"]
toc: true
 
draft: false
original: true
image: ""
---

在之前的文章中我们已经认识了 MP4 视频，知道了它是音频和视频的容器，并且由一系列 Box 组成。

在前文的附件中，我们也给出了对应的资料，包括 MP4 格式的官方定义以及各种 Box 类型的描述。

但是纸上得来终觉浅，绝知此事要躬行，光是理论上知道了还不行，需要亲自实践加深印象。

这次会在 Windows 平台上用工具解析查看 MP4 格式信息，推荐的工具就是 **Mp4 Explorer**。

<!--more-->

**Mp4 Explorer** 的下载地址如下，目前只有 Windows 平台提供下载。

> https://mp4-explorer.apponic.com/

![](https://image.glumes.com/blog_image20220416162252.png)

---

**Mp4 Explorer** 安装后打开的界面也很简单，没啥花哨功能，就单纯地解析 MP4 。


![](https://image.glumes.com/blog_image20220416175408.png)


选取视频，直接打开，就可以看到如下界面，左侧就是 Box 类型，右侧就是对应信息。


![](https://image.glumes.com/blog_image20220416175434.png)

左侧 Box 的层级结构就和之前文章描述一样，Box 是由 Header 和 Data 两部分组成，而 Data 可以是单纯的数据也可以是其他的子 Box ，此时这种 Box 又叫做 Container Box 。

如图所示，moov、track、edts、mdia 都是 Container Box 。

左侧的整体结构和 MPEG-4 第 12 部分文档描述的大致相同：


![](https://image.glumes.com/blog_image20220410192809.png)


一个 MP4 Box 层级大致的结构都是这样：

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

当然，不是所有的 Box 类型都会出现在同一个 MP4 视频上的，有些不重要的 Box 类型可以省略，我们只要知道那些常见的 Box 就好了。

另外，工具解析也会存在一些误差，有时候发现 MP4 Box 类型或者数据对不上了，也不能全单靠一个工具，可以多个工具验证 MP4 信息或者手动解析。

除了 **Mp4 Explorer** 之外，也还有其他工具可以查看解析 MP4 Box 数据，这里就不做演示了，主要是让大家对 Box 有一个更直观的感受。

关于每个 Box 的作用是什么，如何去解析，我们后面继续讲解，到时候还会再来使用这个软件的。
