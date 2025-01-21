---
title: " 《OpenGL ES 3.x 游戏开发》 光照系列之散射光"
date: 2018-07-22T23:27:04+08:00
subtitle: "学习笔记内容摘录"
draft: false
categories: ["OpenGL"]
tags: ["OpenGL"]
toc: true
 
slug: "opengl-tutorial-diffuse-light"
---


在前面的文章中介绍了 OpenGL 中的环境光，现在就是散射光了。

<!--more-->

散射光指的是从物体表面向全方位 360° 均匀发射的光，如下图：

![](https://image.glumes.com/images/2019/04/27/diffuse_demo.png)

散射光具体代表的是现实世界中粗糙的物体表面被光照射时，发射光在各个方向基本均匀的情况，如下图：

![](https://image.glumes.com/images/2019/04/27/diffuse_demo2.png)


虽然反射后的散射光在各个方向是均匀的，但散射光反射的强度与入射光的强度以及入射的角度密切相关。


因此，当光源的位置发生变化时，散射光的效果会发生明显变化，主要体现在当光垂直照射到物体表面时比斜照时要亮，其计算公式如下：

> 散射光照射结果 = 材质的反射系数 * 散射光强度 * max(cos(入射角)，0)

在实际开发中，散射光照射结果分为两步进行：

> 散射光最终强度 = 散射光强度 * max(cos(入射角)，0)

> 散射光照射结果 = 材质的反射系数 * 散射光最终强度

其中：材质的反射系数实际指的就是物体被照射处的颜色，散射光强度指的是散射光中 RGB（红、绿、蓝）3 个色彩通道的强度。

从公式中可以看到，与环境光计算公式唯一的区别就是引入了最后一项 `max(cos(入射角)，0)`，这代表着入射角越大、发射强度越弱，当入射角的余弦值为负时（即入射角大于 90°），反射强度为 0 。

由于入射角为入射光向量与法向量的夹角，因此，其余弦值并不需要调用三角函数进行计算，只需要将两个向量归一化，然后进行点积计算就可以得出余弦值。

![](https://image.glumes.com/images/2019/04/27/diffuse_cal.png)


图中的 N 代表被照射点表面的法向量，P 为被照射点，L 为从 P 点到光源的向量，N 与 L 的夹角即为入射角。


> 关于点积：在数学中，两个向量的点乘为两个向量夹角的余弦值乘以两个向量的模


## 实践


加入散射光对程序的影响主要还是在着色器代码上。


对于片段着色器，变化不会太大，因为最终的都是物体本身的颜色乘以散射光最终的照射结果，这个结果可以在顶点着色器中计算好，直接传递给片段着色器。

```gls
#version 300 es
precision mediump float;
uniform float uR;
in vec3 vPosition;//接收从顶点着色器过来的顶点位置
in vec4 vDiffuse;//接收从顶点着色器过来的散射光最终强度
out vec4 fragColor;
void main()
{
   // 省略一些代码
   //最终颜色
   vec4 finalColor=vec4(color,0);
   //根据散射光最终强度计算片元的最终颜色值
   fragColor=finalColor*vDiffuse;
}
```

重点就在于顶点着色器里面了，除了以往的代码之外，根据散射光的计算公式，我们还需添加一些新的变量，比如：光源的位置。不同光源位置的照射结果不同。

具体的着色器代码如下：

```glsl
#version 300 es
uniform mat4 uMVPMatrix; 						//总变换矩阵
uniform mat4 uMMatrix; 							//变换矩阵(包括平移、旋转、缩放)
uniform vec3 uLightLocation;						//光源位置
in vec3 aPosition;  						//顶点位置
in vec3 aNormal;    						//顶点法向量
out vec3 vPosition;							//用于传递给片元着色器的顶点位置
out vec4 vDiffuse;							//用于传递给片元着色器的散射光分量
void pointLight (								//散射光光照计算的方法
  in vec3 normal,								//法向量
  inout vec4 diffuse,								//散射光计算结果
  in vec3 lightLocation,							//光源位置
  in vec4 lightDiffuse							//散射光强度
){
  vec3 normalTarget=aPosition+normal;					//计算变换后的法向量
  vec3 newNormal=(uMMatrix*vec4(normalTarget,1)).xyz-(uMMatrix*vec4(aPosition,1)).xyz;
  newNormal=normalize(newNormal);					//对法向量归一化
//计算从表面点到光源位置的向量vp
  vec3 vp= normalize(lightLocation-(uMMatrix*vec4(aPosition,1)).xyz);
  vp=normalize(vp);									//归一化vp
  float nDotViewPosition=max(0.0,dot(newNormal,vp)); 	//求法向量与vp向量的点积与0的最大值
  diffuse=lightDiffuse*nDotViewPosition;			//计算散射光的最终强度
}
void main(){
   gl_Position = uMVPMatrix * vec4(aPosition,1); 	//根据总变换矩阵计算此次绘制此顶点的位置
   vec4 diffuseTemp=vec4(0.0,0.0,0.0,0.0);
   pointLight(normalize(aNormal), diffuseTemp, uLightLocation, vec4(0.8,0.8,0.8,1.0));
   vDiffuse=diffuseTemp;					//将散射光最终强度传给片元着色器
   vPosition = aPosition; 					//将顶点的位置传给片元着色器
}
```

首先，定义了一个函数，用来计算散射光的最终强度。

散射光的最终强度 =  散射光强度 * max(cos(入射角)，0) 。

根据此公式，散射光强度自己设定为 `vec4(0.8,0.8,0.8,1.0)`，那么接下来要做的就是计算入射光向量和法向量之间的夹角，根据点乘公式进行计算：


由于在世界模型里，点的位置会发生移动，所以也要把法向量进行移动到当前的观察坐标下：

```glsl
  vec3 normalTarget=aPosition+normal;					//计算变换后的法向量
  vec3 newNormal=(uMMatrix*vec4(normalTarget,1)).xyz-(uMMatrix*vec4(aPosition,1)).xyz;
```

然后把法向量进行归一化：

```glsl
  newNormal=normalize(newNormal);					//对法向量归一化
```

接下来就是用到光源的位置了，计算入射光向量并进行归一化。

```glsl
  vec3 vp= normalize(lightLocation-(uMMatrix*vec4(aPosition,1)).xyz);
```

最后就是计算余弦值。

```glsl
  float nDotViewPosition=max(0.0,dot(newNormal,vp)); 	//求法向量与vp向量的点积与0的最大值
```

得到了余弦值之后，就可以得到散射光的最终强度了，有了最终强度直接传递给片段着色器进行计算就方便多了。

具体效果如下：

![](https://res.cloudinary.com/glumes-com/image/upload/v1532273099/code/diffuse_use.gif)


可以拖滑动条改变光源的位置，明显看到光源不同得到的照射结果也是不同的。



具体的示例代码可以参考我的 Github 项目，求一波 Star 。

> [https://github.com/glumes/AndroidOpenGLTutorial](https://github.com/glumes/AndroidOpenGLTutorial)

## 参考

1. 《OpenGL ES 3.x 游戏开发》
