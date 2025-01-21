---
title: "《OpenGL ES 3.x 游戏开发》 3D 模型加载和渲染"
date: 2018-07-02T23:37:16+08:00
subtitle: "学习笔记内容摘录"
draft: false
categories: ["OpenGL"]
tags: ["OpenGL"]
toc: true
 
slug: "opengl-tutorial-import-3d-object"
---

> 避免图片丢失，建议阅读微信原文：
> 
> https://mp.weixin.qq.com/s/OCcTOJArxipvY-v_HuD0Eg

在使用 OpenGL 绘制时，我们最多绘制的是一些简单的图形，比如三角形、圆形、立方体等，因为这些图形的顶点数量不多，还是可以手动的写出那些顶点的，可要是绘制一些复杂图形该怎么办呢？

<!--more-->

这时候就可以使用 OpenGL 来加载 3D 模型。先使用 3D 建模工具构建物体，然后再将物体导出成特定的文件格式，最终通过 OpenGL 渲染模型。

例如如下的 3D 模型文件图像：

![](https://image.glumes.com/images/2019/04/27/WechatIMG24.jpg)


## Obj 模型文件

obj 模型文件是众多 3D 模型文件中的一种，它的格式比较简单，本质上就是文本文件，只是格式固定了格式。

obj 文件将顶点坐标、三角形面、纹理坐标等信息以固定格式的文本字符串表示。

截取一小段 obj 文件内容：

```glsl
# Max2Obj Version 4.0 Mar 10th, 2001
#
# object (null) to come ...
#
v  -0.052045 11.934561 -0.071060
v  -0.052045 11.728649 1.039199
...
# 288 vertices
vt 0.000000 0.000000 0.000000
vt 1.000000 0.000000 0.000000
...
vt 1.000000 1.000000 0.000000
# 122 texture vertices
vn 0.000000 0.000000 -1.570796
vn 0.000000 0.000000 -1.570796
...
vn 0.000000 0.000000 1.570796
# 8 vertex normals
g (null)
f 1/10/1 14/12/11 13/4/11 
f 1/11/4 2/12/3 14/12/11
...
f 5/4/5 7/3/7 3/1/3
# 576 faces
g
```

*	"#" 开头的行表示注释，加载过程中可以忽略
*	“v” 开头的行用于存放顶点坐标，后面三个数表示一个顶点的 x , y , z 坐标
	如：
	
```java
v  -0.052045 11.934561 -0.071060
```

*	"vt" 开头的行表示存放顶点纹理坐标，后面三个数表示纹理坐标的 S，T，P 分量，其中 P 指的是深度纹理采样，主要用于 3D 纹理的采样，但使用的较少
	如：
```java
vt 0.000000 0.000000 0.000000
```

*	"vn" 开头的行用于存放顶点法向量，后面三个数分别表示一个顶点的法向量在 x 轴，y 轴，z 轴上的分量。
	如：
```java
vn 0.000000 0.000000 1.570796
```
*	“g” 开头的行表示一组的开始，后面的字符串为此组的名称。组就是由顶点组成的一些面的集合，只包含 “g” 的行表示一组的结束，与 “g” 开头的行对应。
*	"f" 开头的行表示组中的一个面，对于三角形图形，后面有三组用空格分隔的数据，代表三角形的三个顶点。每组数据中包含 3 个数值，用 `/` 分隔，依次表示**顶点坐标数据索引**、**顶点纹理坐标数据索引**、**顶点法向量数据索引**，注意这里都是指索引，而不是指具体数据，索引指向的是具体哪一行对应的坐标
	如：
```java
f 1/10/1 14/12/11 13/4/12 
```
如上数据代表了三个顶点，其中三角形 3 个顶点坐标来自 1、14、13 号以 "v" 开头的行， 3 个顶点的纹理坐标来自 10、12、4 号以 “vt” 开头的行，3 个顶点的法向量来自 1、11、12 号以 “vn” 开头的行。

如果顶点坐标没有法向量和纹理坐标，那么直接可以忽略，用空格将三个顶点坐标索引分开就行
```java
f 1 3 4
```

最后 OpenGL 在绘制时采用的是 `GL_TRIANGLES`，也就是由 ABCDEF 六个点绘制 ABC、DEF 两个三角形，所以 "f" 开头的行都代表绘制一个独立的三角形，最终图像由一个一个三角形拼接组成，并且彼此的点可以分开。 


## 加载 Obj 模型文件

明白了 Obj 模型文件代表的含义，接下来把它加载并用 OpenGL 进行渲染。

Obj 模型文件实质上也就是文本文件了，通过读取每一行来进行加载即可，假设加载的模型文件只有顶点坐标，实际代码如下：

```java
		// 加载所有的顶点坐标数据，把 List 容器的 index 当成 索引
        ArrayList<Float> alv = new ArrayList<>();
        // 代表绘制图像的每一个小三角形的坐标
        ArrayList<Float> alvResult = new ArrayList<>();
        // 最终要传入给 OpenGL 的数组
        float[] vXYZ;
        try {
            InputStream in = context.getResources().getAssets().open(fname);
            InputStreamReader isr = new InputStreamReader(in);
            BufferedReader br = new BufferedReader(isr);
            String temps = null;
			// 遍历每一行来读取内容
            while ((temps = br.readLine()) != null) {
	            // 正则表达式 用空格分开
                String[] tempsa = temps.split("[ ]+");
                // 先把所有的顶点坐标加入到 List 中，这样就有了索引
                if (tempsa[0].trim().equals("v")) {
                    alv.add(Float.parseFloat(tempsa[1]));
                    alv.add(Float.parseFloat(tempsa[2]));
                    alv.add(Float.parseFloat(tempsa[3]));
                } else if (tempsa[0].trim().equals("f")) {
                // 根据 f 指示的索引，找到对应的顶点坐标，
                // 这里 -1 的操作是因为 List 从 0 开始，f 开头的行的索引从 1 开始
                // *3 是因为要跳过 3 的倍数个顶点
                    int index = Integer.parseInt(tempsa[1].split("/")[0]) - 1;
                    alvResult.add(alv.get(3 * index));
                    alvResult.add(alv.get(3 * index + 1));
                    alvResult.add(alv.get(3 * index + 2));

                    index = Integer.parseInt(tempsa[2].split("/")[0]) - 1;
                    alvResult.add(alv.get(3 * index));
                    alvResult.add(alv.get(3 * index + 1));
                    alvResult.add(alv.get(3 * index + 2));

                    index = Integer.parseInt(tempsa[3].split("/")[0]) - 1;
                    alvResult.add(alv.get((3 * index)));
                    alvResult.add(alv.get((3 * index + 1)));
                    alvResult.add(alv.get((3 * index + 2)));
                }
            }
            // 把面的坐标转换为最终要传递给 OpenGL 的数组
            // 根据这个数组，然后按照 GL_TRIANGLES 方式进行绘制
            int size = alvResult.size();
            vXYZ = new float[size];
            for (int i = 0; i < size; i++) {
                vXYZ[i] = alvResult.get(i);
            }
            return vXYZ;
        } catch (IOException e) {
            return null;
        }
```

通过上面的函数就计算出了最终的顶点坐标位置，并将此顶点坐标位置传入给 GPU ，通过 FloatBuffer 进行转换等等，这就和之前的文章内容相同了。

![](https://image.glumes.com/images/2019/04/27/WechatIMG25.jpg)

如果只是单纯的导入了所有顶点，并决定了要绘制的颜色，就会出现类似上面的单一颜色的绘制情况，事实上可以通过修改片段着色器来给 3D 模型添加条纹着色效果。

## 利用着色器添加条纹着色效果

通过修改片段着色器来给 3D 形状添加条纹着色效果。

```glsl
precision mediump float;
varying  vec3 vPosition;  //顶点位置
void main() {
   vec4 bColor=vec4(0.678,0.231,0.129,0);//条纹的颜色
   vec4 mColor=vec4(0.763,0.657,0.614,0);//间隔的颜色
   float y=vPosition.y;
   y=mod((y+100.0)*4.0,4.0);
   if(y>1.8) {
     gl_FragColor = bColor;//给此片元颜色值
   } else {
     gl_FragColor = mColor;//给此片元颜色值
   }
// 默认使用单一颜色进行绘制
//   vec4 white = vec4(1,1,1,1);
//   gl_FragColor = white;
}
```

实现的方式也是根据片段的 y 坐标所在位置来决定该片段是采样条纹的颜色还是间隔的颜色。

最后，加载 3D 模型就先了解到这了，如果想要加载更多效果，倒是可以继续深挖，只是没有 MAC 版本的 3ds Max 软件，却是少了一些乐趣~~

## 参考

1. 《OpenGL ES 3.x 游戏开发》