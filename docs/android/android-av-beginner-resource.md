---
title: "推荐几个堪称教科书级别的 Android 音视频入门项目"
date: 2020-05-06T23:10:48+08:00
subtitle: ""
tags: []
slug: "android-av-beginner-resource"
categories: ["Android"]
toc: true
 
draft: false
original: true
author: "glumes"
---

> 避免图片丢失，建议阅读微信原文：
> 
> https://mp.weixin.qq.com/s/Bb0fgFMgH5QOBqYhOj0uqA


在 **[直播](https://mp.weixin.qq.com/s/HlbGGOyfixc8wgoDjTY0Qw)** 中有提到几个不错的开源项目，这里再重点推荐一下：

目前，市面上关于音视频学习的相关书籍并不多，而且即使看了书籍学了理论，最终还是要回归到代码上来。

毕竟 IT 行业实践性要求高，强调动手能力，音视频这方面就更得多操作和探索了。

推荐下面几个项目会各有侧重，分别涵盖了 Android 音视频录制 API 、OpenGL 渲染和综合运用的例子。

<!--more-->

* GPUImage

Github 地址：

> https://github.com/cats-oss/android-gpuimage

GPUImage 毫无疑问是音视频项目里面必读工程了，它的侧重点在于渲染方面。

有些公司的招聘要求上可能都会写明熟悉GPUImage ，重要性可见一斑。

通过阅读 GPUImage 的源码，能够让你掌握 OpenGL 的渲染以及渲染链的搭建，同时工程里面很多特效 Shader 代码，通过阅读和实践这些 Shader 代码，能够让你掌握初步的 Shader 编写能力。

比如常见的滤镜效果，在 GPUImage 就有现成的代码例子，这一点在我的直播中也有讲到。有兴趣的可以翻阅视频，掌握常见滤镜效果的代码编写。

如果需要 GPUImage 相关的源码分析文档，也可以参考我之前写过的一篇文章： 

[OpenGL 之 GPUImage 源码分析](https://mp.weixin.qq.com/s/Xc0r6PsxrT-dJ_W4-K7V0Q)

* AudioVideoRecordingSample

Github 地址

> https://github.com/saki4510t/AudioVideoRecordingSample

此项目的侧重点在于 Android 音视频相关 API 的使用，尤其是在 录制和编码方面的。

该项目运行后能够将 Camera 采集的视频和音频内容编码成一个 MP4 文件。

这其中用到了 MediaCodec 做编码，用到了 MediaMuxer 将音频和视频混合。

这样的一个完整示例对于掌握 Android 上音视频相关 API 帮忙非常大，因为它能够成功正确运行，而且可以通过去修改其源码来做自己的实验，验证自己对于 API 的理解和掌握。

当你能够熟练掌握其内容，或者你就可以试着更进一步，尝试用 FFmpeg 做音视频的编码和混合，实现和 Android 音视频 API 一样的功能。

* Grafika

Github 地址

> https://github.com/google/grafika

此项目是 Google 提供的一个非官方的项目，它的侧重点在于将 OpenGL 与 Android 音视频 API 综合运用。

它包含了很多个完整小示例，比如如何使用 TextureView 显示 OpenGL 内容、使用三种方式进行 OpenGL 内容的录制、如何进行硬编码操作等。

通过阅读这些例子，能够让你掌握更多的技巧，把前面学会的 OpenGL 和 Android 音视频 API 更灵活运用了，进一步加深理解。

甚至有些例子都可以用到项目早期需求中去的，比如如何进行 EGL 的封装、渲染线程与主线程的分离。

另外，以上三个例子都会包含 Camera 相关的操作，比如如何将 Camera 内容展示到 SurfaceView 、TextureView 上，如何进行 Camera 拍摄等。

## 最后

之前这三个项目堪称教科学书级别的，不是没有理由的。至少我都源码阅读了两边以上。

第一次阅读的时候会觉得 " 嗯，明白怎么回事了 "，等到项目实践了，需要自己从头搞一遍，这时再回头看，会有新的感悟 "哦，原来要这样设计呀" ，等到更熟练的时候，在来看，可能就会觉得 "咦，这块能优化一波了"。

以上，希望对于想从事音视频开发的你，也能够看看上面几个项目源码，学习到更多技巧，共同进步。

