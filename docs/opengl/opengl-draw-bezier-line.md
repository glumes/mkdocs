---
title: "OpenGL 实践之贝塞尔曲线绘制"
date: 2019-08-20T23:51:59+08:00
subtitle: ""
tags: ["oepngl","bezier"]
categories: ["OpenGL"]
 
toc: true
draft: false
original: true
slug: "opengl-draw-bezier-line"
author: "glumes"
---

> 避免图片丢失，建议阅读微信原文：
> 
> https://mp.weixin.qq.com/s/m7BMF1dSiJ5BeHTU1etDqw


说到贝塞尔曲线，大家肯定都不陌生，网上有很多关于介绍和理解贝塞尔曲线的优秀文章和动态图。

以下两个是比较经典的动图了。

二阶贝塞尔曲线：

![](https://yalantis.com/uploads/ckeditor/pictures/502/content_image04.gif)

三阶贝塞尔曲线：

![](https://yalantis.com/uploads/ckeditor/pictures/503/content_image06.gif)

---

由于在工作中经常要和贝塞尔曲线打交道，所以简单说一下自己的理解：

<!--more-->

现在假设我们要在坐标系中绘制一条直线，直线的方程很简单，就是 `y=x` ，很容易得到下图：



![](https://image.glumes.com/blog_image/20211128231610.png)

现在我们限制一下 x 的取值范围为 `0~1` 的闭区间，那么可以得出 y 的取值范围也是 `0~1`。

而在 `0~1` 的区间范围内，x 能取的数有多少个呢？答案当然是无数个了。


![](https://image.glumes.com/blog_image/20211128231635.png)

同理，y 的取值个数也是有无数个。每一个 x 都有唯一的 y 与之对应，一个 (x,y) 在坐标系上就是一个点。

所以最终得到的 `0~1` 区间的线段，实际上是由无数的点组成的。


那么这条线段有多长呢？长度是由 x 的取值范围来决定的，若 x 的取值为 `0~2`，那么线段就长了一倍。

另外，如果 x 的取值范围不是无数个，而是以 0.05 的间距从 0 到 1 之间递增，那么得到的就是一串点了。

> 由于 点 是一个理想状态下的描述，在数学上点是没有宽高、没有面积的。
> 
> 但是，如果你在草稿纸上绘制一个点，不管你用到是铅笔、毛笔、水笔还是画笔，一个点总是要占面积的。
> 
> 毛笔画一个点的面积可能需要铅笔画几十个点了。


在实际生活中，如果要以 0.05 的间距在第一幅坐标系图中画出 x 在 `0~1` 区间的一串点，最终结果就和直接画一条线段没啥差别了。

这就是现实和理想的差别了。理想一串点，现实一条线。

---

我们把这个逻辑放到手机屏幕上。

手机屏幕上的最小显示单位就是像素了，一个 1920 * 1080 的屏幕指的就是各方向上像素点的数量。

假如绘制一条和屏幕一样宽的线段，一个点最小就算一个像素，最多也就 1080 个点了。

点占的像素越多，那么实际绘制时需要的点的数量越少，这也算是潜在的优化项了。

---


说完直线，再回到贝塞尔曲线上。

曲线和直线都有一个共同点，它们都有各自特定的方程，只不过我们用的直线例子比较简单，既 `y = x` ，一眼看出计算结果。

直线方程 y = x，在数学上可以这么描述：y 是关于 x 的函数，既 `y = F(x)` ，其中 x 的取值决定了该直线的长度。

根据上面的理解，这个长度的直线实际又是由在 x 的取值范围内对应的无数个点组成的。

反观贝塞尔曲线方程以及对应的图形如下：

* 二阶贝塞尔曲线：


![](https://image.glumes.com/blog_image/20211128231733.png)
![](https://yalantis.com/uploads/ckeditor/pictures/1961/content_content_image00.png)

其中，P0 和 P2 是起始点，P1 是控制点。

* 三阶贝塞尔曲线

![](https://image.glumes.com/blog_image/20211128231752.png)
![](https://yalantis.com/uploads/ckeditor/pictures/1962/content_content_image09.png)


其中，P0 和 P3 是起始点，P1 和 P2 是控制点。

---

不难理解，假设我们要绘制一条曲线，肯定要有起始和结束点来指定曲线的范围曲线。

而控制点就是指定该曲线的弧度，或者说指定该曲线的弯曲走向，不同的控制点得出的曲线绘制结果是不一样的。

另外，可以观察到，无论是几阶贝塞尔曲线，都会有参数 t 以及 t 的取值范围限定。

t 在 0~1 范围的闭区间内，那么 t 的取值个数实际上就有无数个了，这时的 t 就可以理解成上面介绍直线中讲到的 x 。

这样一来，就可以把起始点、控制点当初固定参数，那么贝塞尔曲线计算公式就成了 B = F(t) ，B 是关于 t 的函数，而 t 的取值范围为 0~1 的闭区间。

> 也就是说贝塞尔曲线，选定了起始点和控制点，照样可以看成是 t 在 0~1 闭区间内对应的无数个点所组成的。


有了上面的阐述，在工(ban)程(zhuan)的角度上，就不难理解贝塞尔曲线到底怎么使用了。

---

## Android 绘制贝塞尔曲线

Android 自带贝塞尔曲线绘制 API ，通过 Path 类的 `quadTo` 和 `cubicTo` 方法就可以完成绘制。  


```java
  // 构建 path 路径，也就是选取
  path.reset();
  path.moveTo(p0x, p0y);
  // 绘制二阶贝塞尔曲线
  path.quadTo(p1x, p1y, p2x, p2y);
  path.moveTo(p0x, p0y);
  path.close();
  
  // 最后的绘制操作
  canvas.drawPath(path, paint);
```

这里的绘制实际上就是把贝塞尔曲线计算的方程式交给了 Android 系统内部去完成了，参数传递上只传递了起始点和控制点。

我们可以通过自己的代码来计算这个方程式从而对逻辑上获得更多控制权，也就是把曲线拆分成许多个点组成，如果点的尺寸比较大，甚至可以减少点的个数实现同样的效果，达到绘制优化的目的。

## OpenGL 绘制

通过 OpenGL 可以实现我们上述的方案，把曲线拆分成多个点组成。这种方案要求我们在 CPU 上去计算贝塞尔曲线方程，根据 t 的每一个取值，计算出一个贝塞尔点，用 OpenGL 去绘制上这个点。

这个点的绘制可以采用 OpenGL 中画三角形 `GL_TRIANGLES` 的形式去绘制，这样就可以给点带上纹理效果，不过这里面的坑略多，起始点和控制点都是运行时动态可变的实现难度会大于固定不变的。

这里先介绍另一种方案，这种方案实现比较简单也能达到优化效果，我们可以把贝塞尔曲线的计算方程式交给 GPU， 在 OpenGL Shader 中去完成。

这样一来，我们只要给定起始点和控制点，中间计算贝塞尔曲线去填补点的过程就交给 Shader 去完成了。

另外，通过控制 t 的数量，我们可以控制贝塞尔点填补的疏密。

t 越大，填补的点越多，超过一定阈值后，不会对绘制效果有提升，反而影响性能。

t 越小，那么贝塞尔曲线就退化成一串点组成了。所以说 t 的取值范围也能对绘制起到优化作用。

绘制效果如下图所示：



![](https://image.glumes.com/blog_image/3ogfg403e8.gif)

以下就是实际的代码部分了，关于 OpenGL 的基础理论部分可以参考之前写过的文章和公众号，就不再阐述了。

在 Shader 中定义一个函数，实现贝塞尔方程：

```cpp
vec2 fun(in vec2 p0, in vec2 p1, in vec2 p2, in vec2 p3, in float t){
    float tt = (1.0 - t) * (1.0 -t);
    return tt * (1.0 -t) *p0 
            + 3.0 * t * tt * p1 
            + 3.0 * t *t *(1.0 -t) *p2 
            + t *t *t *p3;
}
```

该方程可以利用 Shader 中自带的函数优化一波：

```cpp
vec2 fun2(in vec2 p0, in vec2 p1, in vec2 p2, in vec2 p3, in float t)
{
    vec2 q0 = mix(p0, p1, t);
    vec2 q1 = mix(p1, p2, t);
    vec2 q2 = mix(p2, p3, t);
    vec2 r0 = mix(q0, q1, t);
    vec2 r1 = mix(q1, q2, t);
    return mix(r0, r1, t);
}
```

接下来就是具体的顶点着色器 shader ：

```cpp
// 对应 t 数据的传递
attribute float aData;
// 对应起始点和结束点
uniform vec4 uStartEndData;
// 对应控制点
uniform vec4 uControlData;
// mvp 矩阵
uniform mat4 u_MVPMatrix;

void main() {
    vec4 pos;
    pos.w = 1.0;
    // 取出起始点、结束点、控制点
    vec2 p0 = uStartEndData.xy;
    vec2 p3 = uStartEndData.zw;
    vec2 p1 = uControlData.xy;
    vec2 p2 = uControlData.zw;
    // 取出 t 的值
    float t = aData;
    // 计算贝塞尔点的函数调用
    vec2 point = fun2(p0, p1, p2, p3, t);
    // 定义点的 x,y 坐标
    pos.xy = point;
    // 要绘制的位置
    gl_Position = u_MVPMatrix * pos;
    // 定义点的尺寸大小
    gl_PointSize = 20.0;
}
```

代码中的 `uStartEndData` 对应起始点和结束点，`uControlData` 对应两个控制点。

这两个变量的数据传递通过 `glUniform4f` 方法就好了：

```cpp
    mStartEndHandle = glGetUniformLocation(mProgram, "uStartEndData");
    mControlHandle = glGetUniformLocation(mProgram, "uControlData");
    // 传递数据，作为固定值
    glUniform4f(mStartEndHandle,
            mStartEndPoints[0],
            mStartEndPoints[1],
            mStartEndPoints[2],
            mStartEndPoints[3]);
    glUniform4f(mControlHandle,
            mControlPoints[0],
            mControlPoints[1],
            mControlPoints[2],
            mControlPoints[3]);  
            
```

另外重要的变量就是 `aData` 了，它对应的就是 t 在 0~1 闭区间的划分的数量。

```java
    private float[] genTData() {
        float[] tData = new float[Const.NUM_POINTS];
        for (int i = 0; i < tData.length; i ++) {
            float t = (float) i / (float) tData.length;
            tData[i] = t;
        }
        return tData;
    }
```

以上函数就是把 t 在 0~1 闭区间分成 `Const.NUM_POINTS` 份，每一份的值都存在 tData 数组中，最后通过 `glVertexAttribPointer` 函数传递给 Shader 。

最后实际绘制时，我们采用 `GL_POINTS` 的形式绘制就好了。

```java
        GLES20.glDrawArrays(GLES20.GL_POINTS, 0, Const.NUM_POINTS );
```

以上就是 OpenGL 绘制贝塞尔曲线的小实践。

具体的代码部分可以参考我的项目：

> https://github.com/glumes/AndroidOpenGLTutorial


在参考中，也有一个 OpenGL 绘制贝塞尔曲线的例子，不过他绘制的是贝塞尔曲线面，采用的是 `GL_TRIANGLES` 的形式，而且在 `tData` 数组的构造也有些不同，但是都大同小异了，看明白了本文的例子也不难理解参考的文章。

## 参考

1. https://yalantis.com/blog/how-we-created-visualization-for-horizon-our-open-source-library-for-sound-visualization/
