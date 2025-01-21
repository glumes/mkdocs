---
title: "Android JNI 之 Bitmap 操作"
date: 2018-07-25T13:36:50+08:00
subtitle: ""
draft: false
categories: ["Android"]
tags: ["NDK"]
toc: true
original: true
addwechat: true
slug: "android-jni-bitmap-operation"
 
---


在 Android 中通过 JNI 去操作 Bitmap。

<!--more-->

在 Android 通过 JNI 去调用 Bitmap，通过 CMake 去编 so 动态链接库的话，需要添加 jnigraphics 图像库。

```cmake
target_link_libraries( # Specifies the target library.
                       native-operation
                       jnigraphics
                       ${log-lib} )
```

在 Android 中关于 JNI Bitmap 的操作，都定义在 `bitmap.h` 的头文件里面了，主要就三个函数，明白它们的含义之后就可以去实践体会了。


### 检索 Bitmap 对象信息

AndroidBitmap_getInfo 函数允许原生代码检索 Bitmap 对象信息，如它的大小、像素格式等，函数签名如下：

```cpp
/**
 * Given a java bitmap object, fill out the AndroidBitmapInfo struct for it.
 * If the call fails, the info parameter will be ignored.
 */
int AndroidBitmap_getInfo(JNIEnv* env, jobject jbitmap,
                          AndroidBitmapInfo* info);
```

其中，第一个参数就是 JNI 接口指针，第二个参数就是 Bitmap 对象的引用，第三个参数是指向 AndroidBitmapInfo 结构体的指针。

AndroidBitmapInfo 结构体如下：

```cpp
/** Bitmap info, see AndroidBitmap_getInfo(). */
typedef struct {
    /** The bitmap width in pixels. */
    uint32_t    width;
    /** The bitmap height in pixels. */
    uint32_t    height;
    /** The number of byte per row. */
    uint32_t    stride;
    /** The bitmap pixel format. See {@link AndroidBitmapFormat} */
    int32_t     format;
    /** Unused. */
    uint32_t    flags;      // 0 for now
} AndroidBitmapInfo;
```

其中，width 就是 Bitmap 的宽，height 就是高，format 就是图像的格式，而 stride 就是每一行的字节数。

图像的格式有如下支持：

```cpp
/** Bitmap pixel format. */
enum AndroidBitmapFormat {
    /** No format. */
    ANDROID_BITMAP_FORMAT_NONE      = 0,
    /** Red: 8 bits, Green: 8 bits, Blue: 8 bits, Alpha: 8 bits. **/
    ANDROID_BITMAP_FORMAT_RGBA_8888 = 1,
    /** Red: 5 bits, Green: 6 bits, Blue: 5 bits. **/
    ANDROID_BITMAP_FORMAT_RGB_565   = 4,
    /** Deprecated in API level 13. Because of the poor quality of this configuration, it is advised to use ARGB_8888 instead. **/
    ANDROID_BITMAP_FORMAT_RGBA_4444 = 7,
    /** Alpha: 8 bits. */
    ANDROID_BITMAP_FORMAT_A_8       = 8,
};
```

如果 AndroidBitmap_getInfo 执行成功的话，会返回 0 ，否则返回一个负数，代表执行的错误码列表如下：

```cpp
/** AndroidBitmap functions result code. */
enum {
    /** Operation was successful. */
    ANDROID_BITMAP_RESULT_SUCCESS           = 0,
    /** Bad parameter. */
    ANDROID_BITMAP_RESULT_BAD_PARAMETER     = -1,
    /** JNI exception occured. */
    ANDROID_BITMAP_RESULT_JNI_EXCEPTION     = -2,
    /** Allocation failed. */
    ANDROID_BITMAP_RESULT_ALLOCATION_FAILED = -3,
};
```


### 访问原生像素缓存

AndroidBitmap_lockPixels 函数锁定了像素缓存以确保像素的内存不会被移动。

如果 Native 层想要访问像素数据并操作它，该方法返回了像素缓存的一个原生指针，

```cpp
/**
 * Given a java bitmap object, attempt to lock the pixel address.
 * Locking will ensure that the memory for the pixels will not move
 * until the unlockPixels call, and ensure that, if the pixels had been
 * previously purged, they will have been restored.
 *
 * If this call succeeds, it must be balanced by a call to
 * AndroidBitmap_unlockPixels, after which time the address of the pixels should
 * no longer be used.
 *
 * If this succeeds, *addrPtr will be set to the pixel address. If the call
 * fails, addrPtr will be ignored.
 */
int AndroidBitmap_lockPixels(JNIEnv* env, jobject jbitmap, void** addrPtr);
```

其中，第一个参数就是 JNI 接口指针，第二个参数就是 Bitmap 对象的引用，第三个参数是指向像素缓存地址的指针。

AndroidBitmap_lockPixels 执行成功的话返回 0 ，否则返回一个负数，错误码列表就是上面提到的。


### 释放原生像素缓存

对 Bitmap 调用完 AndroidBitmap_lockPixels 之后都应该对应调用一次 AndroidBitmap_unlockPixels  用来释放原生像素缓存。

当完成对原生像素缓存的读写之后，就应该释放它，一旦释放后，Bitmap Java 对象又可以在 Java 层使用了，函数签名如下：

```cpp
/**
 * Call this to balance a successful call to AndroidBitmap_lockPixels.
 */
int AndroidBitmap_unlockPixels(JNIEnv* env, jobject jbitmap);
```

其中，第一个参数就是 JNI 接口指针，第二个参数就是 Bitmap 对象的引用，如果执行成功返回 0，否则返回 1。


> 对 Bitmap 的操作，最重要的就是 AndroidBitmap_lockPixels 函数拿到所有像素的缓存地址，然后对每个像素值进行操作，从而更改 Bitmap 。



## 实践

通过对 Bitmap 进行旋转，上下翻转，左右镜像来体验 JNI 的开发。

效果如下：

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftl40fytkxj20u01hcdmh.jpg)


具体代码可以参考我的 Github 项目，欢迎 Star。

> [https://github.com/glumes/AndroidDevWithCpp](https://github.com/glumes/AndroidDevWithCpp)


### 通过 JNI 将 Bitmap 旋转

首先定义一个这样的 native 函数：

```java
    // 顺时针旋转 90° 的操作
    public native Bitmap rotateBitmap(Bitmap bitmap);
```

传入一个 Bitmap 对象，然后返回一个 Bitmap 对象。

然后在 C++ 代码中，首先检索 Bitmap 的信息，看看是否成功。

```cpp
    AndroidBitmapInfo bitmapInfo;
    int ret;
    if ((ret = AndroidBitmap_getInfo(env, bitmap, &bitmapInfo)) < 0) {
        LOGE("AndroidBitmap_getInfo() failed ! error=%d", ret);
        return NULL;
    }
```

接下来就是获得 Bitmap 的像素缓存指针：

```cpp
    // 读取 bitmap 的像素内容到 native 内存
    void *bitmapPixels;
    if ((ret = AndroidBitmap_lockPixels(env, bitmap, &bitmapPixels)) < 0) {
        LOGE("AndroidBitmap_lockPixels() failed ! error=%d", ret);
        return NULL;
    }
```

这个指针指向的就是 Bitmap 像素内容，它是一个以一维数组的形式保存所有的像素点的值，但是我们在定义 Bitmap 图像时，都会定义宽和高，这就相对于是一个二维的了，那么就存在 Bitmap 的像素内容如何转成指针指向的一维内容，是按照行排列还是按照列排列呢？

在这里是按照行进行排列的，而且行的排列是从左往右，列的排列是从上往下，起始点就和屏幕坐标原点一样，位于左上角。

通过 AndroidBitmap_lockPixels 方法，bitmapPixels 指针就指向了 Bitmap 的像素内容，它的长度就是 Bitmap 的宽和高的乘积。

要将 Bitmap 进行旋转，可以通过直接更改 bitmapPixels 指针指向的像素点的值，也可以通过创建一个新的 Bitmap 对象，然后将像素值填充到 Bitmap 对象中，这里选择后者的实现方式。


首先创建一个新的 Bitmap 对象，参考之前文章中提到的方式：[Android 通过 JNI 访问 Java 字段和方法调用](https://glumes.com/post/android/android-jni-access-field-and-method/)。

在 Java 代码中，通过  createBitmap 方法可以创建一个 Bitmap，如下所示：

```java
 Bitmap.createBitmap(int width, int height, @NonNull Config config)`
```
所以在 JNI 中就需要调用 Bitmap 的静态方法来创建一个 Bitmap 对象。

```cpp
jobject generateBitmap(JNIEnv *env, uint32_t width, uint32_t height) {

    jclass bitmapCls = env->FindClass("android/graphics/Bitmap");
    jmethodID createBitmapFunction = env->GetStaticMethodID(bitmapCls,
                                                            "createBitmap",
                                                            "(IILandroid/graphics/Bitmap$Config;)Landroid/graphics/Bitmap;");
    jstring configName = env->NewStringUTF("ARGB_8888");
    jclass bitmapConfigClass = env->FindClass("android/graphics/Bitmap$Config");
    jmethodID valueOfBitmapConfigFunction = env->GetStaticMethodID(
            bitmapConfigClass, "valueOf",
            "(Ljava/lang/String;)Landroid/graphics/Bitmap$Config;");

    jobject bitmapConfig = env->CallStaticObjectMethod(bitmapConfigClass,
                                                       valueOfBitmapConfigFunction, configName);

    jobject newBitmap = env->CallStaticObjectMethod(bitmapCls,
                                                    createBitmapFunction,
                                                    width,
                                                    height, bitmapConfig);
    return newBitmap;
}
```

首先通过 `FindClass` 方法找到 Config 类，得到一个 ARGB_8888 的配置，然后得到 Bitmap 类，调用它的静态方法 `createBitmap` 创建一个新的 Bitmap 对象，具体可以参考之前的文章。

在这里要传入新 Bitmap 的宽高，这个宽高也是通过 `AndroidBitmap_getInfo` 方法得到原来的宽高之后，根据不同的操作计算后得到的。

```cpp
	// 旋转操作，新 Bitmap 的宽等于原来的高，新 Bitmap 的高等于原来的宽
   uint32_t newWidth = bitmapInfo.height;
    uint32_t newHeight = bitmapInfo.width;
```

有了新的 Bitmap 对象，又有了原有的 Bitmap 像素指针，接下来就是创建新的像素指针，并填充像素内容，然后把这个像素内容再填充到 Bitmap 上。

```cpp
    // 创建一个新的数组指针，把这个新的数组指针填充像素值
	uint32_t *newBitmapPixels = new uint32_t[newWidth * newHeight];
    int whereToGet = 0;
    for (int y = 0; y < newHeight; ++y) {
        for (int x = newWidth - 1; x >= 0; x--) {
            uint32_t pixel = ((uint32_t *) bitmapPixels)[whereToGet++];
            newBitmapPixels[newWidth * y + x] = pixel;
        }
    }
```

在这两个 `for`循环里面就是从原来的像素指针中取出像素值，然后把它按照特定的排列顺序填充到新的像素指针中对应位置的值，这里也就是前面强调的像素指针是按照行进行排列的，起点是 Bitmap 的左上角。

```cpp
    void *resultBitmapPixels;
    if ((ret = AndroidBitmap_lockPixels(env, newBitmap, &resultBitmapPixels)) < 0) {
        LOGE("AndroidBitmap_lockPixels() failed ! error=%d", ret);
        return NULL;
    }
    int pixelsCount = newWidth * newHeight;
    memcpy((uint32_t *) resultBitmapPixels, newBitmapPixels, sizeof(uint32_t) * pixelsCount);
    AndroidBitmap_unlockPixels(env, newBitmap);
```

再次创建一个 resultBitmapPixels 指针，并调用 AndroidBitmap_lockPixels 方法获取新的 Bitmap 的像素指针缓存，然后调用 `memcpy` 方法，将待填充的像素指针填充到 resultBitmapPixels 上，这样就完成了像素的赋值，最后调用 `AndroidBitmap_unlockPixels` 方法释放像素指针缓存，完成整个赋值过程。


就这样通过读取原有 Bitmap 的像素内容然后进行操作后再赋值给新的 Bitmap 对象就完成了 JNI 操作 Bitmap 。


### 通过 JNI 将 Bitmap 上下翻转和左右镜像


将 Bitmap 进行上下翻转以及左右镜像和旋转操作类似了，只是针对像素指针的操作方式不同。

上下翻转的操作：

```cpp
    int whereToGet = 0;
    for (int y = 0; y < newHeight; ++y) {
        for (int x = 0; x < newWidth; x++) {
            uint32_t pixel = ((uint32_t *) bitmapPixels)[whereToGet++];
            newBitmapPixels[newWidth * (newHeight - 1 - y) + x] = pixel;
        }
    }
```

左右镜像的操作：

```cpp
    int whereToGet = 0;
    for (int y = 0; y < newHeight; ++y) {
        for (int x = newWidth - 1; x >= 0; x--) {
            uint32_t pixel = ((uint32_t *) bitmapPixels)[whereToGet++];
            newBitmapPixels[newWidth * y + x] = pixel;
        }
    }
```

其他的操作都相同了，具体还是看项目代码吧。

参考

1. 《Android C++ 高级编程--使用 NDK》

