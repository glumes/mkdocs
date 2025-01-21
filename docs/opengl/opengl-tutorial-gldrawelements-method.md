---
title: "OpenGL 的 glDrawElements  绘制方法"
date: 2018-05-25T19:47:29+08:00
subtitle: ""
draft: false
categories: ["OpenGL"]
tags: ["OpenGL"]
 
toc: true
original: true
addwechat: true
slug: "opengl-tutorial-gldrawelements-method"
---

> 避免图片丢失，建议阅读微信原文：
> 
> https://mp.weixin.qq.com/s/WcWdYE5j8Ycw2dtJYS-Cxg


在之前的绘制中，我们都是通过 `glDrawArrays` 方法来实现的，它会按照我们传入的顶点顺序和指定的绘制方式进行绘制。


回顾一下之前提到的绘制类型：

|绘制类型|绘制方式|
|---|----|
|GL_POINTS|将传入的顶点坐标作为单独的点绘制|
|GL_LINES|将传入的坐标作为单独线条绘制，ABCDEFG六个顶点，绘制AB、CD、EF三条线|
|GL_LINE_STRIP|将传入的顶点作为折线绘制，ABCD四个顶点，绘制AB、BC、CD三条线|
|GL_LINE_LOOP|将传入的顶点作为闭合折线绘制，ABCD四个顶点，绘制AB、BC、CD、DA四条线。|
|GL_TRIANGLES|将传入的顶点作为单独的三角形绘制，ABCDEF绘制ABC,DEF两个三角形|
|GL_TRIANGLE_STRIP|将传入的顶点作为三角条带绘制，ABCDEF绘制ABC,BCD,CDE,DEF四个三角形|
|GL_TRIANGLE_FAN|将传入的顶点作为扇面绘制，ABCDEF绘制ABC、ACD、ADE、AEF四个三角形|

<!--more-->

假设要绘制一个立方体，以 `GL_TRIANGLES` 的类型进行绘制，那么六个面，每个面由两个三角形组成，就得向渲染管线传入 36 个顶点，36 个顶点按照顺序进行绘制，而实际上，一个矩形也就才 8 个顶点而已。

为了优化绘制的效率，减少数据的传递，于是就有了 `glDrawElements` 绘制方法。


## glDrawElements 绘制方法

`glDrawElements` 方法还是需要传递顶点数据，但只需要传递物体实际上的顶点数据，也就是最少的，不重复的顶点数据。

然后再向渲染管线传递要绘制的顶点数据的索引，根据索引从顶点数据中取出对应的顶点，然后再按照指定的方式进行绘制。

如下图所示，图片截自《OpenGL ES 3.x 游戏开发上卷》：

![](https://image.glumes.com/images/2019/04/27/glDrawElements_single.png)


由三个三角形组成的倒置的梯形，实际上只有五个顶点 $ \{v0,v1,v2,v3,v4\}$，因此也只传递了五个顶点，接下来就是确定这个五个顶点的索引顺序。

索引顺序和我们要绘制的方式有很大的关系，不同绘制方式的索引顺序不同。

采用 `GL_TRIANGLE_STRIP` 的类型绘制，那么索引顺序就是 $\{0,1,2,3,4\}$。

具体方法调用情况代码：

```java
	// 函数原型
    public static native void glDrawElements(
        int mode, // 绘制方式
        int count, // 绘制数量
        int type, // 索引的数据类型
        java.nio.Buffer indices // 索引缓冲
    );
	// 定义顶点的索引数据
    private byte[] front = {
            // 前面索引
            0, 1, 2, 3
    };
    // 索引数据传递到缓冲区
    private ByteBuffer frontBuffer = ByteBuffer.allocateDirect(indices.length *Constants.BYTES_PRE_BYTE)
									 .put(indices)
	frontBuffer.position(0)								 
	// glDrawElements 绘制
	GLES20.glDrawElements(GLES20.GL_TRIANGLE_STRIP,front.size,GLES20.GL_UNSIGNED_BYTE, frontBuffer)
	// glDrawArrays 绘制
	// GLES20.glDrawArrays(GLES20.GL_TRIANGLE_STRIP, 0, 4)
```

在函数原型中定义了要传入的参数，根据要绘制的方法和索引缓冲区，找到对应的点进行绘制。

对于绘制时有重复点，采用这种方式，就可以减少向渲染管线中传递重复的点了。

而且，在定义一个顶点时，大都是 `float` 类型，它是四个字节，而对于绘制量比较小，顶点数量在 `byte` 所能表达整数范围内，可以采用 `byte` 类型定义索引顺序，它只占一个字节，减少了内存的使用。


## glDrawElements 和 glDrawArrays 的对比

`glDrawElements` 方法的 `count` 的参数定义了要取多少个索引出来绘制，而且这个绘制是连续的，必须要把 `count` 数量的顶点绘制完。

这里就有一个很有意思的地方了，有一些小游戏是这样的：要求一笔绘制完一个形状，而且不允许交叉。

比如，要求一笔绕过立方体的六个面，而且不允许交叉，这就很难做到的了。

而使用 `glDrawElements` 方法同样会这样，采用索引不能一次不交叉的把图形全部绘制完，得采取两次绘制。

比如在实践绘制一个立方体时，采用了如下的方式：

```java
	// 前面索引
    private byte[] front = {0, 1, 2, 3};
    // 后面索引
    private byte[] back = { 4, 5, 6, 7};
	// 上面索引
    private byte[] top = {0, 1, 4, 5};
    // 下面索引
    private byte[] bottom = { 2, 3, 6, 7};
    // 左面索引
    private byte[] left = {0, 4, 2, 6};
    // 右侧索引
    private byte[] right = {1, 5, 3, 7};
```

把立方体分解成了六个面的内容进行绘制，也就是采用了六个索引缓存。

而对于使用 `glDrawArrays`的方式，可以一次性把所有顶点传到渲染管线，并且可以选择绘制的开始和结尾点，这样就只要一个缓冲区就好了，不过代码就是要多占用内存空间了。

所以说，能用 glDrawElements 方式的还是要采用的。

具体代码详情，可以参考我的 Github 项目：

https://github.com/glumes/AndroidOpenGLTutorial


