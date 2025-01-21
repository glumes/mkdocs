---
title: "【音视频连载-008】基础学习篇-SDL 播放 PCM 音频文件（下）"
date: 2020-03-22T18:38:09+08:00
subtitle: ""
tags: ["SDL"]
categories: ["SDL"]
toc: true
 
draft: false
original: true
slug: "av-beginner-008"
---

接上篇 SDL 播放 PCM 音频文件，已经实现了 `推` 的模式去播放，接下来看看 `拉` 的模式如何实现。

<!--more-->

## PCM 文件素材准备

前面的文章中已经准备好了相关素材，这里就不重复了，还是用同样的 PCM 文件作为这次实验素材。


## 代码实践

首先还是要通过 `SDL_OpenAudioDevice` 方法打开一个音频设备。

```cpp
    SDL_AudioSpec audioSpec;
    audioSpec.freq = 44100;
    audioSpec.format = AUDIO_S16SYS;
    audioSpec.channels = 2;
    audioSpec.silence = 0;
    audioSpec.samples = 1024;
    // 拉的模式，这里要传一个函数
    audioSpec.callback = fill_audio;

    SDL_AudioDeviceID deviceId;
    if ((deviceId = SDL_OpenAudioDevice(nullptr, 0, &audioSpec, nullptr, SDL_AUDIO_ALLOW_ANY_CHANGE)) < 2) {
        cout << "open audio device failed " << endl;
        return -1;
    }
```

不同的是，这里 `callback` 参数不能是 nullptr 了，要传一个函数指针。这个函数在 `拉` 模式下会不断回调，从而将音频数据填充给设备缓冲区。

函数声明如下：

```cpp
typedef void (SDLCALL * SDL_AudioCallback) (
    // 传用户自定义的数据
    void *userdata, 
    // 指向要填充给设备缓冲区的音频数据Buffer的指针
    Uint8 * stream,
    // 音频数据Buffer的长度
    int len);
```

参数 `stream` 是个指针类型，它指向要填充给设备缓冲区的音频数据 Buffer ，而 len 就是 Buffer 的长度。`userdata` 是我们自定义的数据，需要的时候可以用到。

在这个函数中我们要做的就是将读取的 PCM 音频数据传给 `stream` 指向的 Buffer ，而且还不能超出 len 的长度，如果超出了截断一下，下次回调时传剩下的部分。

因此就有了如下的实现：

```cpp
// 读取出 pcm 数据长度
static Uint32 audio_len;
// 读取出的音频数据 Buffer
static Uint8 *audio_pos;

// 函数实现
void fill_audio(void *udata, Uint8 *stream, int len) {
    SDL_memset(stream, 0, len);
    if (audio_len == 0) {
        return;
    }
    // 数据大小不能超过 len
    len = len > audio_len ? audio_len : len;
    
    // 将 stream 和 audio_pos 进行混合播放
//    SDL_MixAudio(stream, audio_pos, len, SDL_MIX_MAXVOLUME);

    // 单独播放 audio_pos,也就是解码出来的音频数据
    memcpy(stream, audio_pos, len);

    audio_pos += len;
    audio_len -= len;

    if (audio_len <= 0){
        // 读取完了，通知继续读取数据
        notifyGetAudioFrame();
    }
}
```


首先将 `stream` 数据清空。然后比较读出的 pcm 数据长度 `audio_len` 和 `len` 的大小，保证数据大小不超过 `len` 的要求。

在播放时，也就是给 `stream` 写数据时有两种方式。一种是直接 `memcpy` 将音频数据 `audio_pos` 拷贝到 Buffer 上就好了。另一种是通过 `SDL_MixAudio` 方法。


`SDL_MixAudio` 方法顾名思义就是混音了，将 `stream` 和音频数据 `audio_pos` 混合播放，由于一开始就将 `stream` 数据清空为 0 了，所以看似混音，实际上和直接播放音频数据效果一致的。


最后，如果读出的 pcm 数据长度大于 `len`，那说明数据还没有全部填充完，下一次回调把剩下的填充到缓冲区，同时移动相应的指针位置。

如果小于，就得通知继续读取数据了，这里自定义了一个事件去通知应用读取音频数据。

```cpp
// 自定义事件，通知读取音频数据
void notifyGetAudioFrame(){
    SDL_Event sdlEvent;
    sdlEvent.type = SDL_EVENT_BUFFER_END;
    SDL_PushEvent(&sdlEvent);
}

// 在程序事件循环中去响应事件，读取音频 Buffer
 while (!bQuit) {
        while (SDL_PollEvent(&windowEvent)) {
            switch (windowEvent.type) {
                case SDL_EVENT_BUFFER_END:
                    // 读取音频数据
                    if (fread(buffer, 1, bufferSize, pFile)) {
                        data_count += bufferSize;
                        audio_chunk = reinterpret_cast<Uint8 *>(buffer);
                        audio_len = bufferSize;
                        audio_pos = audio_chunk;
                    }
                default:
                    break;
            }
        }
    }
```

在事件的消息循环中进行响应，读取音频 Buffer 。如果读取的到的长度等于 0 了，也可以通过 `fseek` 方法将指针 seek 到 0，循环读取。


最后运行一下程序，就会播放出和原来 mp3 一样的音乐了。


## 总结

以上就是音视频基础学习连载的 `008` 篇。

通过两篇文章讲解了 SDL 播放音频的两种方式，后续会主要以 `拉` 的模式进行开发。

本文具体代码见仓库：

> https://github.com/glumes/av-beginner

本篇文章对应的提交 `tag` 为 `av-beginner-004`，可切换至对应源码查看。

能力有限，文中有不对之处，欢迎加我微信 ezglumes 进行交流~~