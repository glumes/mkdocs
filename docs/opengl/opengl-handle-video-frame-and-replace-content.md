---
title: "用 OpenGL 对视频帧内容进行替换"
date: 2018-07-12T16:59:47+08:00
subtitle: ""
draft: false
categories: ["OpenGL"]
tags: ["OpenGL"]
toc: true
 
original: true
addwechat: true
slug: "opengl-handle-video-frame-and-replace-content"
---

> 避免图片丢失，建议阅读微信原文：
> 
> https://mp.weixin.qq.com/s/WZWY5Ml9YP9N9tv-iiFOqg

在群里面有人提到了这么一个实现：现有一段素材视频，想要对视频中的某个内容进行替换，换成自己的图片，这个怎么用 OpenGL 去实现呢？

<!--more-->

首先要明确的是，视频是由一帧一帧图像组成的，它利用了人眼的视觉暂留效应，一秒内播放足够帧数的图片才会感觉到是连续的。

而想要对视频的内容进行替换，也就是要将每一帧图像的内容都进行替换了，一般来说这应该是属于视频后期处理了，用专业的 AE （Adobe After Effects）软件来处理会比较好。

## 处理思路

如果用 OpenGL 来处理，有这样的一个思路：

首先通过 MediaCodec 对每一帧图像内容进行解码，然后再通过 OpenGL 对当前解码的一帧图像进行处理，在原图像上加一个透明的遮罩层，遮罩层的要求就是对于要替换的内容区域是非透明的，其他区域透明，将遮罩层和原图像进行融合，最后得到的就是一帧被替换过内容图像了，再将处理过的一帧图像进行编码，重新编码成新的视频内容。

一直重复 解码 -> 处理 -> 编码这个过程，直到视频的每一帧内容都处理完了，就实现了对视频内容替换。

当然这仅仅是个思路，难点在于如何找到合适的遮罩层，如果视频图像内容是变动的，要替换的内容不是固定的，那么对于遮罩层要求更高了，每一帧处理都得有个合适的遮罩。

下面会针对视频的一帧图像内容进行处理，如何将一帧的图像内容替换了。

## 直接效果

效果如下：


![Sketch 设计图](https://image.glumes.com/images/2019/04/27/frame_replace.png)

代码实现的效果，左上方的内容被右上方内容替换了，最后成了右下角的图片。

![软件实现图](https://image.glumes.com/images/2019/04/27/WechatIMG57.jpg)


## 准备工作

> 不会做设计的开发不是好码农

是时候掏出我的大宝石软件 Sketch 切个图了：

准备一张待替换内容：

![待替换图片](https://image.glumes.com/images/2019/04/27/replace_origin.png)


然后再切一张同等大小，并把中间圆形位置的图片替换成想要的图片，其他周边内容设置透明度为 0 。

![带透明度的遮罩图](https://image.glumes.com/images/2019/04/27/replace_shape.png)

接下来的事情就是将两张图片融合，分别介绍基于着色器和颜色混合来替换内容。

> 这两个方案都有一个共同点，就是要将带遮罩的图片覆盖在原图上，不同的是如何处理两个图片之间的覆盖，透明度就是一个比较好的切入点。

## 使用着色器进行替换


在 OpenGL 的渲染管线中，会先构建图形，然后进行光栅化，光栅化后对每一个片元着色，在这个着色过程中可以根据需要对片元进行处理，包括抛弃某些片元等，简单说在 OpenGL 中就是先有形后有色，而在有形有色的过程中可以搞点小操作~~

对片元进行处理就是我们的片元着色器脚本了。

```glsl
precision mediump float;
varying vec2 vTextureCoord; //接收从顶点着色器过来的参数
uniform sampler2D sTexture;//纹理内容数据
void main() { 
   vec4 bcolor = texture2D(sTexture, vTextureCoord);//给此片元从纹理中采样出颜色值 
   if(bcolor.a<0.6) {
   		discard;
   } else {
      gl_FragColor=bcolor;
}}
```
我们的遮罩图除了要替换的内容，其他地方都是透明的，根据采样出的透明度值小于阈值，就抛弃该片元，直接就不显示了。

而透明度满足要求的就会显示，并且在最后映射到视口上时，直接覆盖了原有的颜色。

通过这种方式就实现了内容替换。

![使用着色器进行替换](https://res.cloudinary.com/glumes-com/image/upload/v1531384608/code/glsl_replace_content.gif)


## 使用颜色混合进行替换

使用颜色混合的方式不像着色器那样简单粗暴，要么抛弃某些片元，要么直接覆盖了。

它是根据一定的计算规则，来计算两个颜色之间的融合。

在 OpenGL 中使用颜色混合要设置合理的混合因子。

```java
        glEnable(GL_BLEND);
        glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA)
        // 绘制
        glDisable(GL_BLEND)
```

混合因子的设置使得如果遮罩图是透明的，使用被遮罩图的颜色，如果不是透明的，使用遮罩图的颜色，这样就不是直接抛弃某些片元了。

效果如下：


![使用颜色混合进行替换](https://res.cloudinary.com/glumes-com/image/upload/v1531384608/code/blend_replace_content.gif)

## 代码实现

在具体的代码实现中，采用了 EGL 来实现离屏的渲染。

在非主线程中，初始化 EGL 环境，然后准备好绘制的必要工作，接着执行绘制，最后把绘制的结果通过 glReadPixels 读取出来。

```kotlin
        Observable.fromCallable {
        // 初始化 EGL 环境
            return@fromCallable initEgl()
        }.map {
        // 设置各种矩阵
            prepare(width, height)
            return@map it
        }.map {
        // 执行绘制
            replaceContent(isBlend)
            return@map it
        }.map {
        // 读取像素
            val result = readPixel(width, height)
            it.destroy()
            return@map result
        }.subscribeOn(Schedulers.computation())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe({
                // 设置效果
                    mResultImage.setImageBitmap(it)
                }, {
                    showToast("replace failed")
                })
```


具体的绘制过程比较简单，如果采用了颜色混合就执行颜色混合的绘制，否则采用着色器的绘制，也体现了就是将遮罩图直接覆盖在原图上的思想。

```kotlin
 private fun replaceContent(isBlend: Boolean) {
        glClearColor(1f, 1f, 1f, 1f)
        glClear(GL_COLOR_BUFFER_BIT or GL_DEPTH_BUFFER_BIT)
        mOriginImage?.drawSelf(mOriginTextureId)
        if (isBlend) {
            glEnable(GL_BLEND);
            glBlendFunc(GL_SRC_ALPHA, GL_ONE_MINUS_SRC_ALPHA)
            mReplaceImage?.drawSelf(mReplaceTextureId)
            glDisable(GL_BLEND)
        } else {
            mAlphaTextureRect?.drawSelf(mReplaceTextureId)
        }
    }
```

在最后读取像素内容时要注意，glReadPixels 读取的内容是上下颠倒的，需要将它翻转过来。

```glsl
   for (i in 0 until height) {
            for (j in 0 until width) {
                pixelMirroredArray[(height - i - 1) * width + j] = pixelArray[i * width + j]
            }
        }
```

具体的实现可以参考我的 Github 项目，求一波 Star 。

> [https://github.com/glumes/AndroidOpenGLTutorial](https://github.com/glumes/AndroidOpenGLTutorial)


## 后续想法

对于视频内容替换，这里仅仅是给出了一帧图像内容的替换，而且还是基于透明度的。

看到好莱坞有些电影场景拍摄时，后面都会给出一块纯色的幕布，然后在后期处理时把幕布内容替换成背景，这种替换通过着色器比较颜色的范围应该也是可以实现的。

当然了，要是搭配图像识别来替换内容玩法就更加丰富了。


## 最后

俗话说：

> 独学而无友，则孤陋而寡闻

学习的路上是需要交流和分享的，这里有个二维码，如果你对 OpenGL ES 感兴趣欢迎一起来交流讨论。

![加群讨论](https://image.glumes.com/images/2019/04/27/WechatIMG432.jpg)

要是二维码过期了，加微信 ezglume 好友，备注 OpenGL ，拉你入群~


