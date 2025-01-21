---
title: "《OpenGL ES 3.x 游戏开发》光照系列之环境光"
date: 2018-07-21T22:45:03+08:00
subtitle: "学习笔记内容摘录"
draft: false
categories: ["OpenGL"]
tags: ["OpenGL"]
toc: true
 
slug: "opengl-tutorial-ambient-light"
---

在 OpenGL 中使用光照。

<!--more-->

OpenGL 中的光照模型相对实现世界进行了很大的简化，将光照分成了 3 种组成元素。

*	环境光
*	散射光
*	镜面光

这三种光照是分别采用不同的数学模型独立计算的。


## 环境光 Ambient

环境光指的是从四面八方照射到物体上，全方位 360° 都均匀的光。它代表的是现实世界中从光源射出，经过多次反射后，各方向基本均匀的光。


效果如下图所示：


![环境光效果](https://image.glumes.com/images/2019/04/27/embient_light_explained.png)

环境光最大的特点是不依赖于光源的位置，而且没有方向性，而且环境光不但入射是均匀的，发射也是各项均匀的。

用于计算环境光的数学模型非常简单：

> 环境光照射结果 = 材质的反射系数 * 环境光强度


环境光强度指的是环境光中 RGB（红、绿、蓝）3 个色彩通道的强度，材质的反射系数实际指的就是物体被照射处的颜色。


## 实践

加入环境光对程序的影响就是着色器代码会有些不同。

在着色器代码上的区别就是增加了接收环境光强度，以及使用环境光强度与片元本身颜色值加权计算产生最终片元颜色值的相关代码。

以下是使用的顶点着色器代码：


```glsl
#version 300 es
uniform mat4 uMVPMatrix; //总变换矩阵
in vec3 aPosition;  //顶点位置
out vec3 vPosition;//用于传递给片元着色器的顶点位置
out vec4 vAmbient;//用于传递给片元着色器的环境光分量
void main()
{
   //根据总变换矩阵计算此次绘制此顶点位置
   gl_Position = uMVPMatrix * vec4(aPosition,1);
   //将顶点的位置传给片元着色器,用于纹理采样
   vPosition = aPosition;
	//将环境光强度传给片元着色器
   vAmbient = vec4(0.15,0.15,0.15,1.0);
}
```

在顶点着色代码中，会多定义一个变量 `vAmbien`，它是由四个数组成的向量，代表环境光强度。

我们需要把这个环境光强度传递给片段着色器，让片段着色器去根据环境光强度去生成最终的颜色。

以下是片段着色器代码：

```glsl
#version 300 es
precision mediump float;
uniform float uR;
in vec3 vPosition; //接收从顶点着色器过来的顶点位置
in vec4 vAmbient;//接收从顶点着色器过来的环境光强度
out vec4 fragColor;
void main()
{
   // 省略一些代码
   //最终颜色，代表经过采样或者是计算后的颜色
   vec4 finalColor=vec4(color,0);
	//根据环境光强度计算最终片元颜色值
   fragColor=finalColor*vAmbient;
}
```

可以看到代码中的 `fragColor` 就是物体经过环境光照射后的颜色，它也是根据上面的计算公式得来的，将物体的颜色与环境光强度相乘得到。

以下是使用环境光照射的结果实例，环境光强度为 `vec4(0.15,0.15,0.15,1.0)`。


![环境光](https://image.glumes.com/images/2019/04/27/light_ambient.jpg)


可以看到物体暗淡了一些，这就像是在黑夜中去观察物体一样，只有微弱的月光，而且月光又是均匀分布的，通过翻转球体的各个面，可以观察到明亮程度都是一样的。


而在之前的所有绘制中，物体都不是这么暗淡的，其差别就是没有给最终的颜色乘以环境光强度，或者说，我们之前绘制的环境光强度设置是 `vec4(1.0,1.0,1.0,1.0)`。

![环境光为 1 的情况](https://image.glumes.com/images/2019/04/27/light_ambient_1.jpg)

这种情况下就让其他的各种光照都显得无意义了，因为环境光是均匀分布的，而环境光都是这么亮了，照亮了每一个角落，还要其他光照干嘛呢。

所以在实际开发中，要注意给环境光照设置合适的值。


具体的示例代码可以参考我的 Github 项目，求一波 Star 。

> [https://github.com/glumes/AndroidOpenGLTutorial](https://github.com/glumes/AndroidOpenGLTutorial)

## 参考

1. 《OpenGL ES 3.x 游戏开发》
