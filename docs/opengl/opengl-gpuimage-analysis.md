---
title: "OpenGL 之 GPUImage 源码分析"
date: 2018-09-02T21:14:09+08:00
subtitle: ""
tags: ["OpenGL"]
categories: ["OpenGL"]
toc: true
 
draft: false
original: true
addwechat: true
slug: "opengl-gpuimage-analysis"
---

> 避免图片丢失，建议阅读微信原文：
> 
> https://mp.weixin.qq.com/s/2EeI91pu6y6Z7pnLPToXYA

[GPUImage](https://github.com/BradLarson/GPUImage) 是 iOS 上一个基于 OpenGL 进行图像处理的开源框架，后来有人借鉴它的想法实现了一个 Android 版本的 [GPUImage](https://github.com/CyberAgent/android-gpuimage) ，本文也主要对 Android 版本的 GPUImage 进行分析。


<!--more-->

## 概要

 在 GPUImage 中既有对图像进行处理的，也有对相机内容进行处理的，这里主要以相机处理为例进行分析。

大致会分为三个部分：

*	相机数据的采集
*	OpenGL 对图像的处理与显示
*	相机的拍摄

### 相机数据采集


> 相机数据采集实际上就是把相机的图像数据转换成 OpenGL 中的纹理。


在相机的业务开发中，会给相机设置 `PreviewCallback` 回调方法，只要相机处于预览阶段，这个回调就会被重复调用，返回当前预览帧的内容。

```java
	camera.setPreviewCallback(GPUImageRenderer.this);
    camera.startPreview();
```

默认情况下，相机返回的数据是 `NV21` 格式，也就是 `YCbCr_420_SP` 格式，而 OpenGL 使用的纹理是 RGB 格式，所以在每一次的回调方法中需要将 `YUV` 格式的数据转换成 `RGB` 格式数据。

```java
GPUImageNativeLibrary.YUVtoRBGA(data, previewSize.width, previewSize.height,
						     mGLRgbBuffer.array());
```

有了图像的 RGB 数据，就可以使用 `glGenTextures` 生成纹理，并用 `glTexImage2D` 方法将图像数据作为纹理。

另外，如果纹理已经生成了，当再有图像数据过来时，只需要更新数据就好了，无需重复创建纹理。

```java
	// 根据图像数据加载纹理
    public static int loadTexture(final IntBuffer data, final Size size, final int usedTexId) {
        int textures[] = new int[1];
        if (usedTexId == NO_TEXTURE) {
            GLES20.glGenTextures(1, textures, 0);
            GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, textures[0]);
	        // 省略部分代码
            GLES20.glTexImage2D(GLES20.GL_TEXTURE_2D, 0, GLES20.GL_RGBA, size.width, size.height,
                    0, GLES20.GL_RGBA, GLES20.GL_UNSIGNED_BYTE, data);
        } else {
            GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, usedTexId);
            // 更新纹理数据就好，无需重复创建纹理
            GLES20.glTexSubImage2D(GLES20.GL_TEXTURE_2D, 0, 0, 0, size.width,
                    size.height, GLES20.GL_RGBA, GLES20.GL_UNSIGNED_BYTE, data);
            textures[0] = usedTexId;
        }
        return textures[0];
    }
```

通过在 `PreviewCallback` 回调方法中的操作，就完成了将图像数据转换为 OpenGL 的纹理。

接下来就是如何将纹理数据进行处理，并且显示到屏幕上。

在相机数据采集中，还有一些小的细节问题，比如相机前置与后置摄像头的左右镜像翻转问题。

对于前置摄像头，再把传感器内容作为纹理显示时，前置摄像头要做一个左右的翻转处理，因为我们看到的是一个镜像内容，符合正常的自拍流程。

在 GPUImage 的 TextureRotationUtil 类中有定义了纹理坐标，这些纹理坐标系的原点不是位于左下角进行定义的，而是位于左上角。

如果以左下角为纹理坐标系的坐标原点，那么除了要将纹理坐标向右顺时针旋转 90° 之外，还需要进行上下翻转才行，至于为什么要向右顺时针旋转 90° ，参考这篇文章：


[Android 相机开发中的尺寸和方向问题](https://glumes.com/post/android/android-camera-aspect-ratio--and-orientation/)

当我们把纹理坐标以左上角为原点，并相对于顶点坐标顺时针旋转 90 ° 之后，才能够正常的显示图像：

```java
	// 顶点坐标
    static final float CUBE[] = {
            -1.0f, -1.0f,
            1.0f, -1.0f,
            -1.0f, 1.0f,
            1.0f, 1.0f,
    };
    // 以左上角为原点，并相对于顶点坐标顺时针旋转 90° 后的纹理坐标
    public static final float TEXTURE_ROTATED_90[] = {
            1.0f, 1.0f,
            1.0f, 0.0f,
            0.0f, 1.0f,
            0.0f, 0.0f,
    };
```

## 图像处理与显示

在有了纹理之后，需要明确的是，这个纹理就是相机采集到的图像内容，我们要将纹理绘制到屏幕上，实际上是绘制一个矩形，然后纹理是贴在这个矩形上的。

所以，这里可以回顾一下 OpenGL 是如何绘制矩形的，并且将纹理贴到矩形上。

[OpenGL 学习系列---纹理](https://glumes.com/post/opengl/opengl-tutorial-texture/)

在 GPUImage 中，`GPUImageFilter` 类就完成了上述的操作，它是 OpenGL 中所有滤镜的基类。

### 解析 GPUImageFilter 的代码实现：

在 GPUImageFilter 的构造方法中会确定好需要使用的顶点着色器和片段着色器脚本内容。

在 `init` 方法中会调用 `onInit` 方法和 `onInitialized` 方法。

*	 `onInit` 方法会创建 OpenGL 中的 Program，并且会绑定到着色器脚本中声明的 `attribute` 和 `uniform` 变量字段。
*	 `onInitialized` 方法会给一些 `uniform` 字段变量赋值，在 GPUImageFilter 类中还对不同类型的变量赋值进行了对应的方法，比如对 `float` 变量：

```java
    protected void setFloat(final int location, final float floatValue) {
        runOnDraw(new Runnable() {
            @Override
            public void run() {
                GLES20.glUniform1f(location, floatValue);
            }
        });
    }
```

在 onDraw 方法中就是执行具体的绘制了，在绘制的时候会执行 `runPendingOnDrawTasks` 方法，这是因为我们在 `init` 方法去中给着色器语言中的变量赋值，并没有立即生效，而是添加到了一个链表中，所以需要把链表中的任务执行完了才接着执行绘制。

```java
    public void onDraw(final int textureId, final FloatBuffer cubeBuffer,
                       final FloatBuffer textureBuffer) {
        GLES20.glUseProgram(mGLProgId);
        // 执行赋值的任务
        runPendingOnDrawTasks();
		// 顶点和纹理坐标数据
        GLES20.glVertexAttribPointer(mGLAttribPosition, 2, GLES20.GL_FLOAT, false, 0, cubeBuffer);
        GLES20.glEnableVertexAttribArray(mGLAttribPosition);
        GLES20.glVertexAttribPointer(mGLAttribTextureCoordinate, 2, GLES20.GL_FLOAT, false, 0, textureBuffer);
        GLES20.glEnableVertexAttribArray(mGLAttribTextureCoordinate);
		// 在绘制前的最后一波操作
        onDrawArraysPre();
        // 最终绘制
        GLES20.glDrawArrays(GLES20.GL_TRIANGLE_STRIP, 0, 4);
    }
```

在绘制时，还需要给顶点坐标赋值，给纹理坐标赋值，GPUImageFilter 并没有去管理顶点坐标和纹理坐标，而是通过传递参数的形式，这样就不用去处理在前置摄像头与后置前摄像头、手机竖立放置与横屏放置时的关系了。

在执行具体的 `glDrawArrays` 方法之前，还提供了一个 `onDrawArraysPre` 方法，在这个方法里面还可以执行绘制前的最后一波操作，在某些滤镜的实现中有用到了。

最后才是 `glDrawArrays` 方法去完成绘制。

当我们不需要 GPUImageFilter 进行绘制时，需要将它销毁掉，在 destroy 方法去进行销毁，并且提供 onDestory 方法去为某些滤镜提供自定义的销毁。

```java
    public final void destroy() {
        mIsInitialized = false;
        GLES20.glDeleteProgram(mGLProgId);
        onDestroy();
    }

    public void onDestroy() {
    }
```

在 GPUImageFilter 方法中定义了片段着色器脚本，这个脚本是将图像内容原样贴到了矩形上，并没有做特殊的图像处理操作。

而其他滤镜中，更改了着色器脚本，也就会对图像进行其他的处理，在整个 GPUImage 项目中，最精华的也就是那些着色器脚本内容了，如何通过着色器去做图像处理又是一门高深的学问了~~~


### 解析 GPUImageFilterGroup 的代码实现

当想要对图像进行多次处理时，就得考虑使用 GPUImageFilterGroup 了。

GPUImageFilterGroup 继承自 GPUImageFilter， 顾名思义就是一系列 GPUImageFilter 滤镜的组合，可以把它类比为 ViewGroup ，ViewGroup 即可以包含 View ，也可以包含 ViewGroup ，同样 GPUImageFilterGroup 即可以包含 GPUImageFilter，也可以包含 GPUImageFilterGroup。

在用 GPUImageFilterGroup 进行绘制时，需要把所有的滤镜内容都进行一遍绘制，而对于 GPUImageFilterGroup 包含 GPUImageFilterGroup 的情况，就需要把子 GPUImageFilterGroup 内的滤镜内容拆分出来，最终是用 `mMergedFilters` 变量表示所有非 GPUImageFilterGroup 类型的 GPUImageFilter 。

```java
	// 拿到所有非 GPUImageFilterGroup 的 GPUImageFilter
    public void updateMergedFilters() {
        List<GPUImageFilter> filters;
        for (GPUImageFilter filter : mFilters) {
	        // 如果滤镜是 GPUImageFilterGroup 类型，就把它拆分了
            if (filter instanceof GPUImageFilterGroup) {
	            // 递归调用 updateMergedFilters 方法去拆分
                ((GPUImageFilterGroup) filter).updateMergedFilters();
                // 拿到所有非 GPUImageFilterGroup 的 GPUImageFilter
                filters = ((GPUImageFilterGroup) filter).getMergedFilters();
                if (filters == null || filters.isEmpty())
                    continue;
                // 把 GPUImageFilter 添加到 mMergedFilters 中
                mMergedFilters.addAll(filters);
                continue;
            }
            // 如果是非 GPUImageFilterGroup 直接添加了
            mMergedFilters.add(filter);
        }
    }
```

在 GPUImageFilterGroup 执行具体的绘制之前，还创建了和滤镜数量一样多的 FrameBuffer 帧缓冲和 Texture 纹理。

```java
		// 遍历时，选择 mMergedFilters 的长度，因为 mMergedFilters 里面才是保存的所有的 滤镜的长度。
        if (mMergedFilters != null && mMergedFilters.size() > 0) {
            size = mMergedFilters.size();
            // FrameBuffer 帧缓冲数量
            mFrameBuffers = new int[size - 1];
            // 纹理数量
            mFrameBufferTextures = new int[size - 1];

            for (int i = 0; i < size - 1; i++) {
				// 生成 FrameBuffer 帧缓冲
                GLES20.glGenFramebuffers(1, mFrameBuffers, i);
                // 生成纹理
                GLES20.glGenTextures(1, mFrameBufferTextures, i);
                GLES20.glBindTexture(GLES20.GL_TEXTURE_2D, mFrameBufferTextures[i]);
                GLES20.glTexImage2D(GLES20.GL_TEXTURE_2D, 0, GLES20.GL_RGBA, width, height, 0,
                        GLES20.GL_RGBA, GLES20.GL_UNSIGNED_BYTE, null);
                // 省略部分代码
                GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, mFrameBuffers[i]);
                // 纹理绑定到帧缓冲上
                GLES20.glFramebufferTexture2D(GLES20.GL_FRAMEBUFFER, GLES20.GL_COLOR_ATTACHMENT0,
                        GLES20.GL_TEXTURE_2D, mFrameBufferTextures[i], 0);
                // 省略部分代码
            }
        }
```

如果对 FrameBuffer 的使用不熟悉的话，请参考这篇文章：


[OpenGL 之 帧缓冲 使用实践](https://glumes.com/post/opengl/opengl-framebuffer-object-usage/)


```java
        if (mMergedFilters != null) {
            int size = mMergedFilters.size();
            // 相机原始图像转换的纹理 ID
            int previousTexture = textureId;
            for (int i = 0; i < size; i++) {
                GPUImageFilter filter = mMergedFilters.get(i);
                boolean isNotLast = i < size - 1;
                // 如果不是最后一个滤镜，绘制到 FrameBuffer 上，如果是最后一个，就绘制到了屏幕上
                if (isNotLast) {
                    GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, mFrameBuffers[i]);
                    GLES20.glClearColor(0, 0, 0, 0);
                }
				// 滤镜绘制代码
                if (i == 0) {
	                // 第一个滤镜绘制使用相机的原始图像纹理 ID 和参数传递过来的顶点以及纹理坐标
                    filter.onDraw(previousTexture, cubeBuffer, textureBuffer);
                } else if (i == size - 1) {
	                // 
                    filter.onDraw(previousTexture, mGLCubeBuffer, (size % 2 == 0) ? mGLTextureFlipBuffer : mGLTextureBuffer);
                } else {
	                // 中间的滤镜绘制在之前纹理基础上继续绘制，使用 mGLTextureBuffer 纹理坐标
                    filter.onDraw(previousTexture, mGLCubeBuffer, mGLTextureBuffer);
                }

                if (isNotLast) {
	                // 绑定到屏幕上
                    GLES20.glBindFramebuffer(GLES20.GL_FRAMEBUFFER, 0);
                    previousTexture = mFrameBufferTextures[i];
                }
            }
        }
```


在执行具体的绘制时，只要不是最后一个滤镜，那么就会先绑定到 FrameBuffer 上，然后在 FrameBuffer 上进行绘制，这时绘制是绘制到了 FrameBuffer 绑定的纹理上，绘制结束后再接着解绑，绑定到屏幕上。

如果是最后一个滤镜，那么就不用绑定到 FrameBuffer 上了，直接绘制到屏幕上即可。

在这里有个细节，就是如下的代码：

```java
    filter.onDraw(previousTexture, mGLCubeBuffer, (size % 2 == 0) ? mGLTextureFlipBuffer : mGLTextureBuffer);
```

如果是最后一个滤镜，并且滤镜个数为偶数，则使用 mGLTextureFlipBuffer 的纹理坐标，否则使用 mGLTextureBuffer 的纹理坐标。

```java
// 对应的纹理坐标为 TEXTURE_NO_ROTATION
 mGLTextureBuffer.put(TEXTURE_NO_ROTATION).position(0);

// 对应的纹理坐标为 TEXTURE_NO_ROTATION，并且 true 的参数表示进行垂直上下翻转
float[] flipTexture = TextureRotationUtil.getRotation(Rotation.NORMAL, false, true);
mGLTextureFlipBuffer.put(flipTexture).position(0);
```

在第一个滤镜绘制时，使用的是参数传递过来的顶点坐标和纹理坐标，中间部分的滤镜使用的是 mGLTextureBuffer 纹理坐标，它对应的纹理坐标数组为 `TEXTURE_NO_ROTATION`。

在前面讲到过，GPUImage 的纹理坐标原点是位于左上角的，所以使用 `TEXTURE_NO_ROTATION` 的纹理坐标实质上是将图像进行了上下翻转，两次调用`TEXTURE_NO_ROTATION`纹理坐标时，又将图像复原了，这也就可以解释为什么滤镜个数为偶数时，需要使用 `mGLTextureFlipBuffer` 纹理坐标将图像再进行一次翻转，而 `mGLTextureBuffer` 纹理坐标不需要了。



当明白了 GPUImageFilter 和 GPUImageFilterGroup 的实现之后，再去看具体的 Renderer 的代码就明了多了。

在 `onSurfaceCreated` 和 `onSurfaceChanged` 方法中分别对滤镜进行初始化以及设定宽、高，在 onDrawFrame 方法中调用具体的绘制。

当切换滤镜时，先将上一个滤镜销毁掉，然后初始化新的滤镜并设定宽、高。

```java
				final GPUImageFilter oldFilter = mFilter;
                mFilter = filter;
                if (oldFilter != null) {
                    oldFilter.destroy();
                }
                mFilter.init();
                GLES20.glUseProgram(mFilter.getProgram());
                mFilter.onOutputSizeChanged(mOutputWidth, mOutputHeight);
```


## 图像拍摄保存处理

在 GPUImage 中相机的拍摄是调用 Camera 的 `takePicture` 方法，在该方法中返回相机采集的原始图像数据，然后再对该数据进行一遍滤镜处理后并保存。


调用的最后都是通过 `glReadPixels` 方法将处理后的图像读取出来，并保存为 Bitmap 。

```java
    private void convertToBitmap() {
        int[] iat = new int[mWidth * mHeight];
        IntBuffer ib = IntBuffer.allocate(mWidth * mHeight);
        mGL.glReadPixels(0, 0, mWidth, mHeight, GL_RGBA, GL_UNSIGNED_BYTE, ib);
        int[] ia = ib.array();
		// glReadPixels 读取的内容是上下翻转的，要处理一下
        for (int i = 0; i < mHeight; i++) {
            for (int j = 0; j < mWidth; j++) {
                iat[(mHeight - i - 1) * mWidth + j] = ia[i * mWidth + j];
            }
        }
        mBitmap = Bitmap.createBitmap(mWidth, mHeight, Bitmap.Config.ARGB_8888);
        mBitmap.copyPixelsFromBuffer(IntBuffer.wrap(iat));
    }
```


## 小结

对 GPUImage 的分析以及滤镜架构的设计大致就是这样了，这些都还不是它的精华啦，重要的还是它的那些着色器脚本，从那些着色器脚本中学会如果通过 `GLSL` 去实现图像处理算法。



