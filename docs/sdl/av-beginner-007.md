---
title: "【音视频连载-007】基础学习篇-SDL 播放 PCM 音频文件（上）"
date: 2020-03-16T09:47:30+08:00
subtitle: ""
tags: ["SDL"]
categories: ["SDL"]
toc: true
 
draft: false
original: true
slug: "av-beginner-007"
---

在前面的文章中已经能够利用 SDL 去播放 YUV 视频文件了，接下来要通过 SDL 去播放 PCM 音频文件。


SDL 播放音频文件有两种方法，可以理解成 `推(push)`和`拉(pull)`两种模式。

`推` 就是我们主动向设备缓冲区填充 Buffer ，而 `拉` 就是由设备拉取 Buffer 填充到缓冲区。

在一些开发模型中，如果数据传递能够抽象成`流`的形式，那么肯定就会有`推`和`拉`两种模式。


本篇文章主要是讲解 SDL 以推的形式播放音频文件。

<!--more-->

## PCM 文件素材准备

首先还是得准备素材，做音视频相关实验就是这么麻烦~~

找一个 mp3 文件，使用 FFmpeg 命令将它转换成 pcm 文件，方便的话可以直接使用代码仓库提供的 mp3 文件。

不像在视频播放中准备素材那样简单，音频文件对于参数的信息要求多一点。首先要使用 ffmpeg 查看 mp3 文件的一些信息，比如采样率、声道数等。

```cpp
ffmpeg -i file_name.mp3
```


![](https://images.xiaozhuanlan.com/photo/2020/7d7d871258164b1d1b6d211ca6d961a2.jpg)

得到如图所示的信息，可以看到 mp3 文件采样率是 44100 Hz，双声道，再使用 FFmpeg 转换时要用到上面的信息。

```cpp
ffmpeg -i file_name.mp3 -acodec pcm_s16le -f s16le -ac 2 -ar 44100 file_name.pcm
```

其中：

* -acodec pcm_s16le
    * 指定编码器

* -f s16le
    * 指定文件格式，是大端模式还是小端模式

* -ac 2 
    * 指定通道数，2 代表双通道

* -ar 44100
    * 指定采样率，这里是 44100 Hz


在转换时要根据原文件的采样率和声道数进行转换，否则转换后的 pcm 文件播放声音不对了。

```cpp
ffplay -ar 44100 -channels 2 -f s16le -i file_name.pcm
```

通过上面的命令可以验证转换是否正确，还是要注意声道数和采样率的设置，如果没问题的话，说明 PCM 文件素材就准备完毕，可以进行代码实践了。

## 代码实践


首先要通过 `SDL_OpenAudioDevice` 方法打开一个音频设备。

```cpp
SDL_OpenAudioDevice(const char
                 *device,
                 int iscapture,
                 const SDL_AudioSpec * desired,
                 SDL_AudioSpec *obtained,
                 int allowed_changes);
```

其中结构体 `SDL_AudioSpec` 指定了一系列音频相关的参数，具体如下：

```cpp
typedef struct SDL_AudioSpec
{
    int freq;                   /**< DSP frequency -- samples per second */
    SDL_AudioFormat format;     /**< Audio data format */
    Uint8 channels;             /**< Number of channels: 1 mono, 2 stereo */
    Uint8 silence;              /**< Audio buffer silence value (calculated) */
    Uint16 samples;             /**< Audio buffer size in sample FRAMES (total samples divided by channel count) */
    Uint16 padding;             /**< Necessary for some compile environments */
    Uint32 size;                /**< Audio buffer size in bytes (calculated) */
    SDL_AudioCallback callback; /**< Callback that feeds the audio device (NULL to use SDL_QueueAudio()). */
    void *userdata;             /**< Userdata passed to callback (ignored for NULL callbacks). */
} SDL_AudioSpec;
```

这些参数和音频是息息相关的，比如采样率、声道、音频数据格式、采样个数等，后面会专门去介绍它们。

`SDL_OpenAudioDevice` 方法有两个参数 `desired` 和 `obtained` 都是 `SDL_AudioSpec` 类型的。

这里的意思是我们传入 `desired` 指定的音频参数，但不一定是 SDL 支持的，所以 SDL 会返回一个它支持的参数信息放在 `obtained` 里面。

不过为了简单就先把它写死好了，但即使写死了有些信息还是要和你的 PCM 文件对应上才行，比如 `freg` 采样率和 `channels` 通道数等。

```cpp
    SDL_AudioSpec audioSpec;
    audioSpec.freq = 44100;
    audioSpec.format = AUDIO_S16SYS;
    audioSpec.channels = 2;
    audioSpec.silence = 0;
    audioSpec.samples = 1024;
    // 因为是推模式，所以这里为 nullptr
    audioSpec.callback = nullptr;

    SDL_AudioDeviceID deviceId;
    if ((deviceId = SDL_OpenAudioDevice(nullptr,0,&audioSpec, nullptr,SDL_AUDIO_ALLOW_ANY_CHANGE)) < 2){
        cout << "open audio device failed " << endl;
        return -1;
    }
```

注意到 `SDL_AudioSpec` 有个参数 `callback` 设置为了 nullptr 。这个回调是为了在 `拉` 模式中从回调取数据的，因为这里暂时用不到就写成了 nullptr ，下一篇文章就会用到了。

这样就打开了音频设备，返回一个文件 Id，如果结果小于 2 说明打开失败了。


接下来通过 `SDL_PauseAudioDevice` 方法去播放或者暂停音乐。

```cpp
SDL_PauseAudioDevice(SDL_AudioDeviceID dev,
                   int pause_on);
```

`SDL_AudioDeviceID` 参数就是上面返回的文件 Id，`pause_on` 为 0 的话代表播放，1 代表暂停。

最后就要开始主动向设备缓冲区填充 Buffer 了。

就向 SDL 播放 YUV 视频那样，要从 PCM 文件中读取一块 Buffer ，然后通过 `SDL_QueueAudio` 方法进行填充。

```cpp
    int bufferSize = 4096;
    char* buffer = (char *)malloc(bufferSize);
    // 省略中间代码
    num =  fread(buffer,1,bufferSize,pFile);
    if (num){
        // 填充
        SDL_QueueAudio(deviceId,buffer,bufferSize);
    }
```

如上代码，首先定义了缓冲区的大小 4096，然后 fread 方法读取这么大的内容，最后把它填充进去。


此时运行程序，就会听到和原来 mp3 文件一样的声音了。


不过这里有要注意的地方，并不是填充了一下 Buffer 就马上会有声音播放出来的，要多填充一些才会有声音播放。

另外，当播放声音时，必须要让程序不能退出，因为音频播放并不是一个阻塞当前主线程的方法，填充完数据就不管了的话，是听不到声音的。要么加个 SDL_Delay 方法要么就把 `SDL_QueueAudio` 方法放在接受消息队列信息的循环中，我采用的就是后者。

## 总结

以上就是音视频基础学习连载的 `007` 篇。

本文具体代码见仓库：

> https://github.com/glumes/av-beginner

本篇文章对应的提交 `tag` 为 `av-beginner-004`，可切换至对应源码查看。

能力有限，文中有不对之处，欢迎加我微信 ezglumes 进行交流~~



