---
title: "百倍变速--解码到底能不能丢 非参考帧 ？FFmpeg 有话说！！！"
date: 2021-11-07T10:58:20+08:00
subtitle: ""
categories: ["FFmpeg"]
tags: ["FFmpeg"]
toc: true 
draft: false
original: true
slug: "ffmpeg-decode-drop-frame"
author: "glumes"
---

**昨天周六，群里面还有人在技术交流！！**。

> 默默吐槽一下：这些人真卷啊，大周末还搞技术，是游戏不好玩还是电影不好看。

<!--more-->

![](https://image.glumes.com/blog_image/jianying-100-speed.png)


一开始是讨论剪映的 100 变速是如何实现，群主作为相关人士肯定就不方便透露这些了，不过也有其他大佬给出了思路。

讨论焦点还是围绕如何**丢帧**展开的。



百倍变速，比如正常速度下一帧是播放第 1s 时刻的内容，而变速后要播放 100s 时刻了。

此时的逻辑有以下几种情况：

1. 如果下一个播放时刻要超过目前 GOP 大小了，那么就及时 seek 到离目标 pts 最近的关键帧，比如从 1s 变速后到了 100s ，那就 seek 到第 98s 。
2. 如果下一个播放时刻在同一 GOP 内了，那就继续往下解码，判断解码后的帧 pts 不需要显示就直接丢弃再接着往后解 ，直到接近了目标时间点就显示。
3. seek 后的时间点没达到目标时间点的情况，需要继续解码的可以重复第二步。

以上是针对群内大佬的总结，拿着小本本赶紧记下来。

此时，还有大佬对解码丢帧给出了其他意见：

![](https://image.glumes.com/blog_image/decode_nal_ref_idc.png)

主要是针对非参考帧的丢帧处理，也是文章的重点内容。

当我们通过 av_read_frame 得到一个 AVPacket 之后，可以判断它的 nal_ref_idc 值来决定是否要丢弃。

如果为 0 ，说明其他帧不需要参考该帧，可以直接丢弃不发送给解码器，而不是解码后再丢帧。


如果你不清楚 nal_ref_idc 是什么意思 ？ 那么可以了解一下 H.264 码流 NALU 的概念。

H.264 码流传输时以 NALU 的形式进行，NALU 主要由一个字节的 HAL Header 和 RBSP 两部分组成。

HAL Header 的组成形式如下图所示：

![](https://image.glumes.com/blog_image/760d423740ee655f92fc38525b0eeea.jpg)

HAL Header 的计算如下：

```cpp
forbidden_zero_bit(1bit) + nal_ref_idc(2bit) + nal_unit_type(5bit)
```

nal_unit_type 不同的值代表不同类型的帧，解析 AVPacket 完全可以得到如上的信息，**后面在公众号音视频开发进阶继续更新文章详解如何计算**。

所以，在解码时完全是可以丢弃这些非参考帧的，放心大胆地操作吧。

![](https://image.glumes.com/blog_image/decode_drop_frame.png)

而且丢非参考帧的操作也是经过了产品亿级检测的，这一点我确实可以作证！！！

### FFmpeg 中的丢帧

以上的丢帧逻辑是根据 H.264 规范来的，那么在 FFmpeg 的源码中有没有针对这一逻辑做处理呢？ 

那必然是有的啊！！！

如果仔细看 ffplay 的源码，在源码中有如下的调用方式：

```cpp
/* this thread gets the stream from the disk or the network */
static int read_thread(void *arg)
{
    // 省略部分代码
    for (i = 0; i < ic->nb_streams; i++) {
        AVStream *st = ic->streams[i];
        enum AVMediaType type = st->codecpar->codec_type;
        // AVDISCARD_ALL  抛弃所有的帧
        st->discard = AVDISCARD_ALL;
        if (type >= 0 && wanted_stream_spec[type] && st_index[type] == -1)
            if (avformat_match_stream_specifier(ic, st, wanted_stream_spec[type]) > 0)
                st_index[type] = i;
    }
    
    // 省略部分代码
    if (!video_disable)
        st_index[AVMEDIA_TYPE_VIDEO] =
            av_find_best_stream(ic, AVMEDIA_TYPE_VIDEO,
                                st_index[AVMEDIA_TYPE_VIDEO], -1, NULL, 0);
    if (!audio_disable)
        st_index[AVMEDIA_TYPE_AUDIO] =
            av_find_best_stream(ic, AVMEDIA_TYPE_AUDIO,
                                st_index[AVMEDIA_TYPE_AUDIO],
                                st_index[AVMEDIA_TYPE_VIDEO],
                                NULL, 0);

    /* open the streams */
    if (st_index[AVMEDIA_TYPE_AUDIO] >= 0) {
        // 开启解码线程
        stream_component_open(is, st_index[AVMEDIA_TYPE_AUDIO]);
    }
    ret = -1;
    if (st_index[AVMEDIA_TYPE_VIDEO] >= 0) {
       // 开启解码线程
        ret = stream_component_open(is, st_index[AVMEDIA_TYPE_VIDEO]);
    }
    // 省略代码
}
```
read_thread 方法运行在单独线程上，该方法首先进行解封装操作，然后开启一个线程进行解码，接下来调用 av_read_frame 方法读取 AVPacket 放到队列中供解码线程使用。

在 av_find_best_stream 方法之前先将 discard 置为 AVDISCARD_ALL ，过滤掉 AVStream 中的数据，接下来就是 stream_component_open 操作。

```cpp

/* open a given stream. Return 0 if OK */
static int stream_component_open(VideoState *is, int stream_index)
{
    // 省略部分代码
    // AVDISCARD_DEFAULT 默认模式，过滤无效数据
    ic->streams[stream_index]->discard = AVDISCARD_DEFAULT;
    switch (avctx->codec_type) {
    case AVMEDIA_TYPE_AUDIO:
     // 省略部分代码
        if ((ret = decoder_start(&is->auddec, audio_thread, "audio_decoder", is)) < 0)
            goto out;
    // 省略部分代码
    case AVMEDIA_TYPE_VIDEO:
     // 省略部分代码
        if ((ret = decoder_start(&is->viddec, video_thread, "video_decoder", is)) < 0)
            goto out;
    // 省略部分代码
}
```
stream_component_open 方法又将 discard 置为了 AVDISCARD_DEFAULT ，仅过滤掉无效数据。

继续跟进这个 discard 字段，就会有新发现了！

关于 discard 的所有类型值和使用方式，FFmpeg 中有如下定义：

```cpp
/**
 * @ingroup lavc_decoding
 */
enum AVDiscard{
    /* We leave some space between them for extensions (drop some
     * keyframes for intra-only or drop just some bidir frames). */
     // 不抛弃，不放弃任何数据
    AVDISCARD_NONE    =-16, ///< discard nothing
    //  丢掉无用的数据，比如 size 为 0 这种
    AVDISCARD_DEFAULT =  0, ///< discard useless packets like 0 size packets in avi
    // 丢掉所有的非参考帧
    AVDISCARD_NONREF  =  8, ///< discard all non reference
    // 丢掉所有的双向帧
    AVDISCARD_BIDIR   = 16, ///< discard all bidirectional frames
    // 丢掉所有的非内帧
    AVDISCARD_NONINTRA= 24, ///< discard all non intra frames
    // 丢掉所有的非关键帧
    AVDISCARD_NONKEY  = 32, ///< discard all frames except keyframes
    // 丢掉所有帧
    AVDISCARD_ALL     = 48, ///< discard all
};
```

**所以，完全可以使用 discard 字段来标识解码时丢弃哪些帧。**

另外，在 avcodec_send_packet 方法源码注释中也提示了可以通过 **AVCodecContext.skip_frame** 字段来决定丢弃哪些帧。

```cpp
/**
 * Internally, this call will copy relevant AVCodecContext fields, which can
 * influence decoding per-packet, and apply them when the packet is actually
 * decoded. (For example AVCodecContext.skip_frame, which might direct the
 * decoder to drop the frame contained by the packet sent with this function.)
 *  省略部分注释
 */
int avcodec_send_packet(AVCodecContext *avctx, const AVPacket *avpkt);
```

摘录了部分注释内容，写的就很清楚了。

所以，在解码时也可以不用自己解析 AVPacket 的 nal_ref_idc 字段值，直接通过 AVCodecContext.skip_frame 实现同样的目的。

> 亲测有效，过滤非关键帧之后，解码出来的全是关键帧了。



以上，就是关于丢帧的一些分享，技术交流探讨欢迎加我微信 ezglumes 交流！！！


