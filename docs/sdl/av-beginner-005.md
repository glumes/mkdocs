---
title: "【音视频连载-005】基础学习篇-SDL 加载 YUV 文件并显示"
date: 2020-03-10T00:05:03+08:00
subtitle: ""
tags: ["SDL"]
categories: ["SDL"]
 
toc: true
draft: false
original: true
slug: "av-beginner-005"
---


在前面的文章中已经完成了图片的加载和显示，接下来要做的就是加载 YUV 文件并显示。


<!--more-->

## YUV 文件素材准备

做这个功能点比较麻烦的是素材问题，上哪去找一个 YUV 文件出来，谷歌和百度搜索都不好使，其实直接使用 FFmpeg 生成文件就好了。

使用如下命令将图片文件转成 YUV 文件：

```cpp
// 把 filename 改为图片对应的文件名
ffmpeg -i image_filename.png -pix_fmt yuv420p yuv_filename.yuv
```

在这里将 YUV 文件格式固定为 `YUV420P` 了，如果你对 YUV 格式不懂的话，强烈建议看看我之前写过的文章，图文并茂，清晰易懂，使用谷歌搜索 YUV 关键字，搜索结果排名前五的必有我这篇文章。

> [一文读懂 YUV 的采样与格式](https://mp.weixin.qq.com/s/taMHYFZunxQ3XNZwR_U-Cg)


顺便可以使用 FFplay 验证生成的 YUV 文件是否有效，使用如下命令：

```cpp
// 100x100 代表图片的宽高，这里只是举例，换成实际的宽高
// 把 filename 改为 YUV 对应的文件名
ffplay -f rawvideo -video_size  100x100 yuv_filename.yuv
```

以上命令会打开一个窗口去展示图片，如果该图片和未转换成 YUV 的图片内容一致，那说明把图片转换成 YUV 格式文件是成功了，这样就有了实验素材。


## 代码实践

有了素材，接下来就是代码实践环节：

### 创建纹理 SDL_Texture

与 SDL 显示图片的方式有些不同，显示图片是将图片转换成了 SDL_Surface，然后将这个 SDL_Surface 的内容转换到 SDL_Window 对应的 SDL_Surface 上，最后上屏就显示图片了。

显示 YUV 文件需要创建一个纹理，然后将纹理内容渲染上屏，这类似于 OpenGL 的操作了。

首先通过 `SDL_CreateTexture` 方法创建 `SDL_Texture`：

```cpp
SDL_CreateTexture(SDL_Renderer * renderer,
                Uint32 format,
                int access, int w,
                int h);
```

参数 `renderer` 是之前文章提到的创建渲染器，`w` 和 `h` 是纹理的宽高，重点是 `format` 参数。


根据文件格式的不同，`format` 参数也不同，比如这里文件是 `YUV420P`，那么对应的就是 `IYUV`.

更多格式可以参考 `SDL_pixels.h` 文件中定义，摘录部分如下：

```cpp
    SDL_PIXELFORMAT_UYVY =      /**< Packed mode: U0+Y0+V0+Y1 (1 plane) */
        SDL_DEFINE_PIXELFOURCC('U', 'Y', 'V', 'Y'),
    SDL_PIXELFORMAT_YVYU =      /**< Packed mode: Y0+V0+Y1+U0 (1 plane) */
        SDL_DEFINE_PIXELFOURCC('Y', 'V', 'Y', 'U'),
    SDL_PIXELFORMAT_NV12 =      /**< Planar mode: Y + U/V interleaved  (2 planes) */
        SDL_DEFINE_PIXELFOURCC('N', 'V', '1', '2'),
    SDL_PIXELFORMAT_NV21 =      /**< Planar mode: Y + V/U interleaved  (2 planes) */
        SDL_DEFINE_PIXELFOURCC('N', 'V', '2', '1'),
    SDL_PIXELFORMAT_EXTERNAL_OES =      /**< Android video texture format */
        SDL_DEFINE_PIXELFOURCC('O', 'E', 'S', ' ')
```

### 打开文件并读取内容

接下来就是打开文件操作了，直接根据 C 语言里面相关方法调用就行。

```cpp
    // 打开文件
    FILE *pFile = fopen(path.c_str(), "rb");
    // 读取文件内容到 buffer 中
    unsigned char *yuv_data;
    // yuv420p 格式的文件大小
    int frameSize = width * height * 3 / 2;
    yuv_data = static_cast<unsigned char *>(malloc(frameSize * sizeof(unsigned char)));
    fread(yuv_data,1,frameSize,pFile);
    // 关闭文件
    fclose(pFile);
```

打开文件，读取文件内容，注意 `YUV420P` 格式文件大小的计算方式是 `width * height * 3 / 2` ，它比正常的 `RGBA` 格式文件要小一点的。

因为读取了文件内容之后，后续也就用不到了，直接 fclose 关闭掉。


## 渲染纹理上屏

有了纹理，也有了 YUV 文件内容，接下来就是把 YUV 文件内容转换到纹理上，在把纹理渲染上屏。

```cpp
    if (texture != nullptr){
        SDL_Event windowEvent;
        while (true){
            if (SDL_PollEvent(&windowEvent)){
                if (SDL_QUIT == windowEvent.type){
                    break;
                }
            }
            // 更新纹理内容，就是把读取的 YUV 数据转换成纹理
            SDL_UpdateTexture(texture, nullptr,yuv_data,width);
            // 清屏操作
            SDL_RenderClear(renderer);
            // 将指定纹理复制到要渲染的地方
            SDL_RenderCopy(renderer,texture, nullptr, nullptr);
            // 上屏操作
            SDL_RenderPresent(renderer);
        }
        SDL_DestroyWindow(window);
        SDL_Quit();
    }
```

首先调用 `SDL_UpdateTexture` 方法将 YUV 内容转换成纹理，然后 `SDL_RenderClear` 清屏操作，OpenGL 相关的渲染也是要清屏操作的。

接下来将 `SDL_Texture` 拷贝到要渲染的地方，然后 `SDL_RenderPresent` 执行上屏操作就行了。

渲染纹理上屏的操作流程基本都是这样了，根据文件格式的不同，转换成纹理的方式也有不同，除了 `SDL_UpdateTexture` 方法之外，还有 `SDL_UpdateYUVTexture` 方法，后面会遇到的。

最后别忘了释放和销毁相应的指针和变量。

运行程序就会看到打开一个窗口，显示一张图片，和之前用 FFmpeg 显示的图片内容一致。


![](https://ae01.alicdn.com/kf/U8ba2831c2a6f498e944fc48bbb3f4e6bE.jpg)


## 总结

以上就是音视频基础学习连载的 `005` 篇。

内容相对比较简单，对于 SDL 接口的一些调用也不算难。实际上并不用太深究 SDL 的接口机制和实现原理，做一些实验性入门基础功能会用好了，毕竟在实际工作中不太会用到。

另外，既然已经可以显示一张 YUV 帧内容了，那么假如是一个 YUV 视频文件又该如何显示了？想知后事如何，请看下回分解。


本文具体代码见仓库：

> https://github.com/glumes/av-beginner

本篇文章对应的提交 `tag` 为 `av-beginner-004`，可切换至对应源码查看。

能力有限，文中有不对之处，欢迎加我微信 ezglumes 进行交流~~