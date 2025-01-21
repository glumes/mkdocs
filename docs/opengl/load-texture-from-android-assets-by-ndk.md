---
title: "Android NDK 开发 —— 从 Assets 文件夹加载图片并上传纹理"
date: 2019-05-14T23:16:45+08:00
subtitle: ""
slug: "load-texture-from-android-assets-by-ndk"
tags: ["JNI"]
categories: ["OpenGL"]
toc: true
 
draft: false
original: true
---

在 OpenGL 开发中，我们要渲染一张图片，通常先是得到一张图片对应的 Bitmap ,然后将该 Bitmap 作为纹理上传到 OpenGL 中。在 Android 中有封装好的 `GLUtils` 类的 `texImage2D` 方法供我们调用。

```java
    public static void texImage2D(int target, int level, int internalformat,
            Bitmap bitmap, int type, int border)
```

该方法的底层原理实际上也是解析了该 Bitmap ，得到了 Bitmap 所有的像素数据，类似于 Android NDK 关于 Bitmap 操作的 `AndroidBitmap_lockPixels` 方法，如果你不太了解该方法，可以参考这篇文章：[Android JNI 之 Bitmap 操作](https://glumes.com/post/android/android-jni-bitmap-operation/)。

得到了所有像素数据之后，实际最终还是调用了 OpenGL 的 `glTexImage2D` 来实现纹理上传。当然，如果可以直接得到所有数据，也不需要走解析 Bitmap 这一步了，这种场景最常见的就是把相机作为输入了。

<!--more-->

接下来我们会通过 Android NDK 开发中去渲染一张图片，步骤还是如上，从图像解析到纹理上传，不同的是我们将会解析 Assets 文件夹中的图片，而不是一张已经保存在手机 SDCard 上的图片。

相比于前者，SDCard 上的图片已经有了绝对地址了，直接把地址传到 `stb_image` 库就可以完成解析了（参考之前的文章 []()），而 `Assets` 文件夹的内容在手机上可没有绝对地址哦，不信你仔细回想，可曾在看到过 APK 安装后 Assets 文件夹对应的内容？


一开始陷入了误区，想着怎么去获得文件的绝对地址，看到了 AssetManager 的 AAsset_openFileDescriptor 方法以为拿到文件描述符就万事大吉了，结果打印的地址是这样的 `/proc/9941/fd/79` ，这基本的不可用了。

换个思路，在 Java 中去加载 Assets 目录下的图片：

```java
InputStream is = getAssets().open(fileName); 
```

通过 AssertManager 的 open 方法直接拿到文件的输入流了。

而在 NDK 开发中同样的方式是行不通的，这里要采用另外一种方式，但其实意思都差不多的：

```cpp
    // NDK 中是 AssetManager
    AAssetManager *mgr = AAssetManager_fromJava(env, assetManager);
    // 打开 Asset 文件夹下的文件
    AAsset *pathAsset = AAssetManager_open(mgr, assetPath, AASSET_MODE_UNKNOWN);
    // 得到文件的长度
    off_t assetLength = AAsset_getLength(pathAsset);
    // 得到文件对应的 Buffer
    unsigned char *fileData = (unsigned char *) AAsset_getBuffer(pathAsset);
    // stb_image 的方法，从内存中加载图片
    unsigned char *contnet = stbi_load_from_memory(fileData, assetLength, &w, &h, &n, 0);
```


NDK 中可拿不到像 Java 那样的输入流，但是可以通过 AssetManager 的 AAsset_getBuffer 或者是 AAsset_read 方法去获取文件内容。

看到上面那两个 API 基本就稳了，再配合 `stb_image` 介绍过的方法，`stbi_load_from_memory` 从内存中加载图片的像素数据，最后就是 glTexImage2D 方法实现纹理上传。

具体的代码实践参考项目：

> [https://github.com/glumes/InstantGLSL](https://github.com/glumes/InstantGLSL)

