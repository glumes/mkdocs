---
title: "OpenGL 学习系列之投影矩阵"
date: 2018-01-24T18:17:58+08:00
subtitle: ""
tags: ["OpenGL"]
draft: false
categories: ["OpenGL"]
 
toc: true
original: true
addwechat: true
slug: "opengl-tutorial-projection-matrix"
---

> 避免图片丢失，建议阅读微信原文：
> 
> https://mp.weixin.qq.com/s/AVyNmsO7s6rGPZw3WH5kcA

在 [OpenGL 坐标系统](https://glumes.com/post/opengl/opengl-tutorial-coordinate/) 文章中，根据点的坐标变换得出了如下的公式：

![](https://image.glumes.com/blog_image/20211128232151.png)

这个公式每左乘一个矩阵，都代表了一种坐标系的变换。

转化为着色器脚本语言如下：

``` glsl
attribute vec4 a_Position;
uniform mat4 u_ModelMatrix;
uniform mat4 u_ProjectionMatrix;
uniform mat4 u_ViewMatrix;
void main()
{
    gl_Position  = u_ProjectionMatrix * u_ViewMatrix * u_ModelMatrix * a_Position;
}
```

本篇文章就主要是对投影矩阵来分析的。

<!--more-->

OpenGL 在观察空间转换到裁剪空间时，需要用到投影矩阵。而在着色器脚本中，也需要提供一个投影矩阵给对应的 `u_ProjectionMatrix`变量。

首先要在程序里绑定到对应的变量，然后再给变量赋值。

``` java
// 绑定到着色器脚本中的对应变量
private static final String U_ProMatrix = "u_ProjectionMatrix";
private int uProMatrixLocation;
uProMatrixLocation = glGetUniformLocation(mProgram,U_ProMatrix);
// 给变量赋值，projectionMatrix 为投影矩阵
glUniformMatrix4fv(uProMatrixLocation,1,false,projectionMatrix,0)
```

正如前文讲到的，投影矩阵会创建一个视景体对物体坐标进行裁剪，得到的裁剪坐标再经过透视除法之后，就会得到归一化设备坐标。归一化设备坐标再经过视口转换，最终将坐标映射到了屏幕上。

OpenGL 提供了两种投影方式：正交投影和透视投影。

## 正交投影矩阵



![](https://image.glumes.com/blog_image/20211128232214.png)

不管是正交投影还是透视投影，最终都是将视景体内的物体投影在近平面上，这也是 3D 坐标转换到 2D 坐标的关键一步。

而近平面上的坐标接着也会转换成归一化设备坐标，再映射到屏幕视口上。

为了解决之前的图像拉伸问题，就是要保证近平面的宽高比和视口的宽高比一致，而且是以较短的那一边作为 1 的标准，让图像保持居中。


OpenGL 提供了 `Matrix.orthoM` 函数来生成正交投影矩阵。

``` java
    /**
     * Computes an orthographic projection matrix.
     *
     * @param m returns the result 正交投影矩阵
     * @param mOffset 偏移量，默认为 0 ,不偏移
     * @param left 左平面距离
     * @param right 右平面距离
     * @param bottom 下平面距离
     * @param top 上平面距离
     * @param near 近平面距离
     * @param far 远平面距离
     */
    public static void orthoM(float[] m, int mOffset,
        float left, float right, float bottom, float top,
        float near, float far)
```



![](https://image.glumes.com/blog_image/20211128232236.png)
需要注意的是，我们的左、上、右、下距离都是相对于近平面中心的。

近平面的坐标原点位于中心，向右为 $X$ 轴正方向，向上为 $Y$ 轴正方向，所以我们的 left、bottom 要为负数，而 right、top 要为正数。同时，近平面和远平面的距离都是指相对于视点的距离，所以 near、far 要为正数，而且 $far > near$。

可以在 GLSurfaceView 的 surfaceChanged 里面来设定正交投影矩阵。

``` java
  @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        float aspectRatio = width > height ? (float) width / (float) height : (float) height / (float) width;
        if (width > height){
            Matrix.orthoM(projectionMatrix,0,-aspectRatio,aspectRatio,-1f,1f,0f,10f);
        }else {
            Matrix.orthoM(projectionMatrix,0,-1f,1f,-aspectRatio,aspectRatio,0f,10f);
        }
    }
```

这样的话，就把近平面的宽高比设定与视口的宽高比一致了。解决了之前绘制的图像被拉伸的问题。


## 透视投影矩阵

OpenGL 提供了两个函数来创建透视投影矩阵：`frustumM` 和 `perspectiveM`。

### frustumM

frustumM 函数创建的视景体是一个锥形。




![](https://image.glumes.com/blog_image/20211128232257.png)
它的视景体有点类似于正交投影，在参数理解上基本都相同的。

``` java
/**
     * Defines a projection matrix in terms of six clip planes.
     *
     * @param m the float array that holds the output perspective matrix
     * @param offset the offset into float array m where the perspective
     *        matrix data is written
     * @param left 
     * @param right
     * @param bottom
     * @param top
     * @param near
     * @param far
     */
    public static void frustumM(float[] m, int offset,
            float left, float right, float bottom, float top,
            float near, float far)
```


需要注意的是 near 和 far 变量的值必须要大于 0 。因为它们都是相对于视点的距离，也就是照相机的距离。

当用视图矩阵确定了照相机的位置时，要确保物体距离视点的位置在 near 和 far  的区间范围内，否则就会看不到物体。

由于透视投影会产生近大远小的效果，当照相机位置不变，改变 near 的值时也会改变物体大小，near 越小，则离视点越近，相当于物体越远，那么显示的物体也就越小了。

当然也可以 near 和 far 的距离不动，改变摄像机的位置来改变观察到的物体大小。


### perspectiveM



![](https://image.glumes.com/blog_image/20211128232318.png)
OpenGL 还提供了  `perspectiveM` 函数来创建投影矩阵，它的视景体和 `frustumM` 函数相同，但是构造的参数有所不同。


``` java
 /**
     * Defines a projection matrix in terms of a field of view angle, an
     * aspect ratio, and z clip planes.
     *
     * @param m the float array that holds the perspective matrix
     * @param offset the offset into float array m where the perspective
     *        matrix data is written
     * @param fovy field of view in y direction, in degrees
     * @param aspect width to height aspect ratio of the viewport
     * @param zNear
     * @param zFar
     */
    public static void perspectiveM(float[] m, int offset,
          float fovy, float aspect, float zNear, float zFar)
```

视景体不再需要确定近平面左、上、右、下距离了。

通过视角来决定我们能看到的视野大小。视角就是图中所示的那个夹角。另外的参数是视口的宽高比，还有近平面和远平面的距离，参数个数减少了。





![](https://image.glumes.com/blog_image/20211128232336.png)

![](https://image.glumes.com/blog_image/20211128232355.png)



上述图片左边是 90 视角，右边是 45 度视角。显然，视野角度越大，则看到的内容更多，但是物体显得更小，而视野角度越小，则看的内容更少，但物体显得更大。有点类似于成语 **一叶障目** 的感觉。

和 `frustumM`不同的是，一旦确定了视角和宽高比，那么整个摄像机视野也就确定了，此时完整的锥形视野已经形成了，也就是说物体的近大远小效果已经完成了。这时，近平面距离和远平面距离只是确定想要截取锥形视野中的哪一部分了。不像在`frustumM`函数中，近、远平面的距离还能够调整近大远小的效果。


## 小结

理解了 OpenGL 中的几种投影矩阵，在实际开发调试时就更得心应手了。


## 参考

1. 《OpenGL ES 应用开发实践指南》
2. 《OpenGL ES 3.x 游戏开发》


具体代码详情，可以参考我的 Github 项目：

https://github.com/glumes/AndroidOpenGLTutorial

