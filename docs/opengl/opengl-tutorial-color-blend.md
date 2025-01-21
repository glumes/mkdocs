---
title: "《OpenGL ES 3.x 游戏开发》之颜色混合和使用"
date: 2018-07-16T11:13:19+08:00
subtitle: "学习笔记内容摘录"
draft: false
categories: ["OpenGL"]
tags: ["OpenGL"]
toc: true
 
slug: "opengl-tutorial-color-blend"
---

> 避免图片丢失，建议阅读微信原文：
> 
> https://mp.weixin.qq.com/s/40ss1fbLh3Qr5X4o76GTjA

在 Android 中有一个类 PorterDuffXfermode ，它是用来设置颜色混合方式的，也就是在已有颜色的基础上再绘制一笔颜色，这两个颜色是如何进行混合的，是新绘制的颜色覆盖了原有颜色，还是新绘制的颜色和原有颜色混合组成另一种颜色呢。


在 OpenGL 中同样有这样颜色混合的问题。

<!--more-->

在 OpenGL 的世界模型中是有深度的概念的，也就是由 z 轴坐标值来决定物体距离坐标原地的远近，但到最后世界模型里的物体都要投影到近平面，最后映射到视口上。而且，距离相机也就是视口越近的物体，就会遮住后面的物体，就和用肉眼去观察物体一下，后面的形状会被前面的形状挡住。


但和肉眼观察不同的是，OpenGL 里最终呈现的颜色，是将两个片元混合之后计算的值，我们可以改变这片元混合的方式，这就和前面 Android 里面提到的 PorterDuffXfermode 混合模式一样。

## 颜色混合基础知识

OpenGL 中的颜色混合就是将通过各种测试准备进入帧缓冲的片元（源片元）与帧缓冲中的原有片元（目标片元）按照设定的比例加权计算最终片元的颜色值。新片元不一定是直接覆盖缓冲区中的源片元。

### 混合因子


OpenGL 通过设置混合因子来指定两个片元的加权比例，每次都需要给出两个混合因子：

*	源因子，用于确定将进入帧缓冲的片元在最终片元中的比例
*	目标因子，用于确定原帧缓冲中的片元在最终片元中的比例


由于 OpenGL 中每个颜色值包括 4 个色彩通道，因此，两种混子因子都有 4 个分量值，分别对应一个色彩通道，具体混合计算细节如下：

*	设源因子和目标因子分别为 $[S_r,S_g,S_b,S_a]$ 和 $[D_r,D_g,D_b,D_a]$ ，$S$ 表示是源因子，$D$ 表示是目标因子，r，g，b，a 下标分别表示 红、绿、蓝、透明度 4 个色彩通道。
*	设源片元和目标片元的颜色值分别为  $[R_s,G_s,B_s,A_s]$  和 $[R_d,G_d,B_d,A_d]$，R，G，B，A 分别表示红、绿、蓝、透明度 4 个色彩通道，s 下标表示源片元，d 下标表示目标片元。
*	混合后最终片元颜色各个色彩通道的值是由颜色混合方程式计算而来，系统提供的常用颜色混合方程式如下：


### 混合方程式

|方程式名|最终片元颜色各个色彩通道的值|
|---|---|
|GL_FUNC_ADD|$[R_sS_r+R_dD_r, G_sS_g+G_dD_g, B_sS_b+B_dD_b, A_sS_a+A_dD_a]$|
|GL_FUNC_SUBTRACT|$[R_sS_r-R_dD_r, G_sS_g-G_dD_g, B_sS_b-B_dD_b, A_sS_a-A_dD_a]$|
|GL_FUNC_REVERSE_SUBTRACT|$[R_dD_r-R_sS_r, G_dD_g-G_sS_g, B_dD_b-B_sS_b, A_dD_a-A_sS_a]$|
|GL_MIN|$[min(R_sS_r,R_dD_r),min(G_sS_g,G_dD_g),min(B_sS_b,B_dD_b),min(A_sS_a,A_dD_a)]$|
|GL_MAX|$[max(R_sS_r,R_dD_r),max(G_sS_g,G_dD_g),max(B_sS_b,B_dD_b),max(A_sS_a,A_dD_a)]$|


默认情况下，系统会自动调用 GL_FUNC_ADD 混合方程式来计算最终片元颜色各个色彩通道的值，如果想要调用其他混合方程式来计算最终的片元颜色，系统也有提供对应的方法：

|方法签名|说明|
|---|---|
|glBlendEquation(int mode)|mode 参数的含义为指定混合方程式，用来计算最终片元的颜色，其值为混合方程式的名字|
|glBlendEquationSeparate(int modeRGB,int modeAlpha)|modeRGB 参数为颜色的 RGB 通道进行混合时所使用的混合方程式名，modeAlpha 参数的含义是颜色的 Alpha 透明度通道进行混合时所使用的混合方程式名字，通过其可以实现 RGB 和 Alpha 通道单独指定混合方程式的功能|

### 源因子和目标因子

对于颜色混合来说，选定了混合方程式，接下来最重要的就是设置混合因子了。

在 OpenGL 中预置了一些混合因子，如下表：

|常量名|RGB 混合因子| A 混合因子|
|---|---|---|
|GL_ZERO|$[0,0,0]$|0|
|GL_ONE|$[1,1,1]$|1|
|GL_SRC_COLOR|$[R_s,G_s,B_s]$|$A_s$|
|GL_ONE_MINUS_SRC_COLOR|$[1-R_s,1-G_s,1-B_s]$|$1-As$|
|GL_DST_COLOR|$[R_d,G_d,B_d]$|$A_d$|
|GL_ONE_MINUS_DST_COLOR|$[1- R_d, 1- G_d, 1- B_d]$|$1-A_d$|
|GL_SRC_ALPHA|$[A_s, A_s, A_s]$|$A_s$|
|GL_ONE_MINUS_SRC_ ALPHA|$[1- A_s, 1- A_s, 1- A_s]$|$1-As$|
|GL_DST_ ALPHA|$[A_d, A_d, A_d]$|$A_d$|
|GL_ONE_MINUS_DST_ ALPHA|$[1- A_d, 1- A_d, 1- A_d]$|$1-A_d$|
|GL_SRC_ALPHA_SATURATE|$[f , f, f] f=min(A_s, 1-A_d)$|1|
|GL_CONSTANT_COLOR|$[R_c, G_c, B_c]$|$A_c$|
|GL_ONE_MINUS_CONSTANT_COLOR|$[1- R_c, 1- G_c, 1- B_c]$|$1-A_c$|
|GL_CONSTANT_ALPHA|$[Ac, Ac, Ac]$|$A_c$|
|GL_ONE_MINUS_CONSTANT_ALPHA|$[1- Ac, 1- Ac, 1- Ac]$|$1- Ac$|

其中，常量名称中有 SRC 代表各通道值来自源片元，有 DST 代表各通道值来自目标片元，另外 GL_SRC_ALPHA_SATURATE 只能用作源因子。


对于常量名中有 CONSTANT 的代表使用预设颜色常量值对应色彩通道的值作为相应的因子值，其中的 $R_c$、$G_c$、$B_c$、$A_c$ 分别代表预设颜色常量值的 RGBA 通道的值，如果没有设置则默认值为 $[0.0f、0.0f、0.0f、0.0f]$ 。

设置颜色常量值使用的是 glBlendColor 方法，如下：

```java
    public static native void glBlendColor(
        float red,
        float green,
        float blue,
        float alpha
    );
```


### 设置混合的方法

有了混合因子，接下来就是设置混合因子了，OpenGL 提供了如下方法来设置：

|方法签名|说明|
|---|---|
|glBlendFunc(GLenum sfactor, GLenum dfactor)|第一个参数为设置的源因子，第二个参数为设置的目标因子，其值如上表|
|glBlendFuncSeparate(GLenum srcRGB, GLenum dstRGB, GLenum srcAlpha,GLenum dstAlpha)|第一个参数为设置源因子的 RGB 值；第二个参数为设置目标因子 RGB 值；第三个参数为设置源因子 Alpha 的值；第四个参数为设置目标因子 Alpha 的值。该方法实现了 RGB 和 Alpha 通道单独指定混合因子值的功能|


### 常用混合组合

对于混合因子和混合 方程式的组合太多了，恰当的组合可以产生很好的效果，下面给出两组常用的组合：

*	源因子 GL_SRC_ALPHA ，目标因子 GL_ONE_MINUS_SRC_ALPHA ，即源因子和目标因子分别为 $[A_s, A_s, A_s, A_s]$ 和 $[1-A_s, 1-A_s, 1-A_s, 1-A_s]$ 。此组合实现的是最典型的半透明遮挡效果。若源片元是透明的，则根据透明度透过后面的内容；若源片元不透明，则仅能看到源片元，因此，使用此组合时往往会采用半透明的纹理或颜色对源片元着色。


*	源因子 GL_SRC_COLOR ，目标因子 GL_ONE_MINUS_SRC_COLOR ，即源因子和目标因子分别为 $[R_s, G_s, B_s, A_s]$ 和 $[1- R_s, 1- G_s, 1- B_s, 1-A_s$ 。此组合可以实现滤光镜效果，也就是平时透过有色眼镜或玻璃观察事物的感觉。与第一种常用组合不同，此组合不要求应用于源片元的颜色或者纹理是半透明的。



## 具体使用

> 前面讲了这么多理论，其实就是阐述两个颜色的 RGBA 值如何计算得到最后的 RGBA 值，并且每一个 R、G、B、A 分量都是两个颜色的 R、G、B、A 对应乘以不同的混合因子后相加得到的，这个混合因子的设置可以根据源片元的颜色来设定，也可以根据目标片元的颜色来设定。


先假设有这样的场景：

![](https://image.glumes.com/images/2019/04/27/blend_scene.jpg)

通过这样的图片经过混合后去查看后面的内容，类似于刺激战场上开镜效果，

![](https://image.glumes.com/images/2019/04/27/lgq.png)

显然，图片的黑色区域是要可以透过看到后面内容的，绿色区域也是一样，不然就全遮盖住后面内容了。

这里就可以用到上面的混合因子组合，源因子 GL_SRC_COLOR，目标因子 GL_ONE_MINUS_SRC_COLOR ，这个混合因子是根据源片元的颜色来设定的。

根据这两个混合因子和混合方程式计算，可以得出最后的颜色值。

根据源因子 GL_SRC_COLOR 计算，由于黑色的 RGB 值都为 0 ，RGB 混合因子都为 0，也就是透明的，源片元不会进入到最终片元。

根据源因子 GL_ONE_MINUS_SRC_COLOR 计算，由于黑色的 RGB 值都为 0 ，那么目标颜色的混合因子都为 1，也就是目标颜色都会被显示，可以看到后面的物体。

对于绿色部分，就不存在为0 或者为 1 的情况了，就是两个颜色按照混合因子计算后的值了。

具体的使用过程很简单，大致代码如下：

```java
			// 先绘制好背景内容 
			// 开启颜色混合进行绘制
            GLES20.glEnable(GLES20.GL_BLEND)
	        // 设置混合因子
            GLES20.glBlendFunc(GLES20.GL_SRC_COLOR, GLES20.GL_ONE_MINUS_SRC_COLOR)
			// 绘制操作
            rect!!.drawSelf(textureId)
            MatrixState.popMatrix()
```

最后效果如下：

![](https://image.glumes.com/images/2019/04/27/blend_src_color.jpg)



当然，还可以使用另外一种混合因子组合 GL_SRC_ALPHA 和 GL_ONE_MINUS_SRC_ALPHA，根据源因子的透明度来设置混合因子。

关于如何使用 GL_SRC_ALPHA 和 GL_ONE_MINUS_SRC_ALPHA 混合因子，可以参考之前的文章 [用 OpenGL 对视频帧内容进行替换](https://glumes.com/post/opengl/opengl-handle-video-frame-and-replace-content/)，大概原理都一样的，就是图片换成带透明度的，并且更改一下混合因子组合，就不赘述了。


具体的实现可以参考我的 Github 项目，求一波 Star 。

> [https://github.com/glumes/AndroidOpenGLTutorial](https://github.com/glumes/AndroidOpenGLTutorial)


## 参考

1. 《OpenGL ES 3.x 游戏开发》

## 最后

俗话说：

> 独学而无友，则孤陋而寡闻

学习的路上是需要交流和分享的，这里有个二维码，如果你对 OpenGL ES 感兴趣欢迎一起来交流讨论。

![加群讨论](https://image.glumes.com/images/2019/04/27/WechatIMG434.jpg)

要是二维码过期了，加微信 ezglume 好友，备注 OpenGL ，拉你入群~

