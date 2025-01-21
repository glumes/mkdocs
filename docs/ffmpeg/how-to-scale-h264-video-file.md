---
title: "H264视频文件如何缩放分辨率"
date: 2022-03-21T09:50:34+08:00
slug: "how-to-scale-h264-video-file"
subtitle: ""
categories: ["FFmpeg"]
tags: ["FFmpeg"]
toc: true
 
draft: false
original: true
image: ""
author: "glumes"
---

前几天在知识星球里面有位朋友请教问题：如何将 H264 视频缩放分辨率？

<!--more-->

具体的问题详情如下：

![](https://image.glumes.com/blog_image20220326201042.png)

将 800x600 的 H264 文件缩放成 400x300 的，大概的流程是先解码，得到 AVFrame 后对其做缩放操作，然后再编码，得到 AVPacket 后写入文件即可。

思路没错的话，就可以看看这位同学的写法了，截取部分代码：

![](https://image.glumes.com/blog_image20220326201112.png)

> 大家一起来挑错，看看遇到问题一般解决思路是什么样的。

## 问题一

首先提问者一直在 av_parser_parse2 函数处报错。

遇到这种情况不要慌，必现的问题先断点一下看看输入参数是否正确。

> IDE 的断点调试是常见操作了，工程师进阶必备。


我没有调试提问者的代码，但也怀疑输入参数 pkt 有异常，顺着代码往下看，果然在函数最末尾几行调用了 av_packet_free 方法。

一般通过 av_packet_alloc 去申请 AVPacket ，而 av_packet_free 就是直接释放并置 nullptr ，这样下次在执行 av_parser_parse2 方法肯定就挂了。

```cpp
void av_packet_free(AVPacket **pkt)
{
    if (!pkt || !*pkt)
        return;

    av_packet_unref(*pkt);
    av_freep(pkt);
}
```

所以这里肯定有问题呀。

## 问题二

接着看其他问题，想要缩放分辨率，可以代码截图中并没有看到任何缩放的代码，直接将解码后的 AVFrame 送去编码就可以缩放吗？

我猜想，提问者应该在设置编码的 AVCodecContext 时就已经指定好了缩放后的分辨率 400x300 ，但送去编码的 AVFrame 还是 800x600 的，这样编码的结果会是缩放的吗？

经过试验证明，编码的视频确实是 400x300 的，但画面却是从 800x600 截取的一部分，并没有显示完全，所以这样是不能起到缩放效果的。

要缩放还是得用 FFmpeg 中 SwsContext 提供的方法。

```cpp
// 初始化
swsCtx = sws_getContext(800,600,AV_PIX_FMT_YUV420P,400,300,AV_PIX_FMT_YUV420P,SWS_BILINEAR, nullptr, nullptr,nullptr);
// 缩放
sws_scale(swsCtx,decodeFrame->data,decodeFrame->linesize,0,decodeFrame->height,encodeFrame->data,encodeFrame->linesize);
```

将缩放后的 AVFrame 送去编码就可以得到正确的效果了。


## 问题三

再仔细看提问者的代码，有必要在解码 avcodec_receive_frame 之前调用 av_frame_make_writable 吗？

由于提问者的代码本身不对，其实也不用调用 av_frame_make_writable 的，正常的缩放应该要两个 AVFrame 的，解码的 AVFrame 不需要，反而编码的 AVFrame 需要保证可写。


以上就是关于这次提问的一些问题反馈了，我自己也实现了一个简单的 H264 视频文件缩放分辨率的例子，**完整的代码就放在知识星球里**。