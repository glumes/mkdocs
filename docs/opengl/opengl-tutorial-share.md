---
title: "OpenGL ES 学习资源分享"
date: 2018-07-11T14:25:52+08:00
subtitle: ""
draft: false
categories: ["OpenGL"]
tags: ["OpenGL"]
toc: true
 
original: true
addwechat: true
slug: "opengl-tutorial-share"
author: "glumes"
---

学习了一段时间的 OpenGL ES，并在公司的项目中得到了运用，也算是有了一些积累，现在分享一些当初学习的资源，大家一起来学习，共同交流进步。

<!--more-->

## 关于学习方式

在分享资源之前，简单地聊聊学习的方式。

有句名言说的好：

> 书籍的人类进步的阶梯

在需要解决一些未知领域的问题、完成一些未知领域的需求时，是必须要去学习一些新东西的。

而在学习这些新东西时，不要太依赖于搜索引擎了，不然只是当下解决了某些问题、完成了某些需求。

通过看一些博客文章、看一些文章分析，在某些时刻确实是很有帮助的，但总是会存在一些碎片化知识，没有系统地形成知识网络，此时掌握的仅仅是技巧。还是要通过系统地去学习某些知识内容，在脑海里面有个完整的知识体系。

这个简单的道理大家都懂，就不多说了~


## 简单上手

作为程序员学习一项内容，最重要的就是 Hello World 了。

> 《OpenGL ES 应用开发实践指南》

![](https://image.glumes.com/images/2019/04/27/opengles2.png)

这本书比较通俗易懂，直接上手使用 OpenGL ES，可以说是手把手教学了。

作为初学者，最重要的是啥？环境配置、Demo 运行呀~~~

在 《OpenGL ES 应用开发实践指南》里面，跟着书中的章节顺序走，每一章都会有代码示例，也算是一步步引导了。

你可以暂时不求甚解，先把示例工程运行起来，等熟练了再去深究原理。

美中不足的是，这本书针对的 OpenGL ES 版本是 2.0 的，在 OpenGL ES 3.x 中的一些特性无法体验到了，而且现在的手机大多支持 OpenGL ES 3.x 版本了，不过要是考虑到兼容低版本的情况，还是可以使用 OpenGL ES 2.0 版本的。

这本书是翻译过来的，它的英文原版封面如下：

![《OpenGL ES 应用开发实践指南》](https://image.glumes.com/images/2019/04/27/opengles2_eng.png)

简单上手了 OpenGL ES 2.0 之后，该了解一下 OpenGL Shading Language （GLSL）了。

GLSL 就是着色器脚本语言，这个语言是用来给 GPU 运行的，灵活地使用它才能更好地掌握 OpenGL ES，要知道现在手机相机上的一些滤镜效果都是通过 GLSL 来实现的哦。

>《OpenGL® Shading Language, Second Edition》

![《OpenGL® Shading Language, Second Edition》](https://image.glumes.com/images/2019/04/27/glsl2.jpg)

这本书是英文版的，讲解了 GLSL 的一些语法，基于的版本是 OpenGL ES 2.0 的，正好和前面的书籍配套学习了，而且英文难度不大，易懂。

该书中同样有很多例子可以实践，比如光照、阴影、噪音等。

通过这两本书的配套练习，可以掌握 OpenGL ES 2.x 版本的基本内容了。

当然了，除此之外，你还需要更多的练习。

可以参考这本书，获得更多打怪晋级的经验：

![《Android 3D 游戏开发技术宝典》](https://image.glumes.com/images/2019/04/27/opengles_practice.png)

《Android 3D 游戏开发技术宝典》一书中有很多可以在实践中用到的内容，具体内容就等大家自行探索了~~~

## 高阶版本

当然了，学会了 OpenGL ES 2.0 再去看 OpenGL ES 3.x 就容易多了。

这两者在 GLSL 上是有一些变化的，另外 OpenGL ES 3.x 支持的渲染效果更好，而且支持的特性更多。

关于 OpenGL ES 3.x 版本的学习，有如下书籍推荐：

![OpenGL ES 3.x 游戏开发](https://image.glumes.com/images/2019/04/27/opengles3.png)

![OpenGL ES 3.0 编程指南](https://image.glumes.com/images/2019/04/27/opengles33.png)

在 Android 后续系统版本中，都开始使用 Vulkan 来替代 OpenGL 了。

等掌握了 OpenGL ES 之后，下一个就是 Vulkan 了~~~

另外关于书籍推荐，其实大家可以到京东或者当当上搜索一下关键字就知道了，目前市面上关于 OpenGL ES 的书籍也不多，搜来搜去也就是那几本书啦~~~对于其他领域的书籍情况类似...


## 深入理解

当你已经掌握了 OpenGL ES 的大部分内容，并且可以简单的运用他们了，这时候再想去深入理解它们，那就必须要说到 OpenGL ES 学习中的红宝书和蓝宝书了。

红宝书指的是 《OpenGL 编程指南》，目前已经出到了第九版了，蓝宝书指的是《OpenGL 超级宝典》目前已经出到了第五版了。

![红宝书与蓝宝书](https://image.glumes.com/images/2019/04/27/WechatIMG325.jpg)

这两本书就没有前面那么多代码示例了，更多的是讲解一些原理相关的内容，而且也不是特别针对 Android 开发环境来讲的。这两本书更多是还是当做工具书来使用，当某些知识点不清晰时，看看书查漏补缺~~~（反正我是当工具书用了）

> 听说，下雨天，代码和书籍更配哦~

显然，光是看书是不够的，纸上得来终觉浅，绝知此事要躬行。

在 OpenGL ES 开发中，有一些项目是必看的：

*	https://github.com/CyberAgent/android-gpuimage
*	https://github.com/BradLarson/GPUImage2
*	https://github.com/google/grafika

这些项目中可以看到 OpenGL ES 在相机滤镜和视频录制方面的运用~

## 最后

俗话说：

> 独学而无友，则孤陋而寡闻

光是掌握了这些书上的内容还是不够的，更多的是需要交流和讨论，让这些知识在每一次的探讨中变得更加生动灵活，不再是枯燥的代码和理论。

那就赶紧扫码加入我们吧~~

![](https://image.glumes.com/images/2019/04/27/WechatIMG326.jpg)

要是二维码过期了，加微信 ezglumes 好友，备注 OpenGL ，拉你入群~

本文中提到的书籍资源，皆可在我的 Github 地址上下载得到：

[https://github.com/glumes/AndroidOpenGLTutorial](https://github.com/glumes/AndroidOpenGLTutorial)

一些 OpenGL ES 学习的代码也会在这里，求 Star 一波 。