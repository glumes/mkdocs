---
title: "《OpenGL ES 3.x 游戏开发》 裁剪测试及注意点"
date: 2018-07-03T21:35:34+08:00
subtitle: "学习笔记内容摘录"
draft: false
categories: ["OpenGL"]
tags: ["OpenGL"]
toc: true
 
slug: "opengl-tutorial-scissor-test"
---


在 OpenGL 中启用裁剪测试可以在屏幕或者帧缓冲上指定一个矩形区域，然后在该矩形区域内绘制，只有在该区域内的片元才有机会最终进入帧缓冲，不在该区域内的将会被丢弃。

<!--more-->

裁剪测试的效果就相当于在屏幕上开辟一个矩形区域，在该区域内再单独绘制内容。

它的主要代码如下：

```java
		// 开启裁剪测试
        GLES20.glEnable(GLES20.GL_SCISSOR_TEST)
        // 指定开辟的矩形区域
        GLES20.glScissor(x,y,width,height)
        // 清除该区域内的颜色
        GLES20.glClearColor(1f, 0f, 0f, 1f)
        GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT or GLES20.GL_DEPTH_BUFFER_BIT)
        // 在矩形局域内绘制内容
        mCube.onDrawFrame(gl)
        // 关闭裁剪测试
        GLES20.glDisable(GLES20.GL_SCISSOR_TEST)
```


它的使用并不难，只要在 onDrawFrame 方法里面开启裁剪测试，然后绘制，然后关闭裁剪就行了，具体效果如下：

![](https://res.cloudinary.com/glumes-com/image/upload/v1530624256/code/opengl-scissor.gif)

在原来的图像上裁剪了一小块矩形，然后绘制了一个旋转的形状。

## 注意点


`glScissor` 方法可以指定裁剪的矩形区域，它的方法原型如下：

```java
    public static native void glScissor(
        int x,
        int y,
        int width,
        int height
    );
```

其中，x 和 y 指矩形区域的左下角坐标，width 和 height 指矩形区域的宽和高。

x 和 y 坐标所在的坐标系是以视口矩形区域的左下角为原点的，x 轴向右，y 轴向上的坐标系，对应到屏幕的坐标，就是以屏幕的左下角为原点，这和 Android 中以左上角为原点不同。


另外，在绘制时我们把视口设置为屏幕的宽和高，那么在启用裁剪测试后，想让绘制的内容正好显示在指定的矩形区域内，就必须保证即便没有启用裁剪测试，内容也是绘制在指定的区域内，否则裁剪区域内将不会显示绘制内容。


也就是说，裁剪测试只是在原来的视口标准的绘制区域内开辟一块矩形区域来显示，而不是把内容放到裁剪的区域内来显示。

最后创建了一个微信群，欢迎加群交流讨论 OpenGL ES 相关技术~~

![](https://image.glumes.com/images/2019/04/27/WechatIMG324.jpg)

具体代码详情，可以参考我的 Github 项目，求一波 star~~

[https://github.com/glumes/AndroidOpenGLTutorial](https://github.com/glumes/AndroidOpenGLTutorial)

## 参考

1. 《OpenGL ES 3.x 游戏开发》

