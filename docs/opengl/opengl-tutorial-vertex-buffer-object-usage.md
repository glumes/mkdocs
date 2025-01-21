---
title: "《OpenGL ES 3.x 游戏开发》顶点缓冲区 VBO 的使用"
date: 2018-07-17T22:28:37+08:00
subtitle: "学习笔记内容摘录"
draft: false
categories: ["OpenGL"]
tags: ["OpenGL"]
toc: true
 
slug: "opengl-tutorial-vertex-buffer-object-usage"
---

在之前的绘制过程中，首先都需要将物体的顶点数据保存在内存中，然后 `glDrawArrays` 或 `glDrawElements` 绘制前，将顶点数据送入到显存中，这样会存在 I/O 开销较大的问题，性能也不够好。

可以将顶点数据存放在顶点缓冲区中，就不需要在每次绘制前把顶点数据复制进显存，而是在初始化顶点缓冲区对象时一次性将顶点数据送入显存，每次绘制时直接使用显存中的数据，可以大大提高渲染性能。

<!--more-->

## 顶点缓冲区

在 OpenGL ES  中支持两种类型的顶点缓冲区对象，分别是数组缓冲区对象（Array Buffer）和 元素数组缓冲区对象（Element Array Buffer）。

*	数组缓冲区对象对应的类型为 GL_ARRAY_BUFFER，一般用于存放待绘制物体的顶点相关数据，如顶点坐标、纹理坐标、法向量等。
*	元素数组缓冲区对象对应的类型为 GL_ELEMENT_ARRAY_BUFFER ，一般用于存放图元的组装索引数据，在使用索引法进行绘制时使用。

> 使用顶点缓冲区的好处就在于顶点数量非常多的时候，可以一次性将大量顶点数据送入到显存中。

关于顶点缓冲区，又有另一种说法，叫做 `VBO`，其实就是 `Vertex Buffer Object`的缩写啦。

## 缓冲区对象的常用操作方法

下面介绍一下关于顶点缓冲区的常用方法。

### 创建缓冲区对象 -- glGenBuffers

glGenBuffers 方法用于创建缓冲对象，是在使用每个自定义缓冲前都需要调用的。

```java
	// 方法一
    public static native void glGenBuffers(
        int n,
        java.nio.IntBuffer buffers
    );
    // 方法二
    public static native void glGenBuffers(
        int n,
        int[] buffers,
        int offset
    );
```

1. 在方法一中，参数 n 为需要创建的缓冲区数量，参数 buffers 为用于存放创建的 n 个缓冲区编号的 IntBuffer 。
2. 在方法二中，参数 n 为需要创建的缓冲区数量，参数 buffers 为用于存放创建的 n 个缓冲区编号的数组，参数 offset 为 buffers 数组所需的偏移量。


### 绑定缓冲区对象 -- glBindBuffer

glBindBuffer 方法用于绑定当前缓冲区对象，第一次调用 glBindBuffer 方法绑定缓冲区对象时，缓冲区对象以默认状态分配；如果分配成功，则分配的对象绑定为当前缓冲区对象。

```java
    public static native void glBindBuffer(
        int target,
        int buffer
    );
```

其中，参数 target 用于描述需绑定的缓冲区类型，参数 buffer 为需要绑定的缓冲区编号。

target 值如下：

|target 值|缓冲区类型|
|---|---|
|GL_ARRAY_BUFFER|数组缓冲|
|GL_ELEMENT_ARRAY_BUFFER|元素数组缓冲|
|GL_COPY_READ_BUFFER|复制只读缓冲|
|GL_COPY_WRITE_BUFFER|复制可写缓冲|
|GL_PIXEL_PACK_BUFFER|像素打包缓冲|
|GL_PIXEL_UNPACK_BUFFER|像素解包缓冲|
|GL_TRANSFORM_FEEDBACK_BUFFER|变换反馈缓冲|
|GL_UNIFORM_BUFFER|一致变量缓冲|



### 向缓冲区送入数据 -- glBufferData 和 glBufferSubData

glBufferData 方法一般用于向指定缓冲中送入数据，也可以用于对指定缓冲进行相关存储空间初始化。

```java
  public static native void glBufferData(
        int target,
        int size,
        java.nio.Buffer data,
        int usage
    );
```

其中，target 用于描述指定的缓冲区类型，如上表所示；参数 size 用于给出缓冲区的大小（单位为字节），参数 data 为需要送入缓冲的数据，若没有数据要送入缓冲区，其值可以为 null；参数 usage 用于指定缓冲区的用途，可取的值如下表：

|缓冲区用途可选参数值|参数说明|
|---|---|
|GL_STATIC_DRAW|在绘制时，缓冲区对象数据可以被修改一次，使用多次|
|GL_STATIC_READ|在 OpenGL ES 中读回的数据，缓冲区对象数据可以被修改一次，使用多次，且该数据可以冲应用程序中查询|
|GL_STATIC_COPY|从 OpenGL ES 中读回的数据，缓冲区对象数据可以被修改一次，使用多次，该数据将直接作为绘制图元或者指定图像的信息来源|
|GL_DYNAMIC_DRAW|在绘制时，缓冲区对象数据可以被重复修改、使用多次|
|GL_DYNAMIC_READ|从 OpenGL ES 中读回的数据，缓冲区对象数据可以被重复修改、使用多次，且该数据可以从应用程序中查询|
|GL_DYNAMIC_COPY|从 OpenGL ES 中读回的数据，缓冲区对象数据可以被重复修改、使用多次，该数据将直接作为绘制图元或者指定图像的信息来源|
|GL_STREAM_DRAW|在绘制时，缓冲区对象数据可以被修改一次，使用少数几次|
|GL_STREAM_READ|从 OpenGL ES 中读回的数据，缓冲区对象数据可以被修改一次，使用少数几次，且该数据可以从应用程序中查询|
|GL_STREAM_COPY|从 OpenGL ES 中读回的数据，缓冲区对象数据可以被修改一次，使用少数几次，且该数据将直接作为绘制图元或者指定图像的信息来源|


glBufferSubData 方法一般用于向指定缓冲中送入部分数据进行初始化或者更新，方法如下：

```java
    public static native void glBufferSubData(
        int target,
        int offset,
        int size,
        java.nio.Buffer data
    );
```

其实，参数 target 还是用于描述指定的缓冲区类型，参数 offset 用于给出缓冲区被修改的数据的起始内存偏移量，参数 size 用于给出缓冲区中数据被修改的字节数，参数 data 为需要送入缓冲的数据。

通过 glBufferSubData 方法，可以做到更改指定范围内的缓冲区数据。

### 删除缓冲区对象 -- glDeleteBuffers

glDeteleBuffers 方法用于删除指定的缓冲区对象，其方法签名如下：

```java
	// 方法一
    public static native void glDeleteBuffers(
        int n,
        java.nio.IntBuffer buffers
    );
    // 方法二
    public static native void glDeleteBuffers(
        int n,
        int[] buffers,
        int offset
    );
```


其中，第一个方法有两个参数，参数 n 为将要被删除的缓冲区对象数量，参数 buffers 为存储了 n 个要删除的缓冲区编号的 IntBuffer 。

第二个方法有三个参数，参数 n 为将要被删除的缓冲区对象数量，参数 buffers 为存储了 n 个要删除缓冲区编号的数组，参数 offset 为数组所需的偏移量。

### 缓冲区查询 -- glGetBufferParameteriv

可以通过 glGetBufferParameteriv 方法查询指定缓冲区的信息，其方法签名如下：

```java
    public static native void glGetBufferParameteriv(
        int target,
        int pname,
        int[] params,
        int offset
    );
```

其中，参数 target 用于描述指定的缓冲区类型，参数 pname 为要查询的信息项目，具体项目见下表，参数 params 用于存放查询结果，offset 代表查询偏移量。

|缓冲区查询信息|说明|
|---|---|
|GL_BUFFER_SIZE|缓冲区以字节计的大小|
|GL_BUFFER_USAGE|缓冲区用途|
|GL_BUFFER_MAPPED|是否为映射缓冲区|
|GL_BUFFER_ACCESS_FLAGS|缓冲区访问标志|
|GL_BUFFER_MAP_LENGTH|缓冲区映射长度|
|GL_BUFFER_MAP_OFFSET|缓冲区映射偏移量|


## 实践

前面说了那么理论，那么针对数组缓冲区 Array Buffer 和元素数组缓冲区 Element Array Buffer 分别进行实践，实现的效果都是绘制一个带纹理的矩形。

如下：


![顶点缓冲区的使用](https://image.glumes.com/images/2019/04/27/WechatIMG37.jpg)


### 数组缓冲区使用


对于 Array Buffer  数组缓冲区，它可以存放顶点相关数据，比如顶点坐标、纹理坐标、法向量等，最后采用 glDrawArrays 方法来绘制，

首先，定义好顶点坐标：

```java
        float vertices[] =
                {
                        -width / 2, height / 2, 0,
                        -width / 2, -height / 2, 0,
                        width / 2, height / 2, 0,

                        -width / 2, -height / 2, 0,
                        width / 2, -height / 2, 0,
                        width / 2, height / 2, 0
                };
```
然后定义好纹理坐标：

```java
        float textures[] =
                {
                        0f, 0f, 0f, 1f, 1f, 0f,
                        0f, 1f, 1f, 1f, 1f, 0f,
                };
```

接下来就是创建顶点坐标和纹理坐标的缓存数据，通过 `ByteBuffer` 进行转换得到 `FloatBuffer`。

完成准备工作接下来就是创建对应的顶点缓冲区了。

```java
		// 指定缓存数量并生成
        int[] bufferIds = new int[3];
        GLES20.glGenBuffers(3, bufferIds, 0);

        // 创建顶点坐标缓冲，绑定并传递数据
        mVertexBufferId = bufferIds[0];
        GLES20.glBindBuffer(GLES20.GL_ARRAY_BUFFER, mVertexBufferId);
        GLES20.glBufferData(GLES20.GL_ARRAY_BUFFER, vertices.length * 4, mVertexBuffer, GLES20.GL_STATIC_DRAW);
        // 解除绑定
        GLES20.glBindBuffer(GLES20.GL_ARRAY_BUFFER, 0);

        // 创建纹理坐标缓冲，绑定并传递数据
        mTextureBufferId = bufferIds[1];
        GLES20.glBindBuffer(GLES20.GL_ARRAY_BUFFER, mTextureBufferId);
        GLES20.glBufferData(GLES20.GL_ARRAY_BUFFER, textures.length * 4, mTextureBuffer, GLES20.GL_STATIC_DRAW);
        // 解除绑定
        GLES20.glBindBuffer(GLES20.GL_ARRAY_BUFFER, 0);

```

接下来就是绘制的过程：

```java
		// 绑定到顶点坐标数据缓冲
        GLES20.glBindBuffer(GLES20.GL_ARRAY_BUFFER, mVertexBufferId);
        // 启用顶点坐标数据数组
        GLES20.glEnableVertexAttribArray(maPositionHandle);
        // 指定顶点位置数据使用对应缓冲
        GLES20.glVertexAttribPointer(maPositionHandle, 3, GLES20.GL_FLOAT, false, 3 * 4, 0);
        GLES20.glBindBuffer(GLES20.GL_ARRAY_BUFFER, 0);
		
		// 绑定到纹理坐标数据缓冲
        GLES20.glBindBuffer(GLES20.GL_ARRAY_BUFFER, mTextureBufferId);
        // 启用纹理坐标数据数组
        GLES20.glEnableVertexAttribArray(maTexCoorHandle);
        // 指定纹理位置数据使用对应缓冲
        GLES20.glVertexAttribPointer(maTexCoorHandle, 2, GLES20.GL_FLOAT, false, 2 * 4, 0);
        GLES20.glBindBuffer(GLES20.GL_ARRAY_BUFFER, 0);
		// 执行绘制
	    GLES20.glDrawArrays(GLES20.GL_TRIANGLES, 0, vCount);
```


在绘制时会使用到 `glVertexAttribPointer` 函数，这里有个细节差别，它和之前使用时参数不一样了。

### 元素数组缓冲区使用

使用 元素数组缓冲区 Element Array Buffer 就得采用 glDrawElement 的方式进行绘制，先要更改顶点坐标数据和纹理坐标数据，然后定义顶点索引数组，然后为索引数组生成对应的 元素数组缓冲区。

```java
		// 索引数组
        byte[] indices = {
                1, 2, 3,
                0, 1, 2,
        };
```

为索引数组生成对应的 元素数组缓冲区。

```java
     // 索引缓冲
        mIndicesBufferId = bufferIds[2];
        GLES20.glBindBuffer(GLES20.GL_ELEMENT_ARRAY_BUFFER, mIndicesBufferId);
        GLES20.glBufferData(GLES20.GL_ELEMENT_ARRAY_BUFFER, indices.length, mIndicesBuffer, GLES20.GL_STATIC_DRAW);
        GLES20.glBindBuffer(GLES20.GL_ELEMENT_ARRAY_BUFFER, 0);
```

另外在绘制时，也要绑定元素数组缓冲区后进行绘制：

```java
 GLES20.glBindBuffer(GLES20.GL_ELEMENT_ARRAY_BUFFER, mIndicesBufferId);
 GLES20.glDrawElements(GLES20.GL_TRIANGLES, vCount, GLES20.GL_UNSIGNED_BYTE, 0);
```

## 小结

对于博客的例子，一个纹理矩形只有四个顶点，体现不出顶点数组缓存的优势，多余大量点的情况下，这种优势还是挺明显的，尤其是存在要更改顶点数据的情况下，直接使用 glBufferSubData 方法去更改，而不是在重新传入新的顶点坐标数据。

关于具体的代码实现，可以参考我的 Github 项目：

> [https://github.com/glumes/AndroidOpenGLTutorial](https://github.com/glumes/AndroidOpenGLTutorial)

## 参考

1. 《OpenGL ES 3.x 游戏开发》

