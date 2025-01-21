---
title: "《OpenGL ES 3.x 游戏开发》之利用 Alpha 透明度进行测试"
date: 2018-07-13T09:41:22+08:00
subtitle: "学习笔记内容摘录"
draft: false
categories: ["OpenGL"]
tags: ["OpenGL"]
toc: true
 
slug: "opengl-tutorial-alpha-test"
---



在前面的博客文章中有提到 [OpenGL 裁剪测试及注意点](https://glumes.com/post/opengl/opengl-tutorial-scissor-test/)，并且裁剪测试只能裁剪一个矩形区域，相当于就是把整个内容都绘制上去了，但是透过一个小矩形区域来看绘制的物体。

除了透过矩形区域，还可以实现透过任意形状区域来观察物体，这就是要用到 OpenGL 的 Alpha 透明度测试。

关于 Alpha 透明度测试，在 用 [OpenGL 对视频帧内容进行替换](https://glumes.com/post/opengl/opengl-handle-video-frame-and-replace-content/) 也用实践用到过。

<!--more-->

在 OpenGL 中，每个颜色都有四个色彩通道 --- RGBA，其中 A 就是 Alpha 透明度通道，可以利用它的值进行检测。

Alpha 测试的基本原理为，当绘制一个片元时，首先检测其 Alpha 值，若 Alpha 值满足要求，就通过测试，绘制此片元，否则丢弃此片元，不进行绘制。

比如如下的场景：

![](https://image.glumes.com/images/2019/04/27/WechatIMG27.jpg)

如果是裁剪测试，只能屏幕上透过一个矩形区域来观察实际场景。

![](https://image.glumes.com/images/2019/04/27/WechatIMG26.jpg)


而换成 Alpha 测试，就可以实现透过任意形状的观察，效果如下：

![](https://image.glumes.com/images/2019/04/27/WechatIMG28.jpg)


图上的内容实际由三部分组成，树、地形、中间带透明度的边框。

正因为绘制的边框带透明度，才不会全部遮住后面的内容。

具体绘制代码如下：
```java
            GLES20.glClear(GLES20.GL_DEPTH_BUFFER_BIT or GLES20.GL_COLOR_BUFFER_BIT)
            // 打开深度检测
             GLES20.glEnable(GLES20.GL_DEPTH_TEST)
			// 设置相机和观察矩阵，绘制地形和树
            MatrixState.setProjectFrustum(-ratio!!, ratio!!, -1f, 1f, 1f, 100f)
            MatrixState.setCamera(cx, 0f, cz, 0f, 0f, 0f, 0f, 1.0f, 0f)

            MatrixState.pushMatrix()
            MatrixState.translate(0f, -2f, 0f)
            // 绘制地形
            desert!!.drawSelf(desertId)
            MatrixState.popMatrix()
			// 开启混合，因为树要和地形有混合的地方
            GLES20.glEnable(GLES20.GL_BLEND)
			// 设置混合因子
            GLES20.glBlendFunc(GLES20.GL_SRC_ALPHA, GLES20.GL_ONE_MINUS_SRC_ALPHA)
            MatrixState.pushMatrix()
            MatrixState.translate(0f, -2f, 0f)
            // 绘制树
            tg!!.drawSelf(treeId)
            MatrixState.popMatrix()
			// 关闭混合
            GLES20.glDisable(GLES20.GL_BLEND)
			// 清除深度缓冲
            GLES20.glClear(GLES20.GL_DEPTH_BUFFER_BIT)
			// 设置相机和投影矩阵来绘制带透明度的边框
            MatrixState.pushMatrix()
            MatrixState.setProjectOrtho(-1f, 1f, -1f, 1f, 1f, 100f)
            MatrixState.setCamera(0f, 0f, 3f, 0f, 0f, 0f, 0f, 1.0f, 0.0f)
            rect!!.drawSelf(maskTextureId)
            MatrixState.popMatrix()
```


这里面绘制带透明度的边框时清除了一次深度缓冲，因为带透明度边框不属于主场景，不应该和主场景中的物体以同一标准进行深度检测，否则可能会产生不正确的视觉效果。


## 具体实现

Alpha 测试的重点在于片段着色器的处理，在这里可以对每一个片段进行处理，根据 Alpha 来决定是否抛弃它。

```glsl
precision mediump float;
varying vec2 vTextureCoord; //接收从顶点着色器过来的参数
uniform sampler2D sTexture;//纹理内容数据
void main() { 
	//给此片元从纹理中采样出颜色值 
   vec4 bcolor = texture2D(sTexture, vTextureCoord);
   if(bcolor.a<0.6) {
   		discard;
   } else {
      gl_FragColor=bcolor;
}}
```

根据具体的图片透明度，这里，如果透明度小于 0.6 ，就抛弃这个片元，否则的话就赋值给原本采样的颜色。

其中，在 GLSL 着色器语言中，`discard` 代表放弃该片元。

而 texture2D 方法代表纹理采样，它的结果类型是一个向量，向量在着色器代码中的地位非常重要，可以方便地存储以及操作 **颜色**、**位置**、**纹理坐标** 等不仅包含一个组成部分的量。

对于一个向量，有时需要单独访问向量中的某个分量，基本语法为 `<向量名>.<分量名>`，从不同的角度来看，它的每个分量代表的含义是不同的：

*	将向量看作颜色时，可以使用 r、g、b、a 四个分量名，分别代表红、绿、蓝、透明度 4 个色彩通道。

```glsl
vec4 aColor;
aColor.r = 0.6;
aColor.g = 0.8;
```
若向量是 4 维的，可以使用的分量名为 r、g、b、a；若向量是 3 维的，可以使用的分量名为 r、g、b；若是 2 维的，可以使用的 r、g 两个分量名。

*	将向量看作位置时，可以使用 x、y、z、w 四个分量名，分别代表 x 轴、y 轴、z 轴分量及 w 值。
```glsl
vec4 aPosition;
aPosition.x = 34.2
aPosition.z = 12.2
```
若向量是 4 维的，可以使用的分量名为 x、y、z、w；若向量是 3 维的，可以使用的分量名为 x、y、z；若是 2 维的，可以使用的 x、y 两个分量名。

*	将向量看作纹理坐标时，可以使用 s、t、p、q 四个分量名，分别代表纹理坐标的不同分量。
```glsl
vec4 aTexCoor;
aTexCoor.s = 0.45;
aTexCoor.t = 0.22;
```
若向量是 4 维的，可以使用的分量名为 s、t、p、q；若向量是 3 维的，可以使用的分量名为 s、t、p；若是 2 维的，可以使用的 s、t 两个分量名。

除了使用 `.` 的形式来访问分量名，还可以将向量看作一个数组，用下标来进行访问：

```glsl
aColor[0] = 0.4;
aPostion[2] = 0.34;
aTexCoorp[1] = 0.12;
```

理解了 discard 的作用和向量各分量的含义，就很容易明白 Alpha 测试的原理了。

实际上，Alpha 测试并不是渲染管线上要进行的操作流程，而是我们自己想出的根据 Alpha 透明度值对片元进行处理罢了。


具体的实现可以参考我的 Github 项目，求一波 Star 。

> [https://github.com/glumes/AndroidOpenGLTutorial](https://github.com/glumes/AndroidOpenGLTutorial)


## 参考

1. 《OpenGL ES 3.x 游戏开发》

## 最后

俗话说：

> 独学而无友，则孤陋而寡闻

学习的路上是需要交流和分享的，这里有个二维码，如果你对 OpenGL ES 感兴趣欢迎一起来交流讨论。

![加群讨论](https://image.glumes.com/images/2019/04/27/WechatIMG432.jpg)

要是二维码过期了，加微信 ezglume 好友，备注 OpenGL ，拉你入群~

