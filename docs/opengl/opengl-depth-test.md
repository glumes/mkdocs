---
title: "OpenGL 深度测试与精度值的那些事"
date: 2018-07-20T14:50:22+08:00
subtitle: ""
draft: false
categories: ["OpenGL"]
tags: ["OpenGL"]
toc: true
 
slug: "opengl-depth-test"
---

> 避免图片丢失，建议阅读微信原文：
> 
> https://mp.weixin.qq.com/s/PpLzxm7hPHDbaQLmAniCzQ

在 OpenGL 世界里，使用深度测试可以来防止被阻挡的面渲染到其他面的前面。

直接看一个没有使用深度测试的绘制：

![未开启深度测试的情况](https://res.cloudinary.com/glumes-com/image/upload/v1532052034/code/undepth_test.gif)



按照计划是绘制一个封闭的立方体，六个面都是有的，可从上面的效果来看并不是，立方体的有些面丢失了，只有后面的那个面，前面的面没了。

这就是在没有开启深度测试的情况下，本来应该被遮挡的，绘制在后面的面却绘制到了其他面之上。


要解决这种问题，就得使用深度测试了。

<!--more-->


值得一提的是：在没有开启深度测试的情况下，假设绘制了多个不同远近的物体，那么对于最后的景象来说，哪怕是距离最远的，只要它的最后绘制的，都会显示在景象的前面。



当深度测试被启用时，OpenGL 会将一个片段的深度值与深度缓冲的内容进行对比。OpenGL 会执行一个深度测试，如果这个测试通过了的话，深度缓冲将会更新为新的深度值，如果深度测试失败了，该片段将会被丢弃。



深度缓冲是在片段着色器运行之后，在屏幕空间中运行的。屏幕空间坐标与通过 OpenGL 的 glViewport 所定义的视口密切相关，并且可以通过 GLSL 的内建变量 `gl_FragCoord` 从片段着色器中直接访问。


`gl_FragCoord` 的 x 和 y 分量代表了片段的屏幕空间坐标（其（0，0）位于左下角）。`gl_FragCoord` 中也包含了一个 z 分量，它包含了片段真正的深度值。z 值就是需要与深度缓冲内容所对比的那个汁。

深度缓冲默认是禁止的，通过如下代码开启它：

```java
glEnable(GL_DEPTH_TEST);
```

开启之后，如果一个片段通过了深度测试的话，OpenGL 就会在深度缓冲中存储该片段的 z 值；如果没有通过深度缓冲，则丢弃该片段。

如果开启了深度缓冲，就应该在每个渲染迭代之前，也就是 onDrawFrame 方法中清除深度缓冲，否则就仍在使用上一次渲染迭代时写入的深度值。

```java
// 清除深度缓冲
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
```

如果在某些情况下，需要对所有的片段执行深度测试并丢弃某些片段，但是又不希望深度缓冲被更新，这时就需要使用一个`只读深度缓冲`。

OpenGL 运行我们禁用深度缓冲的写入，只需要设置它的深度掩码为 `GL_FALSE` 即可。

```java
// 设置只读的深度缓冲
glDepthMask(GL_FALSE);
```


## 深度测试函数

OpenGL 允许修改深度测试中使用的比较运算符，允许我们控制 OpenGL 什么时候该通过或丢弃一个片段，什么时候更新深度缓冲。

调用 `glDepthFunc` 函数设置比较运算符。

```java
glDepthFunc(GL_LESS);
```

|函数|描述|
|---|---|
|GL_ALWAYS|永远通过深度测试|
|GL_NEVER|永远不通过深度测试|
|GL_LESS|在片段深度值小于缓冲的深度值时通过测试|
|GL_EQUAL|在片段深度值等于缓冲区的深度值时通过测试|
|GL_LEQUAL|在片段深度值小于等于缓冲区的深度值时通过测试|
|GL_GREATER|在片段深度值大于缓冲区的深度值时通过测试|
|GL_NOTEQUAL|在片段深度值不等于缓冲区的深度值时通过测试|
|GL_GEQUAL|在片段深度值大于等于缓冲区的深度值时通过测试|

默认情况下使用的是 `GL_LESS`，它将丢弃深度值大于当前深度缓冲值的所有片段。

对于如下代码的组合：

```java
glEnable(GL_DEPTH_TEST);
glDepthFunc(GL_ALWAYS);
```

其效果等效于没有开启深度测试。


但我们开启深度测试之后，就可以得到正常的立方体绘制了。



![](https://res.cloudinary.com/glumes-com/image/upload/v1526832824/code/rotate_camera_with_cube.gif) 


## 深度值精度

上面提到的作为比较的深度缓冲，它是位于 0.0 ~ 1.0 之间的深度值，它会与要绘制的物体的 z 值进行比较。

要绘制物体的 z 值就是在运用透视投影或者正交投影视时，介于近平面和远平面之间的任何值。

要把这个 z 值转换为 OpenGL 中的深度值，也就是介于 0.0 和 1.0 之间的值。

有公式如下：


$$\begin{equation} F_{depth} = \frac{z - near}{far - near} \end{equation}$$


对于这种转换，可以看到，当物体非常接近近平面时，深度值会接近 0.0，当物体非常接近远平面时，深度值会接近 1.0 ，这种转换是一种线性的深度缓冲转换。

![](https://image.glumes.com/images/2019/04/27/depth_linear_graph.png)


而在实践中，几乎不可能是这样的线性转换。

因为当 z 值很小的时候，非常接近近平面，此时我们的观察也会更加精细，而对于较远的物体，接近远平面了，对于它的观察也会比较粗略。

这就和人眼一样，近处的物体当然看得很清了，如果看不清，走近一点就好了，而对于很远的物体，走远了和走近了看得的结果差别不大。


所以在实际将 z 值转换为深度缓冲值，用到的是非线性的转换方程。

$$\begin{equation} F_{depth} = \frac{1/z - 1/near}{1/far - 1/near} \end{equation}$$


它的效果如下：

![](https://image.glumes.com/images/2019/04/27/depth_non_linear_graph.png)

可以看到在 z 值位于 1.0 和 2.0 之间时，对应的深度值为 0.0 到 0.5 的区间，这就占据了深度值区间范围的 50 %。而 2.0 之后的范围也才占据了 50 %。

这就给了近处的物体一个很大的深度精度。
 
 对于深度值的区间 0.0 到 1.0 ，其实这个区间的前半部分还是和近平面非常近的，不要以为深度值 0.5 就是位于近平面和远平面之间了，其实还非常接近近平面呢。



关于深度测试，就先说到这了，如果有绘制带有深度层次的内容，可别忘了开启深度测试哦。

关于具体的代码实现，可以参考我的 Github 项目：

> [https://github.com/glumes/AndroidOpenGLTutorial](https://github.com/glumes/AndroidOpenGLTutorial)


## 参考

1. https://learnopengl-cn.github.io/04%20Advanced%20OpenGL/01%20Depth%20testing/


