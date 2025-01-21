---
title: "《OpenGL ES 3.x 游戏开发》光照系列之效果混合"
date: 2018-07-24T09:39:10+08:00
subtitle: "学习笔记内容摘录"
draft: false
categories: ["OpenGL"]
tags: ["OpenGL"]
toc: true
 
slug: "opengl-tutorial-light-mix"
---


在前面的系列文章中，分别介绍了 OpenGL 的环境光、散射光、镜面光。

现在尝试将这些光照效果混合起来，让整个场景显得更加逼真。


具体的效果如下：

![](https://res.cloudinary.com/glumes-com/image/upload/v1532360616/light_mix_xmv86l.gif)

<!--more-->

在这个场景里有环境光、散射光和镜面光。拖动滑块的位置，可以改变光源的位置，不同位置的光源，看到的效果是不同的。

要实现这样的效果，其实就是把之前文章的代码结合一下：

> 混合光照射结果 = 环境光照射结果 + 散射光照射结果 + 镜面光照射结果

至于不同光照射结果计算，可以参考之前的文章了。

对于代码来说，重点还是在于着色器代码上的变化了。

## 实践

片段着色器的改动依旧不大，不同光照射结果都还是在顶点着色器中进行了。

片段着色器代码如下：

```glsl
#version 300 es
precision mediump float;
uniform float uR;
in vec3 vPosition;//接收从顶点着色器过来的顶点位置
in vec4 vAmbient;//接收从顶点着色器过来的环境光最终强度
in vec4 vDiffuse;//接收从顶点着色器过来的散射光最终强度
in vec4 vSpecular;//接收从顶点着色器过来的镜面反射光最终强度
out vec4 fragColor;
void main()
{
   // 省略一些代码
   //最终颜色
   vec4 finalColor=vec4(color,0);
   //给此片元颜色值
   fragColor=finalColor*vAmbient + finalColor*vDiffuse + finalColor*vSpecular;
}
```
由顶点着色器传递过来的环境光最终强度、散射光最终强度、镜面光最终强度，分别乘以物体的颜色，然后三者相加得到最后的颜色结果。


其实，重点还是在于顶点着色器代码中：

```glsl
#version 300 es
uniform mat4 uMVPMatrix; 		//总变换矩阵
uniform mat4 uMMatrix; 			//变换矩阵
uniform vec3 uLightLocation;		//光源位置
uniform vec3 uCamera;			//摄像机位置
in vec3 aPosition;  		//顶点位置
in vec3 aNormal;    		//法向量
out vec3 vPosition;			//用于传递给片元着色器的顶点位置
out vec4 vAmbient;			//用于传递给片元着色器的环境光最终强度
out vec4 vDiffuse;			//用于传递给片元着色器的散射光最终强度
out vec4 vSpecular;			//用于传递给片元着色器的镜面光最终强度
void pointLight(					//定位光光照计算的方法
  in vec3 normal,				//法向量
  inout vec4 ambient,			//环境光最终强度
  inout vec4 diffuse,				//散射光最终强度
  inout vec4 specular,			//镜面光最终强度
  in vec3 lightLocation,			//光源位置
  in vec4 lightAmbient,			//环境光强度
  in vec4 lightDiffuse,			//散射光强度
  in vec4 lightSpecular			//镜面光强度
){
  ambient=lightAmbient;			//直接得出环境光的最终强度
  vec3 normalTarget=aPosition+normal;	//计算变换后的法向量
  vec3 newNormal=(uMMatrix*vec4(normalTarget,1)).xyz-(uMMatrix*vec4(aPosition,1)).xyz;
  newNormal=normalize(newNormal); 	//对法向量归一化
  //计算从表面点到摄像机的向量
  vec3 eye= normalize(uCamera-(uMMatrix*vec4(aPosition,1)).xyz);
  //计算从表面点到光源位置的向量vp
  vec3 vp= normalize(lightLocation-(uMMatrix*vec4(aPosition,1)).xyz);
  vp=normalize(vp);//归一化vp
  vec3 halfVector=normalize(vp+eye);	//求视线与光线的半向量
  float shininess=50.0;				//粗糙度，越小越光滑
  float nDotViewPosition=max(0.0,dot(newNormal,vp)); 	//求法向量与vp的点积与0的最大值
  diffuse=lightDiffuse*nDotViewPosition;				//计算散射光的最终强度
  float nDotViewHalfVector=dot(newNormal,halfVector);	//法线与半向量的点积
  float powerFactor=max(0.0,pow(nDotViewHalfVector,shininess)); 	//镜面反射光强度因子
  specular=lightSpecular*powerFactor;    			//计算镜面光的最终强度
}
void main(){
   gl_Position = uMVPMatrix * vec4(aPosition,1); //根据总变换矩阵计算此次绘制此顶点位置
   vec4 ambientTemp,diffuseTemp,specularTemp;	  //用来接收三个通道最终强度的变量
   pointLight(normalize(aNormal),ambientTemp,diffuseTemp,specularTemp,uLightLocation,
   vec4(0.15,0.15,0.15,1.0),vec4(0.8,0.8,0.8,1.0),vec4(0.7,0.7,0.7,1.0));
   vAmbient=ambientTemp; 		//将环境光最终强度传给片元着色器
   vDiffuse=diffuseTemp; 		//将散射光最终强度传给片元着色器
   vSpecular=specularTemp; 		//将镜面光最终强度传给片元着色器
   vPosition = aPosition;  //将顶点的位置传给片元着色器
}
```

首先，顶点着色器需要在程序中传递光源的位置，也需要传递观察者，也就是相机的位置。

然后在 pointLight 函数里面计算最终的不同的光照强度。

在 pointLight 函数里面最先操作的还是将法向量进行相应的位移，然后就是准备在不同光照情况下需要用到的向量：

*	法向量归一化之后的向量
*	从观察点到相机的向量进行归一化之后的向量
*	从观察点到光源位置的向量进行归一化之后的向量
*   半向量归一化之后的向量


有了这些个向量，要计算余弦值，只要调用 `dot` 函数计算两个向量的点积就好了，最后还是按照之前文章中的公式进行计算就好了。

如果你对之前的光照比较熟悉了，那么对于光照混合效果也会很快上手，就是在着色器之间传递的变量多了一些，其他的按照公式计算就好了。

具体的示例代码可以参考我的 Github 项目，求一波 Star 。

> [https://github.com/glumes/AndroidOpenGLTutorial](https://github.com/glumes/AndroidOpenGLTutorial)

## 参考

1. 《OpenGL ES 3.x 游戏开发》

