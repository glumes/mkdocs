---
title: "【音视频连载-006】基础学习篇-SDL 播放 YUV 视频文件"
date: 2020-03-15T18:38:03+08:00
subtitle: ""
tags: ["SDL"]
categories: ["SDL"]
toc: true
 
draft: false
original: true
slug: "av-beginner-006"
---

在前面的文章中，我们已经能够加载 YUV 帧并显示了，那是把一张图片转换成 YUV 帧得到的素材。


如果是一个 YUV 视频文件的话，那就是很多 YUV 帧连续在一起，既然能展示一帧，那肯定可以连续展示多帧。

接下来就要这样的操作。

<!--more-->

## YUV 视频文件素材

还是要准备一下 YUV 视频素材，不用网上到处去下载，用 FFmpeg 命令将 `mp4` 文件转换成 `yuv` 文件就好了。

```cpp
ffmpeg -i file_name.mp4 filename.yuv
```

命令很简单，其中 `file_name` 是文件的名称，使用时记得替换。

由于生成的 yuv 文件是未经过压缩的，文件过大不方便传 Github ，所以在程序运行前要自己去生成一下文件。

同样，也可以用 `ffplay` 验证一下 yuv 文件转换是否正确。

```cpp
ffplay -f rawvideo -video_size  100x100 yuv_filename.yuv
```

以上命令会打开一个窗口去播放视频内容，如果播放的和原来 mp4 文件内容一致，说明转换是成功的，YUV 文件可用。



## 代码实践

接下来就是代码实践环节，很多地方和前一篇文章[加载 YUV 文件并显示](https://mp.weixin.qq.com/s/nCidtYLmB8_LtAzbW14skg) 是类似的。


```cpp
    // 打开文件 和 创建纹理 的代码和前一篇一样，不在放上来了
    if (texture != nullptr){
        SDL_Event windowEvent;
        while (true){
            if (SDL_PollEvent(&windowEvent)){
                if (SDL_QUIT == windowEvent.type){
                    break;
                }
            }
            // 读取内容
            if (fread(yuv_data,1,frameSize,pFile) != frameSize){
                // 读取内容小于 frameSize ，seek 到 0 ，重新读取，类似于重播
                fseek(pFile,0,SEEK_SET);
                fread(yuv_data,1,frameSize,pFile);
            }
            //
            SDL_UpdateTexture(texture, nullptr,yuv_data,width);
            SDL_RenderClear(renderer);
            SDL_RenderCopy(renderer,texture, nullptr, nullptr);
            SDL_RenderPresent(renderer);
        }
        SDL_DestroyWindow(window);
        SDL_Quit();
    }
```

打开文件和创建纹理代码内容和前面的一致，就不重新贴出来了。

YUV 内容转纹理以及渲染纹理上屏的操作也是一样的。

不同的是，读取 buffer 的操作放在了 while 里面。

如果对 SDL 的消息循环和事件响应还记得的话，就能明白每当 SDL_PollEvent 从消息队列中取出一个消息，只要不是退出事件，就会从 YUV 文件中读取 Buffer 并把它转成纹理渲染上屏。


如果读取的 Buffer 内容小于一帧 YUV 文件的大小，就会  Seek 到文件开头的位置，重新读取，类似于重播了。当然你也可以不重播，直接退出。


以下就是实际的运行效果：




以上的代码还是存在问题的，比如 YUV 视频播放的很快，比原来的 mp4 播放快多了。

这是因为播放的速率控制完全是根据 SDL_PollEvent 事件响应的速度来的，而不是根据 mp4 的帧率来播放。

可以通过自定义 SDL 事件，然后根据帧率控制自定义事件的发送速率，实现控制播放速度的目的。

另外，这里有很多参数都是事先知道的，比如视频宽高数据，在后面我们将通过 FFmpeg 来得到这些数据，实在真正的解码播放。

## 总结


以上就是音视频基础学习连载的 `006` 篇。

在实现加载 YUV 帧并显示的基础上，很容易就实现播放 YUV 视频文件了。

本文具体代码见仓库：

> https://github.com/glumes/av-beginner

本篇文章对应的提交 `tag` 为 `av-beginner-004`，可切换至对应源码查看。

能力有限，文中有不对之处，欢迎加我微信 ezglumes 进行交流~~