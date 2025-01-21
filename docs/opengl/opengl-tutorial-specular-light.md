---
title: "《OpenGL ES 3.x 游戏开发》光照系列之镜面光"
date: 2018-07-23T23:11:20+08:00
subtitle: "学习笔记内容摘录"
draft: false
categories: ["OpenGL"]
tags: ["OpenGL"]
toc: true
 
slug: "opengl-tutorial-specular-light"
---


在前面的文章中介绍了 OpenGL 的[环境光](https://glumes.com/post/opengl/opengl-tutorial-ambient-light/)和[散射光](https://glumes.com/post/opengl/opengl-tutorial-diffuse-light/)，现在就是最后一个镜面光了。


<!--more-->


还是先理解一下什么是镜面光：

![](https://image.glumes.com/images/2019/04/27/specular_demo.png)

就如同使用我们使用镜子反射太阳光一样，在现实世界中，当光滑表面被照射时会有方向很集中的反射光，这就是镜面光。

与散射光最终强度仅依赖于入射光和被照射点法向量的夹角不同，镜面光的最终强度还依赖于观察者的位置，也就是说，如果从摄像机到被照射点的向量不在反射光方向集中的范围内，观察者将不会看到镜面光，就如上图所示。

接下来就是镜面光的照射计算公式：

> 镜面光照射结果 = 

> $材质的反射系数 * 镜面光强度 * max（0，（cos（半向量与法向量的夹角）^{粗糙度}$）


在实际开发中，镜面光照射结果分为两步进行：

> 镜面光最终强度 = 

> $镜面光强度 * max（0，（cos（半向量与法向量的夹角）^{粗糙度}$）

> 镜面光照射结果 = 材质反射系数 * 镜面光最终强度

其中：材质的反射系数实际指的就是物体被照射处的颜色，镜面光强度指的是镜面光中 RGB（红、绿、蓝）3 个色彩通道的强度。

与计算散射光公式不同的是，计算余弦值时对应的角不再是入射角，而是`半向量`和法向量的夹角。其中，半向量指的是从被照射点到光源的向量与从被照射点到观察点向量的平均向量。

`半向量`的图示如下：

![](https://image.glumes.com/images/2019/04/27/specular_cal.png)

其中，V 表示从被照射点到观察点的向量，N 为被照射点表面法向量，H 为半向量，L 为从被照射点到光源的向量。

可以看到，向量 V 和向量 L 有共同交点 P ，所以它们在同一平面，而向量 H 是向量 V 和向量 L 的平均向量，所以向量 H 、V、L 共面，并且向量 H 和向量 V、L 的夹角相等。

因此计算向量 H ，首先需要将向量 V 和向量 L 归一化，然后将归一化的向量 V 和 L 求和并再次归一化即可得到向量 H 。

有了半向量 H 之后，计算其和法向量的余弦值就和在镜面光中提到的一样，进行点积就好了。


除此之外，还有一个要考虑的就是物体的粗糙程度，物体越粗糙，镜面光面积越小，反之镜面光面积越大，这也挺符合现实世界的，毛毯的反射光肯定没有玻璃的反射光大。


## 实践

加入镜面光对程序的影响主要还是在着色器代码上。

对于片段着色器，依旧变化不大，主要的计算工作在顶点着色器，顶点着色器计算完最后镜面光照射强度之后，直接传递给片段着色器就好了。

```glsl
#version 300 es
precision mediump float;
uniform float uR;
in vec3 vPosition;//接收从顶点着色器过来的顶点位置
in vec4 vDiffuse;//接收从顶点着色器过来的镜面光最终强度
out vec4 fragColor;
void main()
{
   // 省略一些代码
   //最终颜色
   vec4 finalColor=vec4(color,0);
   //根据镜面光最终强度计算片元的最终颜色值
   fragColor=finalColor*vDiffuse;
}
```

重点依旧是顶点着色器：

```glsl
#version 300 es
uniform mat4 uMVPMatrix; 	//总变换矩阵
uniform mat4 uMMatrix; 		//变换矩阵
uniform vec3 uLightLocation;	//光源位置
uniform vec3 uCamera;		//摄像机位置
in vec3 aPosition;  	//顶点位置
in vec3 aNormal;   	//法向量
out vec3 vPosition;		//用于传递给片元着色器的顶点位置
out vec4 vSpecular;		//用于传递给片元着色器的镜面光最终强度

void pointLight(				//定位光光照计算的方法
  in vec3 normal,			//法向量
  inout vec4 specular,		//镜面光最终强度
  in vec3 lightLocation,		//光源位置
  in vec4 lightSpecular		//镜面光强度
){
  vec3 normalTarget=aPosition+normal; 	//计算变换后的法向量
  vec3 newNormal=(uMMatrix*vec4(normalTarget,1)).xyz-(uMMatrix*vec4(aPosition,1)).xyz;
  newNormal=normalize(newNormal);  	//对法向量归一化
  //计算从表面点到摄像机的向量
  vec3 eye= normalize(uCamera-(uMMatrix*vec4(aPosition,1)).xyz);
  //计算从表面点到光源位置的向量vp
  vec3 vp= normalize(lightLocation-(uMMatrix*vec4(aPosition,1)).xyz);
  vp=normalize(vp);//格式化vp
  vec3 halfVector=normalize(vp+eye);	//求视线与光线的半向量
  // 物体表面的光滑度，自己拟定为 50.0
  float shininess=50.0;				//粗糙度，越小越光滑

  float nDotViewHalfVector=dot(newNormal,halfVector);			//法线与半向量的点积
  float powerFactor=max(0.0,pow(nDotViewHalfVector,shininess)); 	//镜面反射光强度因子

  specular=lightSpecular*powerFactor;    //最终的镜面光强度
}

void main()  {
   gl_Position = uMVPMatrix * vec4(aPosition,1); //根据总变换矩阵计算此次绘制此顶点的位置
   vec4 specularTemp=vec4(0.0,0.0,0.0,0.0);
   pointLight(normalize(aNormal), specularTemp, uLightLocation, vec4(0.7,0.7,0.7,1.0));//计算镜面光
   vSpecular=specularTemp;	//将最终镜面光强度传给片元着色器
   vPosition = aPosition; 		//将顶点的位置传给片元着色器
}
```

在散射光，我们需要传递光源的位置，而在镜面光中，不仅仅是光源的位置，还需要把相机的位置传递给着色器代码用于计算。


在顶点着色器代码中，还是定义了一个函数 pointLight 用于计算镜面光照射强度。

由于法向量的基于物体模型空间定义的，首先还是要将法向量转换到世界模型空间中。

```glsl
  vec3 normalTarget=aPosition+normal; 	//计算变换后的法向量
  vec3 newNormal=(uMMatrix*vec4(normalTarget,1)).xyz-(uMMatrix*vec4(aPosition,1)).xyz;
```

其次将法向量归一化，然后计算物体表面的点到观察点相机的向量、计算物体表面的点到光源位置的向量，并将这两个向量也进行归一化。

```glsl
  newNormal=normalize(newNormal);  	//对法向量归一化
  //计算从表面点到摄像机的向量
  vec3 eye= normalize(uCamera-(uMMatrix*vec4(aPosition,1)).xyz);
  //计算从表面点到光源位置的向量vp
  vec3 vp= normalize(lightLocation-(uMMatrix*vec4(aPosition,1)).xyz);
  vp=normalize(vp);//归一化vp
```

得到这三个向量归一化之后的向量，相对于就是在为接下来的计算做准备了。

首先是得到`半向量`，并将它归一化，接着就是求`半向量`和法向量的点积，得到余弦值。

```glsl
  vec3 halfVector=normalize(vp+eye);	//求视线与光线的半向量
  float nDotViewHalfVector=dot(newNormal,halfVector);			//法线与半向量的点积
```

然后就是计算余弦值与物体表面粗糙因子的平方，并和 0 进行比较，若小于 0，则为 0 ，最后得到的是镜面反射光强度因子。

```glsl
  // 物体表面的光滑度，自己拟定为 50.0
  float shininess=50.0;				//粗糙度，越小越光滑
  float powerFactor=max(0.0,pow(nDotViewHalfVector,shininess)); 	//镜面反射光强度因子
```

将反射光强度因子和镜面光强度相乘，得到最终的镜面光强度，这个强度就是用来传递给片段着色器的最终镜面光强度。

使用效果如下：

![镜面光使用](https://res.cloudinary.com/glumes-com/image/upload/v1532348436/code/specular_demo.gif)


如果我们改变自己拟定的物体粗糙度，粗糙度越大，会使得镜面光面积越小，粗糙度越小，会使得镜面光面积越大。

同时，拖动滑块，改变光源的位置，也会影响镜面光反射的地方，镜面光也受光源的影响。



具体的示例代码可以参考我的 Github 项目，求一波 Star 。

> [https://github.com/glumes/AndroidOpenGLTutorial](https://github.com/glumes/AndroidOpenGLTutorial)

## 参考

1. 《OpenGL ES 3.x 游戏开发》

