---
title: "KodeLife | Shader 实时编辑预览的强大工具使用实践"
date: 2020-05-18T11:39:56+08:00
subtitle: ""
slug: "shader-tool-kodelife-introduce"
categories: ["OpenGL"]
tags: ["OpenGL"]
toc: true
 
draft: false
original: true
---

> 避免图片丢失，建议阅读微信原文：
> 
> https://mp.weixin.qq.com/s/6ZMP6Tc_MqggjAXS_GjV2Q

经常有朋友在群里面问想学习 Shader 有什么工具可以推荐？

今天它来了~~~

推荐一款强大的 Shader 实时编辑预览的工具 —— `KodeLife` 。

对，它的名字就叫做 `KodeLife` ，可别看成 `KobeLife` 了，一个字母之差完全就是两个概念。

`KodeLife` 的官网地址如下：

> https://hexler.net/products/kodelife

贴一张主页封面图：

![](https://images.xiaozhuanlan.com/photo/2020/7d112ef33beccfe4fbf62e4879eabb25.png)

有需要的同学可以去官网下载安装，它是需要购买 **License** 的，不过可以免费使用两个月。

---

<!--more-->

## KodeLife 的编辑功能

首次打开 **KodeLife** 会加载并演示默认的 Shader 代码效果。


![](https://images.xiaozhuanlan.com/photo/2020/a58f5f70997e5ea66a297a5ecac2e2ed.jpg)


编辑区就是我们写 Shader 代码的地方，背后的画面就是实时预览的效果。这画面效果是会随着时间不断改变的，这里只是静态图看不到而已。

首次打开可能会被这个效果给吓唬到，毕竟这画面五颜六色而且还闪来闪去，其实很多东西都可以去掉的，一个简单的例子会更容易上手一些。

如下图：

![](https://images.xiaozhuanlan.com/photo/2020/31a607942e6809a4c700247953bb1d11.jpg)

看到这中间打钩的三个选项了嘛，它们分别是 OpenGL 渲染不同阶段对应的着色器，由于我们都是用 OpenGL ES ，它是 OpenGL 的子集，一些功能都被移除了，所以下面这些 Shader 都是用不到的。

简单介绍一下它们的名字：

* Tess Control
    * 全名：Tessellation Control Shader
    * 中文名：曲面细分着色器

* Tess Eval
    * 全名：Tessellation Control Shader
    * 中文名：细分计算着色器


* Geometry
    * 全名：Geometry Shader
    * 中文名：几何着色器

抛开这三个不看，那么剩下的标签页就是 `Vertex` 和 `Fragment` ，分别是顶点着色器和片段着色器，这应该很熟悉了。

---

## KodeLife 使用实践

接下来我们就要新建一个 Shader 进行编写。

![](https://images.xiaozhuanlan.com/photo/2020/447dd8893f54fc89b0f35939decabf59.jpg)

在 `File` 里面有两种 `New` 新建文件的类型。

其中第二个 `New From Template` 就会按照 `Shadertoy` 或者 `The Book of Shaders` 的示例来加载 Shader 模板工程。

> 温馨提示:
>  
> Shadertoy 是非常有名的 Shader 学习网站，上面有着绚丽的 Shader 效果，并且有源码供学习，就是网站打开速度太慢了。
> 
> The Book of Shaders 是一本非常有名的 Shader 入门学习书籍，讲解的示例简单易学。


这里我们按照 `The Book of Shaders` 提供的模板新建一个工程来编写代码。

下面是工程建好后对应的代码和效果。

![](https://images.xiaozhuanlan.com/photo/2020/5aa3f537c276771edf6568d5d99f6696.jpg)


它自带了三个 `unifrom` 变量：

* u_resolution
    * 图像的分辨率
* u_mouse
    * 鼠标点的位置
* u_time
    * 时间

可以在 Shader 去利用这个三个变量，它们的输入值是由 KodeLife 来保证的，在右侧可以查看并修改这变量的具体值。

![](https://images.xiaozhuanlan.com/photo/2020/ea1dff8ad9189fc5cc4bae1311b7c43d.jpg)


* 数字 0 区域:
    * Shader 效果的预览区域

* 数字 1 区域：
    * 开关控制是否使用下面的属性内容
    * 查看当前的属性，比如查看并编辑图像分辨率的
    * 指定 Clear Color 时的颜色

* 数字 2 区域：
    * 时间变量，可以在里面控制时间的开始和停止
    * 可以调整时间变化的速
    * 可以调整时间变化的起始和结束值，并在该区域内循环

* 数字 3 区域：
    * 显示图片的分辨率

* 数字 4 区域：
    * 设置鼠标的点击区域
    * 在数字 4 的右侧区域内点击鼠标，改变鼠标区域值
    * 可以单独设置 X 和 Y 值，也可以设置是否要归一化操作


以上就是 **KodeLife** 进行 Shader 编写的操作部分了，相信你也已经知道要如何操作了。

如果使用 `Shadertoy` 提供的模板，它自带的变量会多一点，但都大同小异了，而且这都不是重点，重点还是如何使用这些变量进行创作。

所以接下来就是发挥想象力进行 Shader 的开发了。

---

## KodeLife Shader 编写实践

提供两个简单例子，演示一下在 **KodeLife** 中编写代码实现网格效果。

代码如下：

```cpp
#ifdef GL_ES
precision mediump float;
#endif

uniform vec2 u_resolution;
uniform vec2 u_mouse;
uniform float u_time;

void main() {

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
}
```

效果如下：

![](https://images.xiaozhuanlan.com/photo/2020/63300b6794637a07f49ae40e177217af.jpg)

```cpp
#ifdef GL_ES
precision mediump float;
#endif

uniform vec2 u_resolution;
uniform vec2 u_mouse;
uniform float u_time;

void main() {
    
    vec2 r = vec2(gl_FragCoord.xy - 0.5 * u_resolution);
    vec2 fragcoord = 2.0 * r.xy / u_resolution;
    
    vec3 bgColor = vec3(1.0,1.0,1.0);
    vec3 pixelColor = bgColor;
    vec3 gridColor = vec3(0.5,0.5,0.5);
    vec3 axesColor = vec3(1.0,0.0,0.0);
    
    const float width = 0.1;
    const float minWidth = 0.008;
    for(float i = 0.0; i < 1.0; i+=width){
        if (mod(fragcoord.x,width) < minWidth || mod(fragcoord.y,width) < minWidth){
            pixelColor = gridColor;
        }
    }
    
    if (abs(fragcoord.x) < 0.008 || abs(fragcoord.y) < 0.008){
        pixelColor = axesColor;
    }
    
    
    gl_FragColor = vec4(pixelColor,1.0);
}
```

效果如下：

![](https://images.xiaozhuanlan.com/photo/2020/761980db9591b0d8301010fab6a081b0.jpg)


把代码复制粘贴到 KodeLife 中运行就能看到效果了。

这两个效果的区别就是在于中间坐标轴绘制了，你能从代码中看到有何不同吗？

在后续的文章再来讲解如何编写 Shader 吧~~~

能力不足，经验有限，文中有何不对的地方欢迎批评指正，也可以加我微信 ezglumes 交流~~