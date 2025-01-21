---
title: "FFmpeg 调用 Android MediaCodec 进行硬解码（附源码）"
date: 2021-10-19T10:18:38+08:00
subtitle: ""
tags: ["FFmpeg"]
categories: ["FFmpeg"]
toc: true
 
draft: false
original: true
slug: "ffmpeg-call-android-mediacodec"
---

> 文章原创首发公众号：音视频开发进阶。链接地址：https://mp.weixin.qq.com/s/S8NwQnY4uyQulfZnRF7t_A

FFmpeg 在 3.1 版本之后支持调用平台硬件进行解码，也就是说可以通过 FFmpeg 的 C 代码去调用 Android 上的 MediaCodec 了。

<!--more-->

在官网上有对应说明，地址如下：

https://trac.ffmpeg.org/wiki/HWAccelIntro

![](https://image.glumes.com/blog_image20211019101917.png)

从图中可以看到，不仅仅是 Android 上支持 MediaCodec，iOS 上也支持 VideoToolbox，连 Windows 上的 Direct3D 11 都有支持了。



> 注意：Android MediaCodec 目前仅支持解码，还不支持编码呢。

不过，为了验证是否可行，做个简单的演示，最后会有完整的的代码给出。

首先是 FFmpeg 的编译。它的编译有很多开关选项，要确保打开了 mediacodec 相关的选项，具体如下：

```cpp
--enable-mediacodec
--enable-decoder=h264_mediacodec
--enable-decoder=hevc_mediacodec
--enable-decoder=mpeg4_mediacodec
--enable-hwaccel=h264_mediacodec
```

可以看出 mediacodec 支持的编码格式有 h264、hevc、mpeg4 三种可选，不在范围内的就还是考虑软解吧。

**关于如何编译，就不详细阐述了，后面再专门写一篇来介绍**。

编译出对应的 so 之后，可以打印一下 AVCodec 支持的格式列表，看看有没有 mediacodec 。

具体代码如下：

```cpp
char info[40000] = {0};

AVCodec *c_temp = av_codec_next(NULL);
while (c_temp != NULL) {
    if (c_temp->decode != NULL) {
        sprintf(info, "%s[Dec]", info);
    } else {
        sprintf(info, "%s[Enc]", info);
    }

    switch (c_temp->type) {
        case AVMEDIA_TYPE_VIDEO:
            sprintf(info, "%s[Video]", info);
            break;
        case AVMEDIA_TYPE_AUDIO:
            sprintf(info, "%s[Audio]", info);
            break;
        default:
            sprintf(info, "%s[Other]", info);
            break;
    }
    sprintf(info, "%s %10s\n", info, c_temp->name);
    c_temp = c_temp->next;
}
```

通过 AVCodec 的 next 指针进行遍历，然后打印出结果，看到下面的内容说明编译成功了。

![](https://image.glumes.com/blog_image20211019101935.png)

支持的格式里面已经有了 h264_mediacodec 和 mpeg4_mediacodec 了。

---

接下来就进行解码了。关于 FFmpeg 解码的 API 调用，在公众号以前发布的文章中说过多次，就不详细讲解流程了，简单概况一下：


1. 首先通过 avformat_open_input 方法打开文件，得到 AVFormatContext 。
2. 然后通过 avformat_find_stream_info 查找文件的视频流信息。
3. 得到文件相关信息和视频流信息，主要还是为了得到编码格式信息，然后好找到对应的解码器。也可以通过 avcodec_find_decoder_by_name 方法直接找具体的解码器。
4. 有了解码器就可以创建解码上下文 AVCodecContext，并通过 avcodec_open2 方法打开解码器
5. 然后通过 av_read_frame 读取文件的内容好进行下一步的解码。
6. 接下来就是熟悉的 avcodec_send_packet 发送给解码器，avcodec_receive_frame 从解码器取回解码后的数据。



重点讲解一下调用硬件解码和普通解码的一些区别：


第一步是要在 so 加载的 JNI_OnLoad 方法中将 JavaVM 设置给 FFmpeg 。

```cpp
jint JNI_OnLoad(JavaVM *vm, void *res) {
    av_jni_set_java_vm(vm, 0);
    return JNI_VERSION_1_4;
}
```

缺少这一步就不能反射调用 Java 方法了。

---

接下来还是判断硬件解码类型支不支持，上面是通过 AVCodec 来判断的，实际上 FFmpeg 都给出了硬件类型的定义，在 AVHWDeviceType 枚举变量中。

```cpp
enum AVHWDeviceType {
    AV_HWDEVICE_TYPE_NONE,
    AV_HWDEVICE_TYPE_VDPAU,
    AV_HWDEVICE_TYPE_CUDA,
    AV_HWDEVICE_TYPE_VAAPI,
    AV_HWDEVICE_TYPE_DXVA2,
    AV_HWDEVICE_TYPE_QSV,
    AV_HWDEVICE_TYPE_VIDEOTOOLBOX,
    AV_HWDEVICE_TYPE_D3D11VA,
    AV_HWDEVICE_TYPE_DRM,
    AV_HWDEVICE_TYPE_OPENCL,
    AV_HWDEVICE_TYPE_MEDIACODEC,
    AV_HWDEVICE_TYPE_VULKAN,
};
```

通过 av_hwdevice_get_type_name 方法可以将这些枚举值转换成对应的字符串，比如 AV_HWDEVICE_TYPE_MEDIACODEC 对应的字符串就是 mediacodec ，其实在源码里面也是有的：

```cpp
static const char *const hw_type_names[] = {
    [AV_HWDEVICE_TYPE_CUDA]   = "cuda",
    [AV_HWDEVICE_TYPE_DRM]    = "drm",
    [AV_HWDEVICE_TYPE_DXVA2]  = "dxva2",
    [AV_HWDEVICE_TYPE_D3D11VA] = "d3d11va",
    [AV_HWDEVICE_TYPE_OPENCL] = "opencl",
    [AV_HWDEVICE_TYPE_QSV]    = "qsv",
    [AV_HWDEVICE_TYPE_VAAPI]  = "vaapi",
    [AV_HWDEVICE_TYPE_VDPAU]  = "vdpau",
    [AV_HWDEVICE_TYPE_VIDEOTOOLBOX] = "videotoolbox",
    [AV_HWDEVICE_TYPE_MEDIACODEC] = "mediacodec",
    [AV_HWDEVICE_TYPE_VULKAN] = "vulkan",
};
```

和遍历 AVCodec 一样，也要遍历 FFmpeg 是否支持 mediacodec 。

```cpp
type = av_hwdevice_find_type_by_name(mediacodec);
if (type == AV_HWDEVICE_TYPE_NONE) {
    LOGE("Device type %s is not supported.\n", mediacodec);
    LOGE("Available device types:");
    while((type = av_hwdevice_iterate_types(type)) != AV_HWDEVICE_TYPE_NONE)
        LOGE(" %s", av_hwdevice_get_type_name(type));
    LOGE("\n");
    return -1;
}
```

确定支持 mediacodec ，那么解码就可以用了。前面提到，获取文件信息主要是为了打开解码器的，但比如文件编码格式的 H.264 ，而支持 H.264 的解码器除了软解，还有 mediacodec 要怎么选择呢？ 

为了方便，直接 avcodec_find_decoder_by_name 找到 mediacodec 的解码器就行。

```cpp
if (!(decoder = avcodec_find_decoder_by_name("h264_mediacodec"))) {
    LOGE("avcodec_find_decoder_by_name failed.\n");
    return -1;
}
```

找到解码器之后，还要得到该解码器的一些配置信息，比如解码出的格式是什么样子的？mediacodec 解码就是 NV21 这种。

```cpp
for (i = 0;; i++) {
    // 解码器的配置
    const AVCodecHWConfig *config = avcodec_get_hw_config(decoder, i);
    if (!config) {
        LOGE("Decoder %s does not support device type %s.\n",
                decoder->name, av_hwdevice_get_type_name(type));
        return -1;
    }
    if (config->methods & AV_CODEC_HW_CONFIG_METHOD_HW_DEVICE_CTX &&
        config->device_type == type) {
        // 硬解的格式
        hw_pix_fmt = config->pix_fmt;
        break;
    }
}
```

目前 mediacodec 解码还只有 buffer 模式，没有直接解纹理的那种。

接下来就是给解码上下文 AVCodecContext 添加一些硬件解码的上下文。

```cpp
static int hw_decoder_init(AVCodecContext *ctx, const enum AVHWDeviceType type)
{
    int err = 0;
    if ((err = av_hwdevice_ctx_create(&hw_device_ctx, type,
                                      NULL, NULL, 0)) < 0) {
        LOGE("Failed to create specified HW device.\n");
        return err;
    }
    // 硬解解码的上下文
    ctx->hw_device_ctx = av_buffer_ref(hw_device_ctx);
    return err;
}
```

完成了这一系列操作之后，就是正常的解码了，拿到解码后的 AVFrame 内容。

如果 AVFrame 格式和硬件解码的配置格式一样，那么要用 av_hwframe_transfer_data 方法将它做一下转换，转成正常的 YUV 格式。

```cpp
if (frame->format == hw_pix_fmt) {
    /* retrieve data from GPU to CPU */
    if ((ret = av_hwframe_transfer_data(sw_frame, frame, 0)) < 0) {
        LOGE("Error transferring the data to system memory\n");
        goto fail;
    }
    tmp_frame = sw_frame;
} else
    tmp_frame = frame;
```

等完成这一些操作之后，就已经解码成功了，实际运行也是 OK 的。

获取完整源码的话，可以关注微信公众号：音视频开发进阶，回复 1019 获取下载地址。
