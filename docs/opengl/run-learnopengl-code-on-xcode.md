---
title: "LearnOpenGL 源码在 MAC 上的编译与调试"
date: 2020-01-04T22:53:52+08:00
subtitle: ""
slug: "opengl/run-learnopengl-code-on-xcode"
tags: ["opengl"]
categories: ["OpenGL"]
 
toc: true
draft: false
original: true
slug: "run-learnopengl-code-on-xcode"
---

> 避免图片丢失，建议阅读微信原文：
> 
> https://mp.weixin.qq.com/s/Ah8bK4dELT-LDjwSk9dHiQ

学习 OpenGL ，相信肯定有不少人看过这个网站：

> https://learnopengl.com/



![](https://image.glumes.com/blog_image/20211128104709.png)

这是它的英文原版网站，后来又有了不同语言的翻译版本，对应中文就是：

> https://learnopengl-cn.github.io/



![](https://image.glumes.com/blog_image/20211128105236.png)

这两个网站对于学习 OpenGL 帮助非常大，既可以用作入门的教材，也可以作为工具书，后续进行查漏补缺。


并且它的内容很全面，除了 OpenGL 基础知识、坐标系统、纹理、Shader、模型加载等，还有高级光照、PBR 等渲染技巧，这些在渲染引擎的开发中都是会用到的，后面会继续和大家分享。


<!--more-->

---

本文主要是讲解如何运行 LearnOpenGL 文章中的示例代码，在 XCode 上进行编译和调试，效果如下：



![](https://image.glumes.com/blog_image/20211128105254.png)

在网站上某一章节的内容，就对应于 XCode 工程某一小项的具体代码，我们可以选择要运行的章节代码，在 Mac 看到最终效果。




![](https://image.glumes.com/blog_image/20211128105313.png)

> 另外，我们还可以在 XCode 上修改相关代码，调整某些参数，验证自己的想法和实验结果。


这一点很重要，对于初学者来说就是要不断地试错，在失败中成长。


在开始 LearnOpenGL 网站的代码讲解之前，先介绍一下他的主人。



![](https://image.glumes.com/blog_image/20211128105326.png)

不得不说，这是位大佬，有兴趣的同学都可以去 Follow 一下。

他的个人主页是：

> https://joeydevries.com

从主页上可以看到大佬在图形学和游戏开发上造诣颇深，做了不少有意思的东西。

* 实现了一个简单的渲染引擎，采用 C++ 开发，支持了不少特性，在 LearnOpenGL 网站上都可以看到。

> https://github.com/JoeyDeVries/Cell

这可以当做是学习 OpenGL 之后巩固提高的一个大作业了。

* 另外，还做了一个 Vulkan 的教学网站，虽说目前还没有完成，但仍然值得期待。

> https://learnvulkan.com/

* 当然了，你也可以看我的网站，或许对你有一些帮助

> https://glumes.com/

---

言归正传，讲回代码的编译部分。

LearnOpenGL 网站的示例代码地址如下，clone 这个项目到你的电脑上。

> https://github.com/JoeyDeVries/LearnOpenGL

在这个项目的 `README.md` 上已经有讲如何在 MAC 平台进行编译了。

```sh
brew install cmake assimp glm glfw
mkdir build
cd build
cmake ../.
make -j8
```

但是这个编译结果并不是我们想要的，因为它编译出来的都是二进制可执行文件。



![](https://image.glumes.com/blog_image/20211128105342.png)

虽说我们可以通过 `./xxx` 的方式来运行这些可执行文件，但总不能每改一点代码就全都编译一次吧。

理想的方式就要通过 IDE（集成开发环境） 来编译运行，并且在 IDE 上修改代码，看到结果。


XCode 在这里就充当了 IDE 的角色。

下面给出新的编译代码：

```sh
brew install cmake assimp glm glfw
mkdir build
cd build
cmake -G "Xcode" ..
```

有变化的就是最后一行了，此行代码会在 build 目录下生成 XCode 工程。



![](https://image.glumes.com/blog_image/20211128105402.png)

双击 `LearnOpenGL.xcodeproj` 就可以打开整个工程啦。

---

接下来就是自由发挥时间，你可以在源代码基础上进行任何修改，对照着 LearnOpenGL 网站上的讲解，一步一步地去调试验证，积累经验，在成为大佬的路上越走越远~~~~

> 祝玩得愉快 😀😀😀
