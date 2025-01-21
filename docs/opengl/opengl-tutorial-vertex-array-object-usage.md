---
title: "《OpenGL ES 3.x 游戏开发》 顶点数组对象 VAO 的使用"
date: 2018-07-18T23:16:22+08:00
subtitle: "学习笔记内容摘录"
draft: false
categories: ["OpenGL"]
tags: ["OpenGL"]
toc: true
 
slug: "opengl-tutorial-vertex-array-object-usage"
---


在之前使用 [OpenGL 顶点缓冲区 VBO 的使用](https://glumes.com/post/opengl/opengl-tutorial-vertex-buffer-object-usage/) 为顶点坐标、纹理坐标分别绑定了顶点缓冲区，并且在 onDrawFrame 方法里面也要分别为顶点坐标、纹理坐标指定数据。

这就存在了一些重复的操作。

在 OpenGL ES 3.0 可以使用顶点数组对象来解决这一问题。

<!--more-->

> 顶点数组对象又叫做 VAO （Vertex Array Object）。


顶点数组对象的主要功能就是将绘制的一个物体时所需的对应于不同方面的顶点缓冲及相关设置包装成一个整体，在绘制时直接使用顶点数组对象即可，不必分别使用每个顶点缓冲，这样可以进一步提高绘制效率。

## 方法介绍

顶点数组对象主要提供了 glGenVertexArrays 和 glBindVertexArray 方法。

### glGenVertexArrays 方法

glGenVertexArrays 方法主要用于创建新的顶点数组对象，方法签名如下：

```java
    public static native void glGenVertexArrays(
        int n,
        int[] arrays,
        int offset
    );
```

其中，参数 n 为要创建的顶点数组对象的数量，参数 arrays 用来存放创建的 n 个顶点数组对象的编号，参数 offset 为 arrays 数组的偏移量。


### glBindVertexArray 方法

创建顶点数组对象之后，就可以用 glBindVertexArray 方法绑定指定的顶点数组对象，以便对指定的顶点数组对象进行设置或使用，方法签名如上：

```java
    public static native void glBindVertexArray(
        int array
    );
```

其中，参数 array 为需要绑定的顶点数组对象的编号。

用 glBindVertexArray 方法绑定顶点数组对象之后，更改顶点数组状态的后续调用（如 glBindBuffer、glVertexAttribPointer、glEnableVertexAttribArray、glDisableVertexAttribArray 等）将影响绑定的顶点数组对象。

通过绑定不同的顶点数组对象，绘制时应用程序可以快速在不同顶点数组配置之间进行切换，大大提高了开发效率。


## 实践

下面是具体的实践，还是绘制如下的一张纹理图：


![顶点数组对象的使用](https://image.glumes.com/images/2019/04/27/WechatIMG37.jpg)

使用顶点数组对象 VA0 之前，还是分别为顶点坐标、纹理坐标生成缓冲区，并传入数据。

```java
		// 生成纹理缓冲区
        int[] bufferIds = new int[3];
        GLES30.glGenBuffers(3, bufferIds, 0);
        // 顶点缓冲
        mVertexBufferId = bufferIds[0];
        GLES30.glBindBuffer(GLES30.GL_ARRAY_BUFFER, mVertexBufferId);
        GLES30.glBufferData(GLES30.GL_ARRAY_BUFFER, vertices.length * 4, mVertexBuffer, GLES30.GL_STATIC_DRAW);
        GLES30.glBindBuffer(GLES30.GL_ARRAY_BUFFER, 0);
        // 纹理缓冲
        mTextureBufferId = bufferIds[1];
        GLES30.glBindBuffer(GLES30.GL_ARRAY_BUFFER, mTextureBufferId);
        GLES30.glBufferData(GLES30.GL_ARRAY_BUFFER, textures.length * 4, mTextureBuffer, GLES30.GL_STATIC_DRAW);
        GLES30.glBindBuffer(GLES30.GL_ARRAY_BUFFER, 0);

```

接下来就是使用顶点数组对象了。

```java
		// 生成顶点数组对象
        int[] vaoIds = new int[1];
        GLES30.glGenVertexArrays(1, vaoIds, 0);
        mVAOId = vaoIds[0];
        // 绑定顶点数组对象
        GLES30.glBindVertexArray(mVAOId);
		// 在绑定顶点数组对象之后 调用 glBindBuffer glVertexAttribPointer
		// glEnableVertexAttribArray，glDisableVertexAttribArray 等等
		// 将影响绑定的顶点数组对象
        GLES30.glBindBuffer(GLES30.GL_ARRAY_BUFFER, mVertexBufferId);
        GLES30.glEnableVertexAttribArray(maPositionHandle);
        GLES30.glVertexAttribPointer(maPositionHandle, 3, GLES30.GL_FLOAT, false, 3 * 4, 0);
        GLES30.glBindBuffer(GLES30.GL_ARRAY_BUFFER, 0);

        GLES30.glBindBuffer(GLES30.GL_ARRAY_BUFFER, mTextureBufferId);
        GLES30.glEnableVertexAttribArray(maTexCoorHandle);
        GLES30.glVertexAttribPointer(maTexCoorHandle, 2, GLES30.GL_FLOAT, false, 2 * 4, 0);
        GLES30.glBindBuffer(GLES30.GL_ARRAY_BUFFER, 0);
		// 绑定到默认的顶点数组对象
		  GLES30.glBindVertexArray(0);
```

这里要注意的是，调用了 glBindVertexArray 绑定顶点数组对象之后，我们调用的 glBindBuffer、glVertexAttribPointer、glEnableVertexAttribArray、glDisableVertexAttribArray 等方法都属于该顶点数组对象之内了，都会影响到这个顶点数组对象，直到绑定了默认的顶点数组对象。

在我们使用 顶点缓冲区 VBO 绘制时，要在 onDrawFrame 方法里面为每一个顶点坐标、纹理坐标调用 glBindBuffer 、glVertexAttribPointer、glEnableVertexAttribArray 方法，而使用了 顶点数组对象 VA0 之后，就不需要这么麻烦了，因为相当于顶点数组对象把它们都接管了，直接绑定顶点数组对象就好了。

使用 顶点数组对象 VAO 绘制的代码如下：

```java
		// 绑定顶点数组对象
        GLES30.glBindVertexArray(mVAOId);
        GLES30.glBindBuffer(GLES30.GL_ELEMENT_ARRAY_BUFFER, mIndicesBufferId);
        GLES30.glDrawElements(GLES30.GL_TRIANGLES, vCount, GLES30.GL_UNSIGNED_BYTE, 0);
        GLES30.glBindBuffer(GLES30.GL_ELEMENT_ARRAY_BUFFER, 0);
		// 解除绑定顶点数组对象
        GLES30.glBindVertexArray(0);
```

通过这种方式相当于替代了那些繁琐的方法，并且可以直接通过更改绑定的顶点数组对象 ID 去更改绘制的内容，简单高效地实现切换顶点数据。


关于具体的代码实现，可以参考我的 Github 项目：

> [https://github.com/glumes/AndroidOpenGLTutorial](https://github.com/glumes/AndroidOpenGLTutorial)

## 参考

1. 《OpenGL ES 3.x 游戏开发》



