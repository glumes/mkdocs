---
title: "Seek策略以及在有B帧情况下的处理"
date: 2022-03-18T21:04:39+08:00
slug: "video-seek-with-b-frame"
subtitle: ""
categories: ["FFmpeg"]
tags: ["FFmpeg"]
toc: true
 
draft: false
original: true
image: ""
author: "glumes"
---

最近在做 Seek 相关功能时遇到的问题排查，顺便也学到了一些新的东西，和大家分享下。

<!--more-->

在视频播放时执行 Seek 到任意点的操作，一般都是 Seek 到任意点往前最近的 I 帧，然后再逐帧解码到指定时间点。

这里可以优化，假设当前时间和指定时间在一个 GOP 内，就可以不用 seek ，直接顺序向下解码就好。

而正是这个优化出现了一点问题，现象如下：

已经判断播放点 A 和 Seek 点 B 不在一个 GOP 内，然后执行 av_seek_frame 方法还是把时间点 A 所在 GOP 全部解码了，导致播放上出现了卡顿。

这里就很奇怪了，明明判断不在一个 GOP ，那 Seek 时就应该从时间点 B 所在 GOP 的 I 帧开始解码， 但执行时还是解码了上一个 GOP 的内容。

到底是判断是否同一个 GOP 的函数出问题了还是 Seek 方法有问题呢？

带着疑问开始深入源码探索。

FFmpeg 没有直接提供判断两帧是否同一个 GOP 的方法，所以通过 av_index_search_timestamp 方法得到传入时间点最近的 I 帧的 index 索引，如果两个时间点的索引相同则表示为同一个 GOP 内，因为最近的 I 帧相同。


然而 av_index_search_timestamp 方法是通过 AVIndexEntry 中的 timestamp 来判断的，它是一个 DTS 值，通过二分查找得到最近的索引。

在没有 B 帧的情况下，I 帧的 PTS 等于 DTS ，所以判断不会出问题。然而正是有了 B 帧，如果 I 帧的 PTS 和 DTS 不相等的话，那么上面的判断相当于是拿一个 PTS 值和 I 帧的 DTS 比较是否同一个 GOP 了。

如果将错就错，判断 GOP 时得到结论是非同一个 GOP ，那么 Seek 也应该是非同一个 GOP ，但现实恰恰相反，Seek 当做了同一个 GOP ，这里面肯定有计算出问题了，继续深入源码。

通过在 mov.c 源码中看到了如下的操作：

```cpp
static int mov_seek_stream(AVFormatContext *s, AVStream *st, int64_t timestamp, int flags)
{
    MOVStreamContext *sc = st->priv_data;
    FFStream *const sti = ffstream(st);
    int sample, time_sample, ret;
    unsigned int i;
    // Here we consider timestamp to be PTS, hence try to offset it so that we
    // can search over the DTS timeline.
    timestamp -= (sc->min_corrected_pts + sc->dts_shift);
    ret = mov_seek_fragment(s, st, timestamp);
    if (ret < 0)
        return ret;
    sample = av_index_search_timestamp(st, timestamp, flags);
    av_log(s, AV_LOG_TRACE, "stream %d, timestamp %"PRId64", sample %d\n", st->index, timestamp, sample);
    // 省略部分代码
```

注意到如下一行代码：

```cpp
timestamp -= (sc->min_corrected_pts + sc->dts_shift);
```

也就是说我们传入的时间都会被减上一个值，然后再执行 av_index_search_timestamp 方法，而这个值导致判断 GOP 和 Seek 之间的关键帧索引出问题了。

正如代码中的注释所示，假设传入的时间是 PTS 值，然后给它减去偏移以得到 DTS 值，因为 av_index_search_timestamp 方法就通过 DTS 进行比较的嘛。

出现问题的原因就是 seek 的时间点正好在 I 帧的 PTS 和 DTS 范围之间了，执行 seek 时减去偏差值就小于 DTS 了，所以变成了同一个 GOP 。

现在要解决问题就是如何得到 sc->min_corrected_pts + sc->dts_shif 之和，然后判断 GOP 时减去它以修正得到 DTS 值。

还好通过遍历源码发现它的值是不会运行时改变的，一旦决定了就定下来了。另外我们可以用第一个 I 帧的 DTS 值作为偏移值。

```cpp
  auto indexEntry = avStream->index_entries;
  auto nbIndexEntry = avStream->nb_index_entries;
  for (int i = 0; i < nbIndexEntry; ++i) {
    if (indexEntry[i].flags == AVINDEX_KEYFRAME) {
      DTSOffset = indexEntry[i].timestamp;
      return;
    }
  }
```

如果没有 B 帧，DTS 值为 0 ，有 B 帧，那么首帧的 DTS 值就可以用来做偏差值进行计算了。