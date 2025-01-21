---
title: "【音视频连载-010】第二季 FFmpeg 日志打印"
date: 2020-04-27T23:37:03+08:00
subtitle: ""
tags: ["FFmpeg"]
categories: ["FFmpeg"]
toc: true
 
draft: false
original: true
slug: "av-beginner-010"
---

音视频连载系列已经停更一段时间，再这么停下去估计就要掉粉了，捡起来继续更新~~~

接下来主要是讲解 FFmpeg 相关的内容，比如这篇就从简单的日志打印开始说起。

<!--more-->

### 日志打印基础使用

在 FFmpeg 中提供了 av_log() 方法去打印日志，它的函数声明如下：

```cpp
void av_log(void *avcl, int level, const char *fmt, ...)
```

其中 `level` 参数指的是日志级别，后面的 `fmt` 和 `...` 代表日志内容，和调用 `print` 打印信息一样。


具体使用如下：

```cpp
av_log(nullptr,AV_LOG_INFO,"this is INFO log color");
av_log(nullptr,AV_LOG_DEBUG,"this is DEBUG log color");
av_log(nullptr,AV_LOG_WARNING,"this is WARNING log color");
av_log(nullptr,AV_LOG_ERROR,"this is ERROR log color");
```

与 Android 日志类似，FFmpeg 也有多种级别的日志。

```cpp
#define AV_LOG_FATAL     8
#define AV_LOG_ERROR    16
#define AV_LOG_WARNING  24
#define AV_LOG_INFO     32
#define AV_LOG_VERBOSE  40
#define AV_LOG_DEBUG    48
#define AV_LOG_TRACE    56
```

看到 `INFO`、`DEBUG`、`ERROR` 这些级别是不是有似曾相识的感觉。

这些数值都是 8 的倍数，按照从小到大的顺序递增。

### 日志打印级别设置

在 FFmpeg 中可以设置和获取当前日志打印的级别。

```cpp
// 设置日志打印级别
void av_log_set_level(int level);
// 获取日志打印级别
int av_log_get_level(void);
```

比如设置了当前级别是 `AV_LOG_INFO` ，那么凡是级别低于它的都不会打印出来了。

那么什么级别算是更低的呢？数字越小的级别越高，数字越大的级别越低。

比如设置了 `AV_LOG_INFO` 级别，它的值是 32 ，那么 `AV_LOG_DEBUG` 级别的日志就不会打印，它的值是 48 ，级别更低。而 `AV_LOG_ERROR` 就会被打印，它的值是 16 ，级别更高。


### 自定义日志打印

在 FFmpeg 中可以通过 `av_log_set_callback` 函数来注册一个日志回调，在回调中自定义日志打印方式。

`av_log_set_callback` 的函数声明如下：

```cpp
void av_log_set_callback(void (*callback)(void*, int level, const char* fmt, va_list));
```

它的参数是传一个函数指针，其中 `level` 指定了日志回调的级别，根据不同级别做不同操作，`fmt` 和 `va_list` 就是回调的日志内容了，和 `print` 函数相似。

以下就是具体的操作：

```cpp
static void log_callback(void *ptr, int level, const char *fmt, va_list vaList) {
    switch (level) {
        case AV_LOG_DEBUG:
            logD(fmt, vaList);
            break;
        case AV_LOG_VERBOSE:
            logV(LOG_MAGENTA, fmt, vaList);
            break;
        case AV_LOG_INFO:
            logI(fmt, vaList);
            break;
        case AV_LOG_WARNING:
            logW(fmt, vaList);
            break;
        case AV_LOG_ERROR:
            logE(fmt, vaList);
            break;
        default:
            log(fmt, vaList);
            break;
    }
}
```

在 `switch` 做日志级别的分发处理，具体的打印方法教给宏定义的函数。

在这里主要是根据不同级别，调整日志打印输出的颜色，如下图所示：

![](https://user-gold-cdn.xitu.io/2020/4/27/171b9e8714fe442e?w=571&h=271&f=png&s=53129)


> 注意的是，如果注册了自定义日志打印，那么除了我们调用 `av_log` 方法会打印日志之外，FFmpeg 内部的一些日志信息也会通过自定义的方法打印出来。


### 自定义日志打印颜色

一般来说，日志打印都是通过宏函数来定义的。

```cpp
#define logD(format,...)        \
logging(LOG_GREEN,format,##__VA_ARGS__)     
```

其中 `##__VA_ARGS__` 意思就是可变参数宏，对应函数里面的三个点可变参数 `...` 。

具体的函数实现如下：

```cpp
static void logging(const char * color ,const char *fmt, va_list vaList)
{
    // 设置日志打印颜色
    printf("%s [av-beginner]: ",color);
    // 打印内容
    vprintf( fmt, vaList );
    // 结束日志颜色设定
    printf(LOG_NONE "\n" );
}

static void logging(const char * color ,const char *fmt, ...)
{
    va_list vaList;
    va_start( vaList, fmt );
    logging(color,fmt,vaList);
    va_end(vaList);
}
```

这里面涉及到可变参数以及日志颜色打印的内容，展开说一下日志颜色打印。

在终端的字符颜色是由转义序列控制的，比如终端中要换行，那么转义序列就是 `\n` 操作，对于颜色控制同样如此。

具体的显示格式如下：

> \033[显示方式;前景色;背景色m输出字符串\033[0m
> 
> 或
> 
> \e[显示方式;前景色;背景色m输出字符串\e[0m


在调用 print 函数打印信息时，就按照以上的方式即可，比如：

```cpp
// 打印红色的日志内容
printf("\033[0;31m print red color log \033[0m\n") ;
```

以上就可以打印出红色的日志信息，具体的关于显示方式、前景色、背景色这些内容，后续的文章再接着说了。



## 总结

以上就是音视频基础学习连载的 010 篇。

简单讲解了 FFmpeg 中的日志打印内容，本文具体代码见仓库：

> https://github.com/glumes/av-beginner

仓库的代码会比文章提前更新，想要抢先知道后续内容，就关注代码吧，欢迎 star 。

能力有限，文中有不对之处，欢迎加我微信 ezglumes 进行交流~~


