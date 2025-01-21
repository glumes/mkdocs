---
title: "OpenGL 之 帧缓冲  使用实践"
date: 2018-09-04T22:28:11+08:00
subtitle: ""
tags: ["OpenGL","FBO"]
categories: ["OpenGL"]
toc: true
 
draft: false
original: true
addwechat: true
slug: "opengl-framebuffer-object-usage"
---

> 避免图片丢失，建议阅读微信原文：
> 
> https://mp.weixin.qq.com/s/l5eYzkYAzR-m21-iYBoyCw

帧缓冲(Framebuffer Object)，简称 `FBO`，在渲染绘制中， 图像最终都是绘制到 FBO 上的，一般都是默认的 FBO 上，也就是我们的屏幕。

除此之外，还可以创建自己的 FBO，用来作为绘制的载体，当在自己的 FBO 上绘制好了之后，可以再把绘制内容显示到屏幕上，实现一个双缓冲的绘制。

FBO 实际上是由颜色附件、深度附件、模板附件组成的，作为着色器各方面（一般包括颜色、深度、深度值）绘制结果存储的逻辑对象。

<!--more-->

渲染缓冲(Renderbuffer Object)，简称 `RBO`，由应用程序分配的 2D 图像缓冲区，可以用于分配和存储 **深度** 和 **模板** 值，也可以用作 FBO 的 *深度* 或者 *模板* 附件，另外，纹理也可以作为 FBO 的颜色和深度附件。

帧缓冲与渲染缓冲和纹理的关系如下：

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1fuxlr1hxafj20a2081dg2.jpg)


## 使用概述

帧缓冲的使用，首先就创建对应的帧缓冲对象，然后给它添加对应的附件，比如颜色附件或者深度附件等。

接着就是切换到帧缓冲渲染，在帧缓冲中进行绘制，此时绘制的内容都是记录在上一步添加的颜色附件或者深度附件上了。

然后切换到屏幕的缓冲区，这时可以把帧缓冲中记录的颜色或者深度信息取出来，再把他们绘制到屏幕上。

帧缓冲的使用看似很简单，但是用处却很普遍，使用帧缓冲可以在一些相机应用中做美颜处理、滤镜处理，也可以用来作贴纸等等效果。

## 使用步骤


### 创建 FBO

按照上面的步骤，首先是创建 FBO 。

```java
        int[] framebuffers = new int[1];
        GLES20.glGenFramebuffers(1, framebuffers, 0);
```

和 OpenGL 中创建的大多对象相同，都是通过一个 `int` 型对象来表示的。


接下来就可以给这个 FBO 添加一些附件了。

### 绑定纹理

```java
// 把纹理绑定到颜色附件
GLES20.glFramebufferTexture2D(
    GLES20.GL_FRAMEBUFFER, 
    GLES20.GL_COLOR_ATTACHMENT0, // 作为颜色附件
    GLES20.GL_TEXTURE_2D, 
    fboTextureId, 
    0);
```

通过 `glFramebufferTexture2D` 函数可以将纹理绑定到 FBO 上作为附件，在绑定时有多种附件选项可以选择。

*   GL_COLOR_ATTACHMENT0    
    *   颜色附件
*   GL_DEPTH_ATTACHMENT     
    *   深度附件
*   GL_STENCIL_ATTACHMENT   
    *   模板附件

当然作为纹理，只有颜色和深度两种可以选择。

如果是使用 OpenGL 3.x 版本，在绑定 FBO 时，还可以选择是绑定只读还是只写的 FBO。

*   GL_READ_FRAMEBUFFER     
    *   只读的 FBO
*   GL_DRAW_FRAMEBUFFER     
    *   只写的 FBO 


如果是使用 `GL_FRAMEBUFFER` 的话，那么读写皆可以。

### 绑定渲染缓冲

除了纹理之外，还可以绑定到渲染缓冲。

首先还是创建一个渲染缓冲对象：

```java
        int[] renderbuffers = new int[1];
        GLES20.glGenRenderbuffers(1, renderbuffers, 0);
```


之后，要绑定到当前的渲染缓冲对象，并初始化渲染缓冲对象的数据存储，再把它添加到 FBO 上。
    
```java
// 绑定到当前渲染缓冲对象
GLES20.glBindRenderbuffer(GLES20.GL_RENDERBUFFER, renderbufferId);
// 初始化渲染数据存储
GLES20.glRenderbufferStorage(GLES20.GL_RENDERBUFFER, GLES20.GL_DEPTH_ATTACHMENT, width, height);
// 将渲染缓冲添加到 FBO 上
GLES20.glFramebufferRenderbuffer(GLES20.GL_FRAMEBUFFER, GLES20.GL_DEPTH_ATTACHMENT, GLES20.GL_RENDERBUFFER, renderbufferId);
```


绑定渲染缓冲对象，并且初始化数据存储这一步有点类似于创建纹理并且初始化的操作。

```java
        // 绑定纹理
        GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, textureId);
        // 初始化纹理数据
        GLES20.glTexImage2D(GLES20.GL_TEXTURE_2D, 0, GLES20.GL_RGB, width, height,
                0, GLES20.GL_RGB, GLES20.GL_UNSIGNED_SHORT_5_6_5, null);
```


### 渲染

在渲染时，先渲染到 FBO 上，在渲染到屏幕上。

首先要切换到 FBO 进行渲染：

```java
GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, fboId);
```

如果 `fboId` 参数为 0 ，则代表切换到默认的屏幕上进行绘制。

此时就可以进行正常的绘制，由添加的附件去记录绘制的信息。

```java
// 加载纹理
int textureId = TextureHelper.loadTexture(context, R.drawable.lgq);
// 将纹理绘制到 FBO 上
mTextureRect.drawSelf(textureId);
```

在 FBO 上绘制一张纹理贴图，此时 FBO 所绑定的颜色附件，会记录下纹理贴图的所有颜色内容。

也就是说，FBO 所绑定的纹理作为颜色附件，此时它已经被渲染上了颜色，而这个颜色就是我们绘制的内容，那么接下来就可以使用 FBO 绑定的纹理继续用来绘制。

```java
        // 切换到屏幕的缓冲区
        GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, 0);
        // 使用 FBO 所绑定的纹理进行绘制
        mTextureRect.drawSelf(fboTextureId);
```

切换到屏幕的缓冲区后，直接使用 FBO 绑定的纹理进行绘制，此时看到的效果和未使用 FBO 是相同的。

但是内部的绘制就是完全不一样了。

文章中具体代码部分，可以参考我的 Github 项目，欢迎 Star 。

> [https://github.com/glumes/AndroidOpenGLTutorial](https://github.com/glumes/AndroidOpenGLTutorial)


## 参考

1. https://blog.csdn.net/cauchyweierstrass/article/details/53166940
2. 《OpenGL ES 3.x 游戏开发下卷》