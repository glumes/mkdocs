---
title: "Shader 优化 | OpenGL 绘制网格效果"
date: 2020-05-22T14:33:35+08:00
subtitle: ""
tags: ["OpenGL"]
categories: ["OpenGL"]
toc: true
 
draft: false
original: true
slug: "opengl-draw-grid-optimize"
---

> 避免图片丢失，建议阅读微信原文：
> 
> https://mp.weixin.qq.com/s/FmILdXuy3HMpv8H1Kz_fPw

前几天发布了这样一篇文章：

> [KodeLife | Shader 实时编辑预览的强大工具使用实践](https://mp.weixin.qq.com/s/6ZMP6Tc_MqggjAXS_GjV2Q)

除了介绍 KodeLife 的使用之外，还附带了一个 Shader 绘制网格效果的代码。

把这篇文章发到技术群里，随机就有大佬指出不足之处，提示说代码还可以进一步优化，并且提供了源码学习。

> 可见加入一个高质量的技术群是多么重要，哪怕平时不说话，围观大佬们聊天都能学到很多。


现在加入还来得及，尚有余位，详情点击如下链接：

> [移动端技术交流喊你入群啦~~~](https://mp.weixin.qq.com/s/M4IiA-dTg1R68TeFaFvNPg)

<!--more-->

---

## Shader 讲解

在我的 Shader 代码中是这样绘制网格的：

```cpp
    vec2 fragcoord = vec2(gl_FragCoord.xy / u_resolution);
    vec3 bgColor = vec3(1.0,1.0,1.0);
    vec3 pixelColor = bgColor;
    vec3 gridColor = vec3(0.5,0.5,0.5);

    const float width = 0.1;
    const float minWidth = 0.003;
    for(float i = 0.0; i < 1.0; i+=width){
    if (mod(fragcoord.x,width) < minWidth || mod(fragcoord.y,width) < minWidth){
            pixelColor = gridColor;
        }
    }
    gl_FragColor = vec4(pixelColor,1.0);
```

首先，讲解几个概念：

`gl_FragCoord` 代表当前像素相对于屏幕的坐标，屏幕左下角为原点。

`u_resolution` 是当前图像的分辨率。

用 `gl_FragCoord` 除以 `u_resolution` 得到的结果 `fragcoord` 就是归一化的屏幕坐标。

由于已经归一化了那么 `fragcoord` 的值就在 `[0,1]` 的闭区间内。

同时用 `gridColor` 作为网格的颜色，`bgColor` 作为背景色，也是默认的颜色，`pixelColor` 作为最后输出的颜色。

那么，代码的重点就在于 `for` 循环里面了。

由于 `fragcoord` 归一化有了确定的值域范围，所以可以在 `for` 循环中将它十等分。

另外，因为片段着色器每个像素都会执行一遍，每次 `fragcoord` 值都是变化的，但不管怎么变化，它的范围都会落在 `for` 循环的十等份里面。

比如其中某一份的范围是 `[0.2,0.3)` 的左闭右开区间，当前像素就落在这个范围内。

那么 `mod` 取模函数就会判断当前值距离左区间阈值是否在 `minWidth` 范围内，其中 `minWidth` 相当于是指定网格线的宽度。

如果在范围内，那么显示的颜色就是网格色，否则就是默认的背景色。

以上的讲解对于坐标的 `x` 和 `y` 值是一样的道理。原理通过判断该像素点的坐标是否位于临界范围内来选择性着色。

显示这种绘制方式是有它的弊端，因为每一个像素执行片段着色器的时候，都要进行一次 `for` 循环判断它处于哪个区域内。

这样就有了太多不必要的计算流程，尤其是 `for` 循环的每次遍历。

---

接下来就是微信群中大佬给出的 Shader 代码：

```cpp
vec2 st = vec2(gl_FragCoord.xy / u_resolution);
st.x *= u_resolution.x / u_resolution.y;

vec3 color = vec3(.0);

st *= 10.;

vec2 i_st = floor(st);
vec2 f_st = fract(st);

color += step(.98,f_st.x) + step(.98,f_st.y);

gl_FragColor = vec4(color,1.0);
```


可以一眼看出这里面没有 `for` 循环的操作了。

还是先讲解几个级别操作：

`floor` 函数就是向下取整的操作

`fract` 函数是 `x - floor(x)` 的操作，也就是取小数部分的意思。

通过对 `st` 进行 `floor` 和 `fract` 操作可以分出它的整数和小数部分。

`step` 函数类似于 `if` 判断，当第二个参数大于等于第一个参数，则返回 1 ，否则返回 0 。

整个 Shader 代码第一行还是相同的，都是归一化操作。

然后在第二行
 
```cpp
st.x *= u_resolution.x / u_resolution.y
```

实际上是做了一个比例切换的操作。将 `st` 的 `x` ，`y` 值按照图像分辨率的宽高比做了调整，其中以 `y` 为基准 1 。

这样一来，`st` 的 `y` 值还是在 `[0,1]` 范围内，而 `x` 值可能大于也可能小于这个范围了，这都取决于图像分辨率了。

接下来将 `st` 乘了 10 ，这下 st 的值域范围就在 `[0,10]` 了 ，这样的操作是为了接下来的 `floor` 函数，因为它是取整，如果都在 `[0,1]` 范围内，取的整数永远都是 0 了。

前面转换操作是为了接下来的重点函数 `step` 。

```cpp
color += step(.98,f_st.x) + step(.98,f_st.y);
```

前面的 `floor` 其实也是将 `x` 和 `y` 轴做了等分，比如 `y` 的值域是 `[0,1]` ，乘以 10 之后，就是十等分，`x` 的值域如果是 `[0,1.7]` ，乘以 10 之后，就是十七等分。

而 `fract` 操作的结果范围必然是 `[0,1)` 的左闭右开区间。

`step` 函数的意图就是如果该像素点的坐标接近于等分线，那么 color 的颜色值返回的就是 1 ，显示白色，否则返回 0 ，显示黑色。

比如，st 的 `x` 值是 7.99 了，接近于 8 ，那么就要显示白色网格线了，对于 `y` 值同理。

这样一来就可以对每个像素点进行判断，根据它的坐标决定要显示什么颜色。

## 总结对比

在第二种绘制中，由于做了比例转换操作，所以绘制出来的网格大小都是一致的，且都是正方形。

而第一种没有比例切换操作，当宽高不同的情况下，同样进行十等分的话，画出来的网格是个长方形了。

但是，两种绘制的思路都是相同的，姑且称它为 `接近法` 吧，当绘制的像素接近等分线时，就显示不一样的颜色。

于是，等分线的操作思路就各有不同了。前者是利用 `for` 循环来制造划分，后者则是利用当前像素的 `x`、`y` 值的特点来绘制的。

当然更推崇后者的绘制方式了，也是学到了新技巧~~~

能力不足，经验有限，文中有何不对的地方欢迎批评指正，也可以加我微信 ezglumes 交流~~





