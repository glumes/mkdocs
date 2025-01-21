---
title: "FFmpeg 3.0 版本视频解码浅析 "
subtitle: "MPEG封装格式到PCM数据格式"
date: 2017-12-22T16:05:30+08:00
categories: ["FFmpeg"]
tags: ["FFmpeg"]
 
toc: true
original: true
addwechat: true
slug: "ffmpeg-convert-mpeg-to-pcm"
---


有了上一篇文章基础，这里就只关注 FFmpeg 如何解析的具体实践了。 

在开始工程之前，第一步要做的就是编译 FFmpeg 源码，生成 Android 平台上使用的 so 库。

在生成完了之后，导入 Android 工程项目中，并且配置 CMake 文件，添加对应的库，就可以开始开发了。

<!--more-->

如果觉得麻烦，也可以直接从网上找一个工程使用现成的东西，也可以参考的我的学习项目：[https://github.com/glumes/FFMPEGLearn](https://github.com/glumes/FFMPEGLearn)

## FFmpeg 3.0 与 2.0 的不同之处

如果是刚开始学习，完全可以参照雷神的博客来写例子，而且还能运行成功。不过，既然 FFmpeg 都出了 3.0 版本了，还是要跟进学习一下。

FFmpeg 2.0 解码流程大致如下：

![ffmpeg](http://7xqe3m.com1.z0.glb.clouddn.com/ffmpeg.png)

而 FFmpeg 3.0 就开始有点变化了。

![ffmpeg-outline](http://7xqe3m.com1.z0.glb.clouddn.com/ffmpeg3.0-outline.png)

可以看到，在具体解析操作时发生了变化，不再是之前的`avcodec_decode_video2`函数，而是拆分成了两个函数，`avcodec_send_packet`和`avcodec_receive_frame`。

`avcodec_send_packet`负责发送编码压缩后的数据，也就是 Packet 数据，而`avcodec_receive_frame`负责接收解码后的原始数据，也就是 Frame 数据，这么一来一回的配合，每次从文件中读取一部分内容，直到将整个文件解码完成。

## 解码流程代码

FFmpge 的解码流程大都是相同的，只是在细节地方会有差异，主体流程还是没变，看了好多解码的代码，大多也是这样的，只是在解码成具体的某些格式时，对原始数据 Frame 的操作会有不同。

``` java
// 声明一堆待会要用到的变量
    AVCodecContext *avCodecContext = nullptr;
    AVCodec *avCodec = nullptr;
    AVCodecParserContext *parserContext = nullptr;
    AVPacket *packet;
    AVFrame *frame;

    // 为上面声明的变量在赋值，并且检查赋值是否成功
	avcodec_register_all()
	// 为解码器赋值，并制定解码器类型
	avCodec = avcodec_find_decoder(AV_CODEC_ID_HEVC);
	// 为 AVCodecParserContext 和 AVCodecContext 赋值
	parserContext = av_parser_init(avCodec->id);
	avCodecContext = avcodec_alloc_context3(avCodec);
	// 打开解码器
	avcodec_open2(avCodecContext, avCodec, nullptr);
	// 为 Packet 和 Frame 赋值
    frame = av_frame_alloc();
    packet = av_packet_alloc();

    while // 开始一点点的解码文件内容，直到文件末尾
    	// 解码从 while 中一次读取到的内容
    	av_parser_parse2
    	// 发送读取到的 packet
    	avcodec_send_packet
    	// 接收解析完的 frame
    	avcodec_receive_frame
		// 对原生数据进行相关操作，解码成不同数据格式，操作不同。
		// todo
	
	//最后，释放上面声明的变量
	av_frame_free
	av_packet_free
	avcodec_free_context
	av_parser_close

```

## MPEG->PCM 的解码

在了解视频文件的相关基础以及 FFmpeg 解码的大致流程后，就可以开始进行 MPEG->PCM的解码工作了。

不过，在这里还是少了两个重要的概念：

1. FFmpeg 各种类以及变量的含义。
2. 各种音视频文件的格式协议。

前半部分还好，对着雷神的博客一点点的看，还是能看懂的。至于各种音视频文件的协议，那就要靠慢慢积累了。

这里就不贴上全部的代码了，可以参考如下两个地方：

1. http://ffmpeg.org/doxygen/trunk/decode_video_8c-example.html
2. https://github.com/glumes/FFMPEGLearn/blob/master/FFmpegLib/src/main/cpp/ffmpeg_examples/decode_video.cpp


## 参考

1. http://www.jianshu.com/p/7e96223ff329
2. http://blog.csdn.net/leixiaohua1020/article/details/42181271

