---
title: "【音视频连载-003】基础学习篇-SDL 消息循环和事件响应"
date: 2020-03-03T22:54:26+08:00
subtitle: ""
tags: ["SDL"]
categories: ["SDL"]
toc: true
 
draft: false
original: true
slug: "av-beginner-003"
---

在前面的文章中已经创建了一个 SDL 窗口并且显示指定的颜色。

为了让窗口显示出来，在程序中写了一个死循环，这几行代码就是 SDL 消息循环和事件响应的核心缩影了。

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

<!--more-->

### SDL 消息循环和事件响应

和 Android 中的 Handler 机制有些类似，Handler 会关联一个线程，线程内部维护一个消息队列 MessageQueue，通过 Handler 像 MessageQueue 发送消息，然后再从 MessageQueue 中取出 Message 进行处理。

在 SDL 中通过 `SDL_PollEvent` 从消息队列取出消息，如果有则返回 1，没用则返回 0。 

`SDL_Event` 结构体代表消息事件，其中的 `type` 指定具体的事件类型，在 `SDL_events.h` 文件中能看到所有的事件类型，抄录一些比较典型的：

```cpp
typedef enum
{
   /* Application events */
    SDL_QUIT           = 0x100, /**< User-requested quit */
    
   /* Keyboard events */
    SDL_KEYDOWN        = 0x300, /**< Key pressed */
    SDL_KEYUP,                  /**< Key released */

    /* Mouse events */
    SDL_MOUSEMOTION    = 0x400, /**< Mouse moved */
    SDL_MOUSEBUTTONDOWN,        /**< Mouse button pressed */
    SDL_MOUSEBUTTONUP,          /**< Mouse button released */
    SDL_MOUSEWHEEL,             /**< Mouse wheel motion */
    
    /* Window events */
    SDL_WINDOWEVENT    = 0x200, /**< Window state change */
    SDL_SYSWMEVENT,             /**< System specific event */
} SDL_EventType;
```

如上所示，有键盘、鼠标事件还有窗口事件和应用退出的事件，基本上也就用到这些了。

当用户点击了窗口左上角 `叉` 的时候，对应 SDL_Event 的 type 就是 `SDL_QUIT` ，这个 type 是一定要添加处理的，不然点叉就关闭不了窗口了。

```cpp
    bool bQuit = false;
    while (!bQuit){
        while (SDL_PollEvent(&windowEvent)){
            switch (windowEvent.type){
                case SDL_QUIT:
                    bQuit = true;
                    break;
                default:
                    break;
            }
        }
    }
```

除了 `SDL_PollEvent` 方法去取消息外，还有 `SDL_WaitEvent` 方法。顾名思义，该方法会阻塞当前调用的线程，直到取出一个消息为止。

```cpp
    bool bQuit = false;
    while (!bQuit){
        SDL_WaitEvent(&windowEvent);
        if (windowEvent.type == SDL_QUIT){
            bQuit = true;
            break;
        } else{
            cout << "get event" << endl;
        }
    }
```

使用方法如上所示，但实际上程序不会一直卡在 `SDL_WaitEvent` 上，因为它没有限制监听的事件类型，所以只要有窗口在运行显示，哪怕你鼠标在窗口上滑过、或者按下了键盘，都能算是收到了消息事件，`cout` 方法打印的 log 日志会不断出现的。

同样的，在 `SDL_WaitEvent` 方法中监听了 `SDL_QUIT` 类型的事件，当点击窗口左上角的叉时，也要退出循环，结束程序。

## 键盘响应

现在可以通过 `SDL_Event` 的事件类型来监听特定的键盘事件了。

键盘事件有 `SDL_KEYDOWN` 按下和 `SDL_KEYUP` 抬起两种类型，按需监听。

而具体用户点击键盘上什么按键，这个信息就在 SDL_Event 的 `SDL_KeyboardEvent` 中。

对于不同类型的事件所包含的具体信息，SDL_Event 都有对应的结构体去存储。

```cpp
typedef union SDL_Event
{
    Uint32 type;                    /**< Event type, shared with all events */
    SDL_CommonEvent common;         /**< Common event data */
    SDL_DisplayEvent display;       /**< Window event data */
    SDL_WindowEvent window;         /**< Window event data */
    // 键盘事件的信息
    SDL_KeyboardEvent key;          /**< Keyboard event data */
    // 鼠标事件的信息
    SDL_MouseButtonEvent button;    /**< Mouse button event data */
    SDL_MouseMotionEvent motion;    /**< Mouse motion event data */
}
```

所以想要知道用户点击了哪个按键，去找 `SDL_KeyboardEvent` 对应的信息就好了。

```cpp
    bool bQuit = false;
    while (!bQuit){
        while (SDL_PollEvent(&windowEvent)){
            switch (windowEvent.type){
                case SDL_QUIT:
                    bQuit = true;
                    break;
                case SDL_KEYDOWN:
                    if (windowEvent.key.keysym.sym == SDLK_SPACE){
                        cout << "user click space \n" ;
                    }
                    break;
                default:
                    break;
            }
        }
    }
```

以上代码监听用户是否点击空格键，如果是就输出对应的 log 。

## 鼠标响应

除此之外还可以监听鼠标事件，比如鼠标是否按下、抬起、移动和坐标之类的。

对应的事件类型是 `SDL_MOUSEMOTION` 、`SDL_MOUSEBUTTONDOWN` 、`SDL_MOUSEBUTTONUP` 、`SDL_MOUSEBUTTONUP` ，按自己的需求去监听了。

事件包含的具体信息在 `SDL_MouseMotionEvent` 、`SDL_MouseButtonEvent` 和 `SDL_MouseWheelEvent` 里面。

```cpp
    bool bQuit = false;
    while (!bQuit){
        while (SDL_PollEvent(&windowEvent)){
            switch (windowEvent.type){
                case SDL_QUIT:
                    bQuit = true;
                    break;
                case SDL_MOUSEBUTTONDOWN:
                    printf("button index  is %d\n",windowEvent.button.button);
                    break;
                default:
                    break;
            }
        }
    }
```

以上代码就是监听鼠标点击事件，并且打印出点击按键的 index ,鼠标的左键、右键和中间滚轮按下去对应的 index 不同。

## 自定义事件响应

除了系统事件，还可以自定义事件。

首先定义一个事件类型的宏：

```cpp
#define SDL_CUSTOM_EVENT  (SDL_USEREVENT + 1)
```

其次，要创建一个线程，让它延时五秒后，发送自定义事件，在主线程中去接收到这个事件。

```cpp
// 线程运行函数
int sdl_thread_custom_event(void *){
    // 延时 5 秒
    SDL_Delay(5000);
    // 创建自定义事件并发送到消息队列中去
    SDL_Event sdlEvent;
    sdlEvent.type = SDL_CUSTOM_EVENT;
    SDL_PushEvent(&sdlEvent);
}
// 创建线程并运行
SDL_CreateThread(sdl_thread_custom_event, "custom_event", nullptr);
```

线程运行函数如上所示，定义一个 SDL_Event ，把它的 type 赋值为自定义的类型，然后通过 `SDL_PushEvent` 方法把该消息事件放到消息队列中去。

```cpp
    bool bQuit = false;
    while (!bQuit){
        while (SDL_PollEvent(&windowEvent)){
            switch (windowEvent.type){
                case SDL_QUIT:
                    bQuit = true;
                    break;
                case SDL_CUSTOM_EVENT:
                    cout << "receive user custom event\n";
                    break;
                default:
                    break;
            }
        }
    }
```

SDL_PollEvent 方法会从消息队列中取到我们自定义的消息事件，这时候就能做一些想要的操作呢，比如打印 log 之类的。

## 总结

以上就是关于 SDL 消息循环和事件响应的学习连载 `003` 篇。基本上后续所有的 SDL 代码都会有这样一个消息循环作为程序的主框架，所以这个时候弄明白了，方面后面代码的学习。

具体的代码见仓库：

> https://github.com/glumes/av-beginner

本篇文章对应的提交 `tag` 为 `av-beginner-003`，可切换至对应源码查看。

能力有限，文中有不对之处，欢迎加我微信 ezglumes 进行交流~~