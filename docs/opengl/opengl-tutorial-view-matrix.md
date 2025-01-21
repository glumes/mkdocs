---
title: "OpenGL 学习系列之观察矩阵"
date: 2018-05-22T12:04:10+08:00
subtitle: ""
draft: false
categories: ["OpenGL"]
tags: ["OpenGL","Camera"]
 
toc: true
original: true
addwechat: true
slug: "opengl-tutorial-view-matrix"
---

> 避免图片丢失，建议阅读微信原文：
> 
> https://mp.weixin.qq.com/s/0cWh3IF_7wa5PHoGsh8ZcQ

在 [OpenGL 投影矩阵](https://glumes.com/post/opengl/opengl-tutorial-projection-matrix/) 这篇文章中，讲述了 OpenGL 坐标系统中的投影矩阵，有两种类型的投影矩阵，分别是正交投影和透视投影。

这两种投影实质上是两种类型的裁剪空间，分别创建对应视景体对物体坐标进行裁剪，位于裁剪空间内的才会被映射到屏幕上，如下图所示：（图片来源：[https://glumpy.github.io/modern-gl.html](https://glumpy.github.io/modern-gl.html)）

![](https://image.glumes.com/images/2019/04/27/ViewFrustum_gvy6aq.png)


当定义裁剪空间视景体时，我们都需要提供近平面和远平面的距离，这里的近和远都是指相对于`视点`的，视点也就是我们这篇文章要讲到的摄像机。

<!--more-->

在上面的图片中，我们可以把投影矩阵的视景体的四条虚线边看成是以摄像机为起始点发出的射线。

这样一来，当起始点也就是摄像机位置发生改变时，它所发出的射线也会随之改变，那么视景体的形状也就改变了，在其内部所观察到的内容也会发生变化。

比如，假设此时摄像机位于彩色小球的上面，那么整个视景体就是垂直立起来了的，所观察到的小球也是朝上的那一面了。

所以，可以看到相机的位置和朝向，决定了视景体在什么位置和什么朝向展开。


在 OpenGL 坐标系统的转换公式中也可以印证这一点：

<div>$$ V_{clip}=M_{projection} \cdot M_{view} \cdot M_{model} \cdot V_{local} $$</div>

它的计算顺序是左乘，也就是说要先进行视图矩阵的计算，然后再进行投影矩阵的计算，这样一来我们就要先确定了相机的位置，然后再根据相机确定投影矩阵。

## 确定摄像机位置


在这里，确定相机的位置，并不仅仅是定义相机在三维中的 $(x,y,z)$坐标，而是要确定一个以相机位置为原点的坐标系。

在上面也提到，投影矩阵或者说视景体的一个展开，是以相机作为参考的，那么我们肯定还需要一个摄像机的观察方向，这个方向就是视景体展开的方向。

![camera](https://image.glumes.com/images/2019/04/27/camera_axes.png)

如上图的左二内容所示，摄像机在 Z 轴正方向向坐标系的原点进行观察，假设此时摄像机坐标为 $A(-1,1,1)$，而原点为 $O(0,0,0)$，那么观察方向就是从 $A$ 点向 $O$ 点。而方向向量就是 $A-O$，就是向量  $ \overrightarrow {OA}(-1,1,1)$，它的方向也就是图二中的蓝色箭头所示，可以看到 摄像机的方向向量和它的观察方向正好是相反的。

一个三维的空间坐标系是需要三个互相垂直的轴的，现在已经有了方向向量这一个了。这时可以借助一个辅助向量 上向量  $\overrightarrow{UP}$，把上向量与方向向量进行叉乘，$ \overrightarrow {UP} \cdot \overrightarrow{OA}$，就可以得到一个向量，同时垂直于上向量和方向向量，它就是右向量  $\overrightarrow{Right}$ ，它的方向指向 $X$ 轴正方向 。这里要小心叉乘的顺序，否则得到的方向就是反的了。

如图三所以，灰色的就是辅助上向量 $\overrightarrow{UP}$，而红色箭头所指方向就是 $X$ 轴正方向。

再利用右向量和方向向量的叉乘，就可以得到指向摄像机 $Y$ 轴方向的向量，如最右图的绿色箭头所示。

这样就构造了三个轴互相垂直的坐标系，它就是摄像机的坐标系。

## Matrix.lookAt 函数

从上面的内容可以看到，只要知道了相机坐标点，以及观察的点，还有辅助的上向量，就可以确定摄像机的坐标系了。

确定摄像机之后，就是用它来生成我们的观察矩阵，把观察矩阵用于在 OpenGL 渲染管线中进行处理。

和投影矩阵一样，Android 也提供了对应函数 ` Matrix.setLookAtM` 来生成 OpenGL 坐标转换中的观察矩阵。

```java
	 /**
     * Defines a viewing transformation in terms of an eye point, a center of
     * view, and an up vector.
     *
     * @param rm returns the result
     * @param rmOffset index into rm where the result matrix starts
     * @param eyeX eye point X
     * @param eyeY eye point Y
     * @param eyeZ eye point Z
     * @param centerX center of view X
     * @param centerY center of view Y
     * @param centerZ center of view Z
     * @param upX up vector X
     * @param upY up vector Y
     * @param upZ up vector Z
     */
    public static void setLookAtM(float[] rm, int rmOffset,
            float eyeX, float eyeY, float eyeZ,
            float centerX, float centerY, float centerZ, float upX, float upY,
            float upZ)
```

其中，第一个参数就是要传入的观察矩阵，第二个参数是偏移量，这个一般是  0 ，之后就是相机位置、观察点、辅助上向量。


## 移动相机观察内容

接下来通过移动摄像机来观察物体，从而加深对摄像机的理解。

### 旋转移动相机

用 OpenGL 来绘制一个立方体，并通过旋转移动摄像机，让摄像机绕 $Y$ 轴做圆形旋转，从而可以从不同方向来观察物体，效果图如下：

![rotate-camera](https://res.cloudinary.com/glumes-com/image/upload/v1526832824/code/rotate_camera_with_cube.gif)

让立方体稍微向 $Z$ 轴做一点倾斜，这样最多就可以观察到三个面了。


具体代码示例：

```java
    var num = 0
    var RotateNum  = 360 // 绕 Y 轴做圆形旋转，把圆分成 360 份
    val radian = (2 * Math.PI / RotateNum).toFloat() // 每一份对应的弧度
	override fun onSurfaceChanged(gl: GL10?, width: Int, height: Int) {
		//... 省略代码
		// 通过 RxJava 的 interval 操作符，每隔 30 毫秒，移动相机观察角度
        Observable.interval(30, TimeUnit.MILLISECONDS)
                .subscribe {
                    eyeX = eyeDistance * Math.sin((radian * num).toDouble()).toFloat()
                    eyeZ = eyeDistance * Math.cos((radian * num).toDouble()).toFloat()
                    num++
                    if (num > 360) {
                        num = 0
                    }
                }
		// 设置观察矩阵
	    MatrixState.setCamera(eyeX, eyeY, eyeZ, lookX, lookY, lookZ, upX, upY, upZ)		
	    // 让物体稍微向 Z 轴负方向倾斜一定角度
	    MatrixState.rotate(-30f, 0f, 0f, 1f)
	}
```
在 GLSurfaceView 的 onSurfaceChanged 方法里面设定观察矩阵，并通过 RxJava 的 interval 操作符每隔 30 毫秒改变相机位置的 $X$ 坐标和 $Z$ 坐标，让摄像机在 $XOY$ 平面上绕 $Y$ 轴做圆周运动。

在 onDrawFrame 方法里，每当坐标改变了，就改变相机的位置。

```java
    override fun onDrawFrame(gl: GL10?) {
		//... 省略代码
        MatrixState.setCamera(eyeX, eyeY, eyeZ, lookX, lookY, lookZ, upX, upY, upZ)
        GLES20.glUniformMatrix4fv(uViewMatrixAttr, 1, false, MatrixState.getVMatrix(), 0)
        //... 省略代码
    }
```

由于是做圆周运动，圆的半径是没有变的，所以看到的物体大小是不变的，只是看到的内容不同。


### 前后移动相机

我们还可以前后移动相机，这样就相当于人走近或者离开物体，感觉到物体大小发生变化（实际上并没有）。

![rotate-camera](https://res.cloudinary.com/glumes-com/image/upload/v1526959206/code/rotate-camera-front-and-back.gif)

如上图，物体还是那个物体，但是从不同的远近来观察，所看到的大小就不一样了。

```java
    override fun onSurfaceChanged(gl: GL10?, width: Int, height: Int) {
        super.onSurfaceChanged(gl, width, height)
        var isPlus = true
        var distance = eyeZ
        Observable.interval(100, TimeUnit.MILLISECONDS)
                .subscribe {
                    distance = if (isPlus) distance + 0.1f else distance - 0.1f
                    if (distance < 2.0f) {
                        isPlus = true
                    }
                    if (distance > 5.0f) {
                        isPlus = false
                    }
                    eyeZ = distance
                    updateCamera()
                }
        // 将物体调整一下，可以看到三个面
        MatrixState.rotate(-45f, 0f, 1f, 0f)
        MatrixState.rotate(45f, 1f, 0f, 0f)
    }
```

如上代码所示，我们改变了相机的 $Z$ 轴坐标，让它在 $[2.0,5.0]$ 之间来回移动，这样就达到了前后移动相机的效果。

最后，还可以把两种旋转结合起来，即做圆周运动又前后移动相机，效果如下：

<embed src="https://res.cloudinary.com/glumes-com/video/upload/v1526960805/code/rotate-camera-total.mp4" autostart="false" height="500" width="500" />

## 小结

通过上面的例子，就应该对 OpenGL 中的相机有一个更加清晰的认识了。

具体代码详情，可以参考我的 Github 项目：


[https://github.com/glumes/AndroidOpenGLTutorial](https://github.com/glumes/AndroidOpenGLTutorial)