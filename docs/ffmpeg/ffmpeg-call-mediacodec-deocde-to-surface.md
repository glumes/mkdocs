---
title: "FFmpeg 调用 MediaCodec 硬解码到 Surface 上"
date: 2021-11-14T22:38:39+08:00
subtitle: ""
tags: ["FFmpeg"]
categories: ["FFmpeg"]
toc: true
 
draft: false
original: true
slug: "ffmpeg-call-mediacodec-deocde-to-surface"
---

这是关于 FFmpeg 和 MediaCodec 爱恨情仇系列的第三篇文章了。

<!--more-->

之前写了 FFmpeg 调用 MediaCodec 进行硬解码的内容。

* [FFmpeg 调用 Android MediaCodec 进行硬解码](https://mp.weixin.qq.com/s/S8NwQnY4uyQulfZnRF7t_A)

另外也给出了 FFmpeg 的编译脚本，轻松搞定编译问题。

* [老生常谈-FFmpeg 的编译问题轻松搞定](https://mp.weixin.qq.com/s/DMY_pp7zHHT5ndznBnKLzg)

众所周知，MediaCodec 的解码能力不仅可以解码出 YUV 数据，还能直接解码到 Surface 上。

在短视频领域中，MediaCodec 解码到 Surface 上的能力反而更加常用，这样就能将画面转到 OES 纹理上，从而进行后续各种渲染操作。

之前介绍的 FFmpeg 调用 MediaCodec 进行硬解码只是解码出了 Buffer 数据，没有把解码到 Surface 上的能力用起来。

再看了更多资料之后，发现 FFmpeg 调用 MediaCodec 已经可以解码到 Surface 上。



具体参考的是这篇邮件内容：

http://mplayerhq.hu/pipermail/ffmpeg-devel/2016-March/191700.html

在这里面详细介绍了这种能力，挑重点截图一下：

![](https://image.glumes.com/blog_image/ca12f081ff938c57e18484f1411a537.jpg)

图片内容介绍的很详细，按照步骤实践就好了。

### 代码实践

> 如果熟悉了 FFmpeg 调用 MediaCodec 解码 Buffer 数据的流程，那么解码到 Surface 只是在流程上稍微改动一点就行。

首先要准备好 Surface 对象，在 Java 上层构建好 Surface 对象通过 NDK 传到 Native 层，传下来的是一个 jobject 对象。

如果不熟悉 NDK 的话，可以看看我在慕课网上的录制的免费课程：

[Android NDK 免费视频在线学习！！！](https://mp.weixin.qq.com/s/z7amits-TyT64i8bqzrjIA)

其次是两个新的函数方法：

![](https://image.glumes.com/blog_image/52a40900d753b4105a0fbd149e81c1d.jpg)

av_mediacodec_alloc_context 和 av_mediacodec_default_init 方法就是让 Surface 和 AVMediaCodecContext 、AVCodecContext 三者之间产生关联。

具体就是 AVCodecContext 持有 AVMediaCodecContext ，AVMediaCodecContext 持有 Surface 。

至于为什么要关联，因为在 FFmpeg 源码里要根据 Surface 是否为 nullptr 对 MediaCodec 的初始化和解码后的数据做不同处理。

感兴趣的可以阅读这块的源码，内容不多通俗易懂。

等到解码之后，FFmpeg 同样会返回一个 AVFrame 数据，只不过它的 data 字段不再是 Buffer 内容了。

AVFrame 的格式不再是 NV12 （解码 Buffer 数据的话就是 NV12），而是自定义的 AV_PIX_FMT_MEDIACODEC ，代表走的 Surface 模式。

```cpp
if (s->surface) {
   // surface 不为 null 的情况下：
   // 通过 mediacodec_wrap_hw_buffer 对数据进行处理
  if ((ret = mediacodec_wrap_hw_buffer(avctx, s, index, &info, frame)) < 0) {
    av_log(avctx, AV_LOG_ERROR, "Failed to wrap MediaCodec buffer\n");
       return ret;
  }
}
```

Surface 模式下对数据的处理是 mediacodec_wrap_hw_buffer 函数，Buffer 模式就是 mediacodec_wrap_sw_buffer 函数了。

同时，真正解码后的数据存储在 AVFrame->data[3] 字段上，这个字段是个老员工了。

一般解码非 Buffer 数据的情况，都会将特殊的内容保存到 data[3] 上，比如 Window 上的硬解，部分源码如下：


```cpp
static int mediacodec_wrap_hw_buffer(AVCodecContext *avctx,
                                  MediaCodecDecContext *s,
                                  ssize_t index,
                                  FFAMediaCodecBufferInfo *info,
                                  AVFrame *frame)
// 省略部分源码
// 创建   AVMediaCodecBuffer 对象
buffer = av_mallocz(sizeof(AVMediaCodecBuffer));
frame->buf[0] = av_buffer_create(NULL,
                       0,
                       mediacodec_buffer_release,
                       buffer,
                       AV_BUFFER_FLAG_READONLY);
// buffer 相关属性赋值
buffer->ctx = s;
buffer->serial = atomic_load(&s->serial);
if (s->delay_flush)
    ff_mediacodec_dec_ref(s);
// index 索引，代表 mediacodec 中 buffer 的索引
buffer->index = index;
buffer->pts = info->presentationTimeUs;
// 将 buffer 赋值给 data[3] 字段
frame->data[3] = (uint8_t *)buffer;
```

有了 AVFrame 数据之后，Surface 上还是没有画面。

回想一下，在 MediaCodec 上想要数据渲染到 Surface 还得调用一个 releaseOutputBuffer 方法，其中第二个参数要传 true 才可以。

```cpp
public void releaseOutputBuffer (int index, boolean render)
```

同样，在 FFmpeg 中也有这么个方法。

```cpp
int av_mediacodec_release_buffer(AVMediaCodecBuffer *buffer, int render);
```

buffer 就是 frame->data[3] 的内容，render 的含义和 releaseOutputBuffer 中的含义一致。

另外 releaseOutputBuffer 方法第一个参数 index 其实就已经在 buffer 中赋值过了。

这样一来，解码后就可以直接上屏渲染展示啦。

```cpp
 AVMediaCodecBuffer *buffer = (AVMediaCodecBuffer *) frame->data[3];
 av_mediacodec_release_buffer(buffer, 1);
```

完整代码实践可以在公众号 音视频开发进阶 回复 1019 获取。

经过测试验证确实可行，不过直接不断解码上屏的速度是很快的，可不止视频播放 30ms 一帧的速度哦，想要来做播放器的话，还得自己管理控制一下了。

另外，完整代码演示中直接解码到了 SurfaceView 的 Surface 上。

除此之外，还可以解码到 SurfaceTexture 构造的 Surface 上，这样就可以用到 SurfaceTexture 的 OnFrameAvailableListener 回调方法， 并且还可以用 attachToGLContext 方法关联到 OpenGL 环境上，每次解码时通过 updateTexImage 更新画面，实现解码到 OES 纹理的目标，具体操作起来也是很容易方便。


