---
title: "003 | 播放器系列专栏-认识MP4视频（上）"
date: 2022-04-10T22:09:22+08:00
slug: "what-is-mp4-1"
subtitle: ""
tags: ["Player"]
categories: ["FFmpeg"]
toc: true
 
draft: false
original: true
image: ""
---


在开始播放器实践之前，我们要先知道播放的内容是什么：认识一下 MP4 视频。

<!--more-->

根据维基百科的介绍，MP4 或称 MPEG-4 第14部分（英语：MPEG-4 Part 14）是一种标准的数字多媒体容器格式。

MPEG-4 第14部分的扩展名为.mp4，以存储数字音频及数字视频为主，但也可以存储字幕和静止图像。


字面意思很容易理解，MP4 其实是一种容器，可以存音频和视频内容。那么问题来了，既然说 MP4 是 MPEG-4 第14部分，那其他部分是什么呢？有没有 MPEG 的 1、2、3 甚至 5、6、7 呢？

## 这里就涉及到 MP4 标准的制定了。

首先需要知道国际上有一个组织叫做 MEPG (Moving Picture Experts Group，动态图像专家组)。

它是 ISO（International Standardization Organization，国际标准化组织）与IEC（International Electrotechnical Commission，国际电工委员会）于1988年成立的专门针对运动图像和语音压缩制定国际标准的组织。 

这里组织名字比较多，就不赘述了，可以直接去百度一下。

大意就是两个大组织 ISO 和 IEC 成立了一个小组织 MEPG 来制定运动图像和语音压缩的标准，其实就是制定视频和音频方面的标准，可能那个年代把视频叫做运动图像吧。

而 MEPG 组织的产出就是一系列标准，并且命名也很简单，就是 MPEG-1 标准、MPEG-2 标准，以此类推。

不过，要注意 MPEG 标准后面的数字可不是依次递增的哦，比如 MPEG-3、MPEG-5、MPEG-6 就不存在的，就好比 Windows 电脑直接从 Win8 跳到 Win10 ，也没有 Win9 了，这也回答了上面的问题，并不是每个数字代表的标准都有的。


## 另外，为什么说 MP4 是 MPEG-4 的第14部分呢？


因为 MPEG-4 标准很大，包括了 27 个部分，详细的 27 个部分内容可以在网络上搜索到，贴个图：

![](https://image.glumes.com/blog_image20220409224901.png)

比如常见的 H.264 就是 MPEG-4 第 10 部分，所以介绍 H.264 的时候也可以说是 MPEG-4 第 10 部分。

这里面的每个部分多多少少都影响着我们的生活了，对于开发人员来说，还是需要了解一些关键部分的内容。

我已经把 MP4 相关的 MPEG-4 第14 部分文档下载好了，文件会同步到知识星球里面，有需要的可以自行下载，截个图如下：

![](https://image.glumes.com/blog_image20220409230040.png)


文档内容不多，就十几页，主要就是讲了 MP4 文件格式的定义和相关语法。

![](https://image.glumes.com/blog_image20220409230241.png)

后面我们也会去解读一下 MP4 的文件格式，这次就先到这里，下次见了！！！