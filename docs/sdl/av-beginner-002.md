---
title: "【音视频连载-002】基础学习篇-SDL 创建窗口并显示颜色"
date: 2020-03-02T20:40:02+08:00
subtitle: ""
tags: ["SDL"]
slug: "av-beginner-002"
categories: ["SDL"]
toc: true
 
draft: false
original: true
---

在前面的文章中我们已经完成了 SDL 的工程配置，接下来就是 SDL 相关功能的开发。

本篇文章主要是创建一个应用程序窗口并显示。

<!--more-->

### 创建 SDL 窗口

通过 SDL 中的 `SDL_CreateWindow` 函数就能够创建了，

```cpp
extern DECLSPEC SDL_Window * SDLCALL SDL_CreateWindow(
        const char *title,
        int x, int y, int w,
        int h, Uint32 flags);
```

其中，`title` 就是窗口的标题，`x`,`y`,`width`,`height` 就是窗口的左上角坐标点和宽高值。

最后的 `flags` 参数有很多类型，不同的类型对应不同的窗口功能，比如窗口全屏、是否可见等，具体可以在 SDL 源码中找到。

这里简单设置成如下：

```cpp
    int width = 400;
    int height = 400;
    SDL_Window *window = SDL_CreateWindow("Hello SDL",
            SDL_WINDOWPOS_CENTERED,
            SDL_WINDOWPOS_CENTERED,
            width,height,
            SDL_WINDOW_ALLOW_HIGHDPI);
```

对于窗口的左上角坐标点使用 SDL 默认的宏 `SDL_WINDOWPOS_CENTERED` 让它居中显示就好了，flags 使用 `SDL_WINDOW_ALLOW_HIGHDPI`。


### 展示 SDL 窗口

SDL_Window 并没有什么 `show` 的方法，看到网上的文章应该一创建就可以显示出来了，如果出现随着程序退出，窗口一闪而过的情况加个 `SDL_Delay` 延时一段时间也行。

不过可能是由于 MAC 系统或者 SDL 版本的问题，实际上并没有窗口弹出来，倒是在任务栏中确实能看到有个程序在运行。

后来在创建窗口之后，加上如下的代码就好了：

```cpp
    SDL_Event windowEvent;
    while (true){
        if (SDL_PollEvent(&windowEvent)){
            if (SDL_QUIT == windowEvent.type){
                break;
            }
        }
    }
```

在程序中创建一个死循环，当做消息循环机制，只有当满足特定条件时才退出循环结束程序。

添加这段代码之后在运行，就能看到窗口了。

![](https://images.xiaozhuanlan.com/photo/2020/1d025b77a1530eedbf0782e26e233bf6.png)

### 渲染 SDL 窗口

现在还是一个黑漆漆的窗口，那是因为还没有给它渲染上颜色。

渲染窗口，首先要创建一个渲染器，并设置渲染颜色，然后开始渲染。

如下代码所示：

```cpp
    SDL_Renderer* pRenderer = NULL;
    // 创建渲染器
    pRenderer = SDL_CreateRenderer(window, -1, 0);
    // 指定渲染颜色
    SDL_SetRenderDrawColor(pRenderer,0,255,0,255);
    // 清空当前窗口的颜色
    SDL_RenderClear(pRenderer);
    // 执行渲染操作，更新窗口
    SDL_RenderPresent(pRenderer);
```

调用 `SDL_CreateRenderer` 方法来创建渲染器，并通过 `SDL_SetRenderDrawColor` 来指定颜色，颜色参数都是 `red`、`green`、`blue`、`alpha` 四个，这里指定了渲染为绿色。

然后通过 `SDL_RenderClear` 方法清空一下当前窗口上的颜色，避免和要渲染的颜色混在一起了，最后就可以执行渲染了。

这个流程和 OpenGL 的渲染操作有点类似了：

```cpp
glClearColor()
glClear()
glDrawArrays() 
```

也是先清空后渲染，实际效果如下：

![](https://images.xiaozhuanlan.com/photo/2020/26c7c7f2bd4a2135d668ff898352fd3f.png)

这样就创建了一个窗口，并且显示指定颜色。

### 销毁 SDL 窗口

最后，当退出循环时，要执行销毁操作，把创建的 SDL_Window 和 SDL_Renderer 都释放了。

```cpp
    SDL_DestroyWindow(window);
    SDL_DestroyRenderer(pRenderer);
    SDL_Quit();
```

这样就完成了整个程序。

### 总结

以上就是音视频基础学习连载的 `002` 篇。

具体代码见仓库：

> https://github.com/glumes/av-beginner

本篇文章对应的提交 `tag` 为 `av-beginner-002`，可切换至对应源码查看。

能力有限，文中有不对之处，欢迎加我微信 ezgluems 进行交流~~