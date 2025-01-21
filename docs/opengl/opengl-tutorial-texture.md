---
title: "OpenGL 学习系列之纹理"
date: 2018-03-21T23:08:02+08:00
subtitle: ""
draft: false
categories: ["OpenGL"]
tags: ["OpenGL"]
 
toc: true
original: true
addwechat: true
slug: "opengl-tutorial-texture"
---

> 避免图片丢失，建议阅读微信原文：
> 
> https://mp.weixin.qq.com/s/Y3FsQrJJWQogD3PWWo1cVQ

接下来探索纹理了。

<!--more-->

纹理，简单的理解就是一副图像。而把一副图像映射到图形上的过程，叫做`纹理映射`。


比如有如下图形和三角形，想要把图形中的一部分映射到三角形上。





结果就是这样的：





这就是纹理映射的一个小小例子。


## 基本原理

要注意到，OpenGL 绘制的物体是 3D 的，而纹理是 2D 的，那么纹理映射就是将 2D 的纹理映射到 3D 的物体上，可以想象成用一张纸裹着一个物体一样，不过要按照一定规律来。


OpenGL 中绘制的物体是有坐标系的，每个点都对应 x、y、z 坐标，而纹理也有着它的坐标，只要 3D 物体中的每个点都对应了 2D 纹理中的某个点，那么就可以把纹理映射到 3D 物体上去了。

纹理的坐标，叫做`纹理坐标系`。它的范围只有 $[0,0]$ 到 $[1,1]$ 。


![](https://image.glumes.com/images/2019/04/27/opengl_texture_coordinate.png)


它的坐标原点位于左下角，水平向右为 S 轴，竖直向上为 Y 轴。不论实际的纹理图片尺寸大小如何，横向、纵向坐标最大值都是 1 。

例如：实际图为 512 x 256 像素分辨率，则横向第 512 个像素对应纹理坐标为 1 ，纵向第 256 个像素对应纹理坐标为 1 。不过，纹理图最好是采用像素为 2 的 n 次方的纹理图。


纹理映射的基本思想就是：首先为图元中的每个顶点指定恰当的纹理坐标，然后通过纹理坐标在纹理图中可以确定选中的纹理区域，最后将选中纹理区域中的内容根据纹理坐标映射到指定的图元上。


纹理映射在 OpenGL 的渲染管线上的体现：在渲染管线中，先进行顶点着色器，绘制出物体的大致形状，之后会进行`光栅化`，将物体光栅化为许多片段组成，然后再进行片段着色器，将图形的每个片段进行着色。

那么就需要在 顶点着色器 中将纹理的坐标传入，在光栅化阶段，纹理坐标将根据 顶点着色器 对它的处理以及 片段和各顶点的位置关系 插值产生，然后才是将插值计算后的结果传入到片段着色器中。

## 着色器操作

相比直接绘制图形，使用纹理后，着色器也要改变了。

顶点着色器：

```glsl
attribute vec4 a_Position;
attribute vec2 a_TextureCoordinates;
varying vec2 v_TextureCoordinates;
uniform mat4 u_ModelMatrix;
uniform mat4 u_ViewMatrix;
uniform mat4 u_ProjectionMatrix;
uniform mat4 u_Matrix;

void main() {
    v_TextureCoordinates = a_TextureCoordinates ;
    gl_Position = u_ProjectionMatrix * u_ViewMatrix * u_ModelMatrix * a_Position;
}
```

在顶点着色器中多了 `v_TextureCoordinates` 变量，它是 `varying` 类型，意思为可变类型，在光栅化处理时会对该变量进行处理，随后传入到片段着色器中。

片段着色器

```glsl
precision mediump float;
uniform sampler2D u_TextureUnit;
varying vec2 v_TextureCoordinates;

void main(){
	// 未使用纹理的颜色赋值 ： gl_FragColor = u_Color;
    gl_FragColor = texture2D(u_TextureUnit,v_TextureCoordinates);
}
```


`v_TextureCoordinates1`变量就是接受来自顶点着色器传的值，`u_TextureUnit`变量就是使用的采样器，类型是`sampler2D`。

使用纹理后的片段着色器要使用 `texture2D` 函数给颜色赋值。

`texture2D`函数的作用就是采样，从纹理中采取像素赋值给 `gl_FragColor`变量，也就是最后的颜色。


## 上层代码

大致了解了着色器代码，接着就是上层的 Java 代码了。

和要创建一个 OpenGL ProgramId 类似，使用纹理也需要创建一个纹理 ID。

```java
	 /**
     * 返回加载图像后的 OpenGl 纹理的 ID
     * @param context
     * @param resourceId
     * @return
     */
    public static int loadTexture(Context context, int resourceId) {
        final int[] textureObjectIds = new int[1];
        glGenTextures(1, textureObjectIds, 0);
        if (textureObjectIds[0] == 0) {
            Timber.d("Could not generate a new OpenGL texture object.");
            return 0;
        }
        final BitmapFactory.Options options = new BitmapFactory.Options();
        options.inScaled = false;
        final Bitmap bitmap = BitmapFactory.decodeResource(context.getResources(), resourceId, options);

        if (bitmap == null) {
            Timber.d("resource Id could not be decoded");
            glDeleteTextures(1, textureObjectIds, 0);
            return 0;
        }

        glBindTexture(GL_TEXTURE_2D, textureObjectIds[0]);

        // 设置缩小的情况下过滤方式
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR_MIPMAP_LINEAR);
        // 设置放大的情况下过滤方式
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);

        // 加载纹理到 OpenGL，读入 Bitmap 定义的位图数据，并把它复制到当前绑定的纹理对象
        // 当前绑定的纹理对象就会被附加上纹理图像。
        texImage2D(GL_TEXTURE_2D, 0, bitmap, 0);

        bitmap.recycle();
        
        // 为当前绑定的纹理自动生成所有需要的多级渐远纹理
        // 生成 MIP 贴图
        glGenerateMipmap(GL_TEXTURE_2D);

        // 解除与纹理的绑定，避免用其他的纹理方法意外地改变这个纹理
        glBindTexture(GL_TEXTURE_2D, 0);

        return textureObjectIds[0];
    }
```

1. 首先使用 `glGenTextures` 创建纹理 ID。
2. 如果创建失败，则使用 `glDeleteTextures` 删除并退出。
3. 创建成功之后，使用 `glBindTexture` 函数将纹理 ID 和纹理目标绑定。
4. 之后会设置纹理在缩小和放大情况下的过滤方式。
5. 再使用 `texImage2D` 将纹理目标和 Bitmap 图片绑定。
6. 使用 `glGenerateMipmap` 函数生成多级渐远纹理和 MIP 纹理贴图。
7. 再使用 `glBindTexture`函数解除绑定。


### glBindTexture 函数

这里要重点说一下 glBindTexture 函数。

它的作用是绑定纹理名到指定的当前活动纹理单元，当一个纹理绑定到一个目标时，目标纹理单元先前绑定的纹理对象将被自动断开。纹理目标默认绑定的是 0 ，所以要断开时，也再将纹理目标绑定到 0 就好了。

所以在代码的最后调用了 `glBindTexture(GL_TEXTURE_2D, 0)` 来解除绑定。

当一个纹理被绑定时，在绑定的目标上的 OpenGL 操作将作用到绑定的纹理上，并且，对绑定的目标的查询也将返回其上绑定的纹理的状态。

也就是说，这个纹理目标成为了被绑定到它上面的纹理的别名，而纹理名称为 0 则会引用到它的默认纹理。所以，当后续对纹理目标调用 `glTexParameteri` 函数设置过滤方式，其实也是对纹理设置的过滤方式。


### 绑定纹理中的值

创建并且设置了纹理着色器ID之后，就需要绑定并设置在着色器语言中的变量了。

```java
		// 绑定着色器脚本中的变量
        uTextureUnitAttr = glGetUniformLocation(mProgram, U_TEXTURE_UNIT)
		mTextureId = TextureHelper.loadTexture(mContext,R.drawable.texture)
		// 激活纹理单元
        glActiveTexture(GL_TEXTURE0)
        // 绑定纹理目标
        glBindTexture(GL_TEXTURE_2D, mTextureId)
        // 给片段着色器中的采样器变量 sample2D 赋值
        glUniform1i(uTextureUnitAttr, 0)

```

在着色器脚本中定义了 `uniform` 类型的采样器变量 `sampler2D`，在上层的应用代码需要将它绑定并赋值。而 `varying` 类型的变量由顶点着色器传过去，不需要另外赋值了。


接下来要使用 glActiveTexture 函数激活纹理单元。在一个系统中，纹理单元的数据是有限的，在源码中从 GL_TEXTURE0 到 GL_TEXTURE31 共定义了三十二个纹理单元，但具体数量根据机型而定。

通过 GL_MAX_COMBINED_TEXTURE_IMAGE_UNITS 常量可以查询到。

```java
   var intBuffer:IntBuffer = IntBuffer.allocate(1)
   glGetIntegerv(GL_MAX_COMBINED_TEXTURE_IMAGE_UNITS,intBuffer)
   LogUtil.d("max combined texture image units " + intBuffer[0])
```

激活了纹理单元，还需要再绑定纹理目标。

一个纹理单元包含了多个类型纹理目标，如：GL_TEXTURE_1D、GL_TEXTURE_2D、CUBE_MAP 等等。

因为纹理单元是纹理的一个别名，所以对纹理单元所做的操作，都相当于对纹理操作的。把一些对纹理所做的操作提取到函数里，最后再加载纹理，并绑定到纹理目标上。

使用`glUniform1i`函数为采样器进行赋值为 0 ，这是和激活纹理单元相对应的。因为激活的纹理单元为 0 ，所以赋值也是为 0 。如果这里不一致，直接就看不到任何东西了。


### 实际效果

当绑定并设置好片段着色器中的值之后，接下来的流程就和绘制基本图形一样了。





具体的绘制操作都在片段着色器里面定义了，而在上层代码中就不用花费很多心思了，在顶点着色器不变的情况下，甚至可以只改变片段着色器的值来绘制不同的纹理效果。


## 总结 & 名词混淆点

在上面既是纹理单元又是纹理目标的很容易搞混，梳理一下概念：

形如 GL_TEXTURE0、GL_TEXTURE1、GL_TEXTURE2 的就是纹理单元，一台机子上纹理单元数量是有限的，依具体机型而定。而 glActiveTexture 则是激活具体的纹理单元。

一个纹理单元又包含多个类型的纹理目标，有：GL_TEXTURE_1D、GL_TEXTURE_2D、CUBE_MAP 等等。

通过 glGenTextures 函数生成的 int 类型的值就是纹理，通过 glBindTexture 函数将纹理目标和纹理绑定后，对纹理目标所进行的操作都反映到对纹理上。

纹理目标需要通过 texImage2D 函数附加上 Bitmap 位图。


具体代码详情，可以参考我的 Github 项目：

[https://github.com/glumes/AndroidOpenGLTutorial](https://github.com/glumes/AndroidOpenGLTutorial)

## 参考

1. http://blog.csdn.net/opengl_es/article/details/19852277
2. http://blog.csdn.net/artisans/article/details/76695614









