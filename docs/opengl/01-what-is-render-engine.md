---
title: "从零打造渲染引擎系列01-什么是渲染引擎"
date: 2021-01-12T19:30:33+08:00
subtitle: ""
slug: "01-what-is-render-engine"
tags: ["engine"]
categories: ["OpenGL"]
toc: true
 
draft: false
original: true
---

前情回顾：[2021 技术新番 - 从零打造渲染引擎系列](https://glumes.com/post/engine/2021-render-engine/)

在开始写代码之前，要先明确渲染引擎到底是什么东西，才能知道要写什么东西。


<!--more-->

在 Google 里面搜索 🔍 渲染引擎关键字，出来的结果都是关于浏览器渲染引擎的。

![](https://image.glumes.com/blog_image/google-search-render-engine.png)

浏览器渲染引擎主要是用来渲染网页内容的，比如内容排版、文字样式那些，显然这不是我想要的。

不过，一方面能看出，搜索引擎对于渲染引擎的定位更偏向于浏览器渲染了，另一方面也是我们搜索的关键字不够清晰准确。

换个方式，直接在 Github 里面搜索 `renderengine` 关键字，出来了大一堆项目，看来很多人都写过此类内容了。

![](https://image.glumes.com/blog_image/github-search-render-engine.png)

以 Google 的 filament 项目为例，地址如下

> [https://github.com/google/filament](https://github.com/google/filament)

filament 是一个基于物理的实时渲染引擎，它可以运行在 Android、iOS、Windows、Linux、macOS 等系统上。

从 README  上可以看到一些例子展示如下：

![](https://image.glumes.com/blog_image/example_materials1.jpg)

![](https://image.glumes.com/blog_image/example_ssr.jpg)

在我还不是特别懂渲染引擎的时候，看到这些图片还是有点懵逼的，尤其是看过几个类似项目之后。

我本想知道渲染引擎用代码写出来之后运行起来是个什么效果，结果就来几张图片，有点 **`开局一张图，内容全靠编`** 的感觉。

后来我才知道，原来这些图就是用渲染引擎渲染出来的效果图。

图中的物体是通过加载模型导入的，然后在原始模型的基础上进行渲染，添加颜色、光照、阴影等内容，最后渲染到屏幕上，而这就是渲染引擎的工作。

看似很简单，但里面每个细节都值得考究。

如果渲染引擎渲染的一张图，你看着就和在现实中用相机拍的图片一样，根本分辨不出是现实还是模拟的，那说明这渲染引擎造诣很早，技术上已经很逼真了。

就好比上面第一张图，那几个物体的位置配合图片的背景，就像有人把东西放在地面上一样，让人傻傻分不清楚。

渲染引擎里面涉及到很多的渲染技术，参考几个 Gihtub 项目，都介绍它们实现了哪些内容，写的越多说明越流弊，比如：

* HDR Lighting
* Deferered Rendering
* Depth of Field
* Screen Spacc grass
* Spot Lighting
* Directional Lighting
* ...

列举一点点做个参考，里面大部分我也不会，听说过概念，后面等开始代码编写了再一一实现一下。

到这里可以尝试给渲染引擎做个简单地定义：就是 **`实现了`** 一系列渲染技术的 **`框架`**。

有两个重点，一个是 **实现了**，一个是 **框架** 。

首先要把渲染引擎作为一个框架，有着完整的架构和生命周期，可以对外提供各项能力；其次就是实现了各项渲染能力，能力不用一次到位，先有再逐渐添加；最后把渲染引擎用到实际项目应用中，呈现出更多精彩内容。

说到这里，有人会觉得这渲染引擎和游戏引擎很类似。是的，渲染引擎更像是游戏引擎的子集，一个很重要的子集，在实现的时候都会去参考游戏引擎的架构设计、功能特点，算是摸着游戏引擎过河了。

像 Unity、Unreal、Cocos 这些游戏引擎，渲染能力都是非常重要的，乃是重中之重呀，但除此之外，游戏引擎还有其他重要功能，比如音频能力、物体系统、脚本系统、网络引擎、人工智能、场景管理等等，这些都是我们暂时用不到，等有需要的时候再来详细分析。

弄清了啥是渲染引擎，后面对要实现的功能就更清晰了，应用 + 框架 + 渲染 ，缺一不可。










