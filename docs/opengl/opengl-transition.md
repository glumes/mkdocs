---
title: "OpenGL 实现视频编辑中的转场效果"
date: 2019-10-20T22:49:00+08:00
subtitle: ""
tags: ["OpenGL"]
categories: ["OpenGL"]
 
toc: true
draft: false
original: true
slug: "opengl-transition"
---

> 避免图片丢失，建议阅读微信原文：
> 
> https://mp.weixin.qq.com/s/OuyU_7ieecjdGyDKTx-iFg

## 转场介绍

转场效果是什么？

转场效果，简单来说就是两段视频之间的衔接过渡效果。

现在拍摄 vlog 的玩家越来越多，要是视频没有一两个炫酷的转场效果，都不好意思拿出来炫酷了。

![](https://res.cloudinary.com/glumes-com/image/upload/v1578201670/transition_hpaktx.gif)

那么如何在视频编辑软件中实现转场效果呢？

这里提供使用 OpenGL 实现视频转场的一个小示例，我们可以通过自定义 GLSL 来实现不同的转场效果。


以在 Android 平台上作为演示，但其实不管是 Android 还是 iOS，实现的原理都是一样的。


首先要有两段视频，视频 A 和视频 B，先播放视频 A 后播放视频 B，中间有一段过程称为 C ，C 就是视频 A、B 做转场动画的时间段。

如下所示：

![transition_model.jpg](https://image.glumes.com/images/2019/10/14/transition_model.jpg)


播放器按照时间顺序，从 A -> C -> B 的播放，这样就有了转场的效果。

<!--more-->

---

视频转场，首先就得有视频，直接从视频 A、B 中解码出当前帧并通过 OpenGL 显示到屏幕上就好了，如果你对这个操作不熟悉的话，可以查看我的公众号【纸上浅谈】历史文章，都有写过相关内容。

这里以图片来替代视频 A、B 中解码出来的帧。

最终效果如下：


[![transition_sample.gif](https://image.glumes.com/images/2019/10/17/transition_sample.gif)](https://image.glumes.com/image/IiWR)


## 实现讲解

### 模拟视频渲染播放

模拟 fps 为 30 的视频，用 RxJava 每间隔 30 ms 就触发一次 OpenGL 渲染。

```java
Observable
        .interval(30, TimeUnit.MILLISECONDS)
        .subscribeOn(AndroidSchedulers.mainThread())
        .subscribe {
            mGLSurfaceView.requestRender()
        }
```

另外，如果在视频 A 播放阶段不断地改变图片，也就是更新纹理内容，就相当于在真实的解码视频进行播放了。

当然这些操作只是为了让这个小例子更加贴近真正的视频转场，重要的还是在于如何实现转场的 Shader 效果。


首先转场的时候要有两个纹理作为输入，那么肯定要定义两个 `sampler2D` 进行采样了。

```cpp
varying vec2 vTextureCoord;//接收从顶点着色器过来的参数
uniform sampler2D sTexture1;
uniform sampler2D sTexture2;
```

其中 `sTexture1` 对应于视频 A 内容，`sTexture2` 对应于视频 B 内容。

`vTextureCoord` 对应于顶点着色器传递过来的纹理坐标，视频 A 和 视频 B 都需要用到这个纹理坐标。

这个时候，只要调用 texture2D 方法就能得到视频 A 和 视频 B 的内容了。

```cpp
// 得到视频 A 的内容
texture2D(sTexture1,vTextureCoord)
// 得到视频 B 的内容
texture2D(sTexture2,vTextureCoord)
```

> 要注意的是这里说得到视频 A/B 的内容，是得到纹理坐标对应的图像内容。也就是说如果纹理坐标是 [0,1] 范围内，那么可以得到视频 A/B 的全部图像内容。如果坐标是 [-0.5,0.5] 那么只能采样得到一半内容了。

### 转场效果实现


#### 混合函数 mix 

由于转场效果是需要视频 A 和视频 B 进行叠加混合的，而 GLSL 内嵌了 `mix` 函数进行调用。

对于 GLSL 中有哪些内嵌的函数可以直接调用的，可以参考写过的文章记录：

> [OpenGL ES 2.0 着色器语言 GLSL 学习](https://glumes.com/post/opengl/opengl-glsl-2-mark/)


`mix` 函数的声明如下：

```cpp
genType mix(genType x,genType y,float a) // 其中 genType 泛指 GLSL 中的类型定义
```

它的主要功能是使用因子 a 对 x 与 y 执行线性混合，既返回 `x * (1-a) + y * a` 。

现在，通过 texture2D 能得到视频帧内容，通过 mix 能进行视频帧混合叠加，那么就可以得到最后转场视频帧了。

```cpp
vec4 color = mix(texture2D(sTexture1,vTextureCoord),texture2D(sTexture2,vTextureCoord),a);
```

#### 渲染进度控制

似乎到这里就可以大功告成了，实际上才刚刚完成了一半~~~

要知道转场效果是随着时间来播放的，就上面的例子中，转场时间内，一开始都是视频 A 的内容，然后视频 A 逐渐减少，视频 B 逐渐增多，到最后全是视频 B 内容，在我们的 Shader 中也要体现这个时间变化的概念。


在 Shader 中定义 `progress` 变量，代表转场的播放进度，进度为 0 ~ 1.0 之间。

```cpp
uniform float progress;
```

同时在每一次渲染时更新 `progress` 变量的值。

```cpp
GLES20.glUniform1f(muProgressHandle, mProgress);
```

当 `progress` 为 0 时代表转场刚刚开始，当 `progress` 为 1 时代表转场已经结束了。

```java
if (mProgress >= 1.0f) {
    mProgress = 0.0f;
} else {
    mProgress += 0.01;
}
```

这里 `progress` 每次递增 0.01，完成一次转场就需要 100 次渲染，每次渲染间隔 30ms，那么一次转场动画就是 3000ms 了，当然这个可以自己调节的。

#### 画面绘制

再回到 `mix` 函数的参数 `a` ，这个参数起到了随时间调节转场混合程度的作用。当 a = 0 时，全是视频 A 的内容， 当 a = 1 时，全是视频 B 的内容。


![transition_frame.jpg](https://image.glumes.com/images/2019/10/17/transition_frame.jpg)


如上图所示，在转场动画的某一帧，左侧是视频 A 的内容，因为此时 a = 0，右侧是视频 B 的内容，此时 a = 1 。

可以看到在一次渲染绘制内 a 既要能等于 0 ，还要能等于 1 ，这个是怎么实现的呢? 


事实上我们说的一次渲染绘制，通常指 OpenGL draw 方法的一次调用，但是在这一次调用里，还是有很多步骤要执行的。

OpenGL 渲染管线会先执行顶点着色器，然后光栅化，再接着就是片段着色器，片段着色器会根据纹理坐标采样纹理贴图上的像素内容进行着色，因此片段着色器在管线中会多次执行，针对每个像素都要进行着色。

![transition_model2.jpg](https://image.glumes.com/images/2019/10/18/transition_model2.jpg)

上面图像的小方块就好比一个像素，每个像素都要执行一个片段着色器。

首先，肯定所有的像素都要进行着色的。左侧方块采样视频 A 的纹理进行着色，右侧方块采样视频 B 的纹理进行着色。

回到如下代码：

```cpp
mix(texture2D(sTexture1,vTextureCoord),texture2D(sTexture2,vTextureCoord),a);
```

只要保证绘制左侧时 a = 0，绘制右侧时 a = 1 就行了。这里可以通过移动纹理坐标来控制 a 的值。

```cpp
vec2 p = vTextureCoord + progress * sign(direction);
float a = step(0.0,p.y) * step(p.y,1.0) * step(0.0,p.x) * step(p.x,1.0);
```

OpenGL 中定义纹理坐标范围是 [0 ~ 1] ，可以将范围右移 0.5 ，从而变成 [0.5 ~ 1.5] ，此时纹理坐标一半位于规定范围内，一半超出界外了。

这样就可以通过对当前像素小方格对应的纹理坐标的 x，y 值运用 `step` 函数进行判断是否在界内，就可以决定是采样视频 A 还是视频 B 的图像了。


当每次刷新 progress 时，就向右移一小段距离，视频 A 随着右移而变少，视频 B 变多，这样就是实现了转场效果。

---

## 联想和总结

不知道这个简单的例子有没有让你想到些什么？

对的，没错，就是升职加薪，走向巅峰必备的 PPT 技能，这种视频转场的实现效果就和我们在编辑 PPT 动画时添加的一样。

![out_transition.png](https://image.glumes.com/images/2019/10/11/out_transition.png)

而且这还是比较简单的，想要做一些花里胡哨的转场特效，缺少灵感就可以参考 PPT 里面的动画了。

另外，我们还可以对转场效果做一些总结分类，比如示例中用的是图片，可以理解成视频 A 的最后一帧显示与视频 B 的第一帧显示做转场效果，这种转场效果实际使用的人比较少，大多数是视频 A 的最后一帧与视频 B 的前一段时间的视频做转场效果。

因此也可以对转场效果做个分类：

*   视频 A 最后一帧与视频 B 第一帧做转场动画
*   视频 A 最后一帧与视频 B 前一段时间视频做转场动画
*   视频 A 最后一段时间视频 与视频 B 第一帧做转场动画
*   视频 A 最后一段时间视频 与视频 B 前一段时间视频做转场动画


这四个分类的实现原理其实都差不多，如果是一段视频的话，那么就在视频播放时更新对应纹理。


以上就在关于使用 OpenGL 在视频编辑中实现转场效果的讲解，通过这篇文章希望大家可以掌握转场的基本实现原理。

文中用到的代码示例，可以关注我的微信公众号【纸上浅谈】，回复 “转场” 即可~~~









