---
title: "图像库 libpng 编译与实践"
date: 2019-07-12T23:26:58+08:00
subtitle: ""
slug: "libpng-compile-and-practice"
tags: ["libpng","image"]
categories: ["OpenGL"]
 
toc: true
draft: false
original: true
---


在之前的文章中介绍了 `stb_image` 图像库，还顺带提到了 libpng 和 libjpeg ，这篇文章就是介绍如何在 Android 平台上用 CMake 编译 libpng 动态库以及 libpng 使用实践。

【[简单易用的图像解码库介绍 —— stb_image](https://glumes.com/post/android/stb-image-introduce/)】

[https://glumes.com/post/android/stb-image-introduce/](https://glumes.com/post/android/stb-image-introduce/)

<!--more-->

## libpng 介绍

libpng 的官方介绍网站如下：

> [http://www.libpng.org/pub/png/libpng.html](http://www.libpng.org/pub/png/libpng.html)

下载地址网站如下：

> [https://sourceforge.net/projects/libpng/files/](https://sourceforge.net/projects/libpng/files/)

博客中使用的版本是 1.6.37 ，也是目前最新的版本了。

关于 libpng 的编译网上已经有不少博客教程了，但有的是基于 Linux，有的是基于 Android.mk 的，本文会介绍如何在 Android Studio 上通过 CMake 来编译 Android 的动态库。


## CMake 编译 libpng 动态库

### neon 相关编译

在 libpng 的源代码中，就提供了 CMakeLists.txt 文件用以说明如何编译，但是却不能直接用在 Android 平台上，不过可以借鉴其源码作为参考。

由于 CMake 跨平台编译的特性，一般大型项目代码编译都会针对平台做适配，常见代码结构如下：

```cmake
    if (CMAKE_SYSTEM_PROCESSOR MATCHES "^arm" OR
            CMAKE_SYSTEM_PROCESSOR MATCHES "^aarch64")
        set(libpng_arm_sources
                arm/arm_init.c
                arm/filter_neon.S
                arm/filter_neon_intrinsics.c
                arm/palette_neon_intrinsics.c)
        // 定义宏
        add_definitions(-DPNG_ARM_NEON_OPT=2)
    endif ()
```

这段代码就是判断系统处理器平台，不同平台所需要编译的代码不一样。而 libpng 会有这样的适配，主要是因为它用到了 neon 相关优化，该优化主要是用在 `filter` 操作方面。

```cmake
// libpng 使用 neon 优化加速的方法
void
png_read_filter_row_up_neon(png_row_infop row_info, png_bytep row,
   png_const_bytep prev_row)
{
   png_bytep rp = row;
   png_bytep rp_stop = row + row_info->rowbytes;
   png_const_bytep pp = prev_row;

   png_debug(1, "in png_read_filter_row_up_neon");

   for (; rp < rp_stop; rp += 16, pp += 16)
   {
      uint8x16_t qrp, qpp;

      qrp = vld1q_u8(rp);
      qpp = vld1q_u8(pp);
      qrp = vaddq_u8(qrp, qpp);
      vst1q_u8(rp, qrp);
   }
}
```

通过查看 libpng 源代码，要启用 neon 优化，还必须通过 add_definitions 方法定义 DPNG_ARM_NEON_OPT 宏的值为 2 ，否则在源码中会认为不需要使用 neon 。

要使用 neon 编译，还需要指定编译器相关参数：

```cmake
set_property(SOURCE ${libpng_arm_sources}
        APPEND_STRING PROPERTY COMPILE_FLAGS " -mfpu=neon")
```

不过看到网上一些 libpng 编译文章，基本没提到 neon 相关东西，估计这个优化加速功能用不上吧。

但是，可以在我的 Demo 上看到如何启用 neon 去编译，以后也会写专门的文章来介绍 neon 的使用~~

### zlip 库依赖

libpng 动态库编译还依赖 zlip 库，要是在其他平台上需要单独下载这个库，但是 Android 上就不需要了，因为 Android 编译环境本身就提供了这个库，就像我们使用 log 库一样。

```cmake
// 指定要编译的 so 依赖哪些其他的 so , z 就是 zlib 库
target_link_libraries(png z log )
```

Android 编译环境中 `z` 就是 zlip 库了。

### 源码编译

其他的就是源码编译了，主要是 add_library 方法的使用，要指定好需要编译的源文件。

具体有哪些源文件需要添加到编译中，还是请参考如下链接，就不贴具体代码了，减少文章篇幅。

[https://github.com/glumes/InstantGLSL/blob/master/instantglsl/src/main/cpp/libpng/CMakeLists.txt](https://github.com/glumes/InstantGLSL/blob/master/instantglsl/src/main/cpp/libpng/CMakeLists.txt)

完成上述三个过程后，就能够编译出 libpng 的动态库了，实际编译过程还是参考项目代码吧。

## libpng 的使用实践

编译是小事，重点在使用~~~

以解码 png 图片获取像素内容为例：

### linpng 初始化

首先是初始化 libpng ，得到 `png_structp` 结构体。它可以说是代表了 libpng 上下文，在方法调用时都需要把它作为第一个参数传入。

```cpp
    // 传 nullptr 的参数是用来自定义错误处理的，这里不需要
    png_structp png = png_create_read_struct(PNG_LIBPNG_VER_STRING, nullptr, nullptr, nullptr);
```

由于是读取，方法名中带有 read ,如果是写入，那就是 png_create_write_struct 方法了。

### 设置错误返回点

由于在创建 png 变量时，用来自定义错误处理的参数都传了 nullptr，所以需要设置错误返回点，这样当 libpng 发生错误时，程序将回到这个调用点，这时候可以做一些清理工作：

```cpp
    if (setjmp(png_jmpbuf(png))) {
        png_destroy_read_struct(&png, nullptr, nullptr);
        fclose(fp);
        return;
    }
```

### 判断文件是否是 png 格式

libpng 提供了 png_sig_cmp 方法来检查文件是否 png 格式。

```cpp
    #define PNG_BYTES_TO_CHECK 4
    char buf[PNG_BYTES_TO_CHECK];
    // 读取 buffer
    if (fread(buf, 1, PNG_BYTES_TO_CHECK, fp) != PNG_BYTES_TO_CHECK) {
        return;
    }
    // 判断
    if (!png_sig_cmp(reinterpret_cast<png_const_bytep>(buf), 0, PNG_BYTES_TO_CHECK)) {
        // 返回值不等于 0 则是 png 文件格式
    } 
```

如果调用了该方法，需要通过 png_set_sig_bytes 方法告诉 libpng 该跳过相应的数据，否则会出现黑屏，或者通过 rewind 方法重置文件指针。

### 获取图像信息

首先创建 png_infop 结构体来代表图像信息：

```cpp
    png_infop infop = png_create_info_struct(png);
```

然后是设置图像的数据源，前提是要得到文件路径：

```cpp
    // 根据文件路径打开文件
    FILE *fp = fopen(mFileName.c_str(), "rb");
    // 设置图像数据源
    png_init_io(png, fp);
```

接下来是读取信息：

```cpp
    png_read_png(png, infop,
                 (PNG_TRANSFORM_STRIP_16 | PNG_TRANSFORM_PACKING | PNG_TRANSFORM_EXPAND),
```

通过如下方法，能得到图像具体某方面信息：

```cpp
    mWidth = png_get_image_width(png, info);
    mHeight = png_get_image_height(png, info);
    mColorType = png_get_color_type(png, info);
    mBitDepth = png_get_bit_depth(png, info);
```

当然也可以通过 png_get_IHDR 方法去获得信息

```cpp
    png_get_IHDR(png, infop, &mWidth, &mHeight, &mBitDepth, &mColorType, &mInterlaceType,
                 &mCompressionType, &mFilterType);
```

### 获取像素内容

通过 png_get_rows 方法按行获取所有的数据，然后赋值到像素指针上去。

```cpp
	// 代表像素内容的指针
    unsigned char *mPixelData;
    // 获取每行的字节数量
    unsigned int row_bytes = png_get_rowbytes(png, infop);
    mPixelData = new unsigned char[row_bytes * mHeight];
    png_bytepp rows = png_get_rows(png, infop);
    // 逐行读取，并填充到像素指针上去
    for (int i = 0; i < mHeight; ++i) {
        memcpy(mPixelData + (row_bytes * i), rows[i], row_bytes);
    }
```

其实在前面的 png_read_png 方法中就已经得到了所有的像素内容，保存在 infop 变量的 row_pointers 中，具体的实现如下：

通过 png_read_image 方式读取像素内容：

```cpp
    // row_pointers 当成了一维指针数组
    png_bytep *row_pointers;
    row_pointers = (png_bytep *) malloc(sizeof(png_bytep) * mHeight);
    for (int y = 0; y < mHeight; y++) {
        row_pointers[y] = (png_byte *) malloc(png_get_rowbytes(png, info));
    }
    png_read_image(png, row_pointers);
```

最后，别忘了调用 png_read_end 方法结束读取。

有了像素内容，就可以做一个常见的渲染操作了，将像素内容渲染绘制到纹理上。

### 保存图片

最后介绍如何根据像素内容去保存图片，在 libpng 中也提供了相应的方法调用，流程就是如下方法：

```cpp
    png = png_create_write_struct()
    infop = png_create_info_struct()
    // 关联数据源，png 和要写入的文件
    png_init_io(png,fp)
    // 设置 infop 相关参数，代表最好要生成的图片文件相关信息
    png_set_IHDR()
    // 写入图片信息
    png_write_info(png, infop);
    // 写入图片像素内容
    png_write_image(png, row_pointers);
    // 结束写入
    png_write_end(png, NULL);
```

流程和读取像素内容恰好相反。

其中 png 变量要通过 png_create_write_struct 创建。

infop 变量还是 png_create_info_struct 方法创建。

接下来就是设置图片信息，写入图片信息，写入像素内容，具体的代码实践可以参考我的代码示例。

## 参考

最后，在 libpng 的源代码中，也提供了丰富的示例，一般这种开源库都会提供相应的 test 代码，通过 test 代码基本都能找到相应的函数调用。

libpng 的官网示例地址如下：

[http://www.libpng.org/pub/png/libpng-manual.txt](http://www.libpng.org/pub/png/libpng-manual.txt)

有疑问的话，基本都可以在这个上面找到答案。

