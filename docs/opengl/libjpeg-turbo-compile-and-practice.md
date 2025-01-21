---
title: "图像库 libjpeg-turbo 编译与实践"
date: 2019-07-13T00:08:18+08:00
subtitle: ""
slug: "libjpeg-turbo-compile-and-practice"
tags: ["libjpeg-turbo","image"]
categories: ["OpenGL"]
 
toc: true
draft: false
original: true
author: "glumes"
---

<!--more-->

> 避免图片丢失，建议阅读微信原文：
> 
> https://mp.weixin.qq.com/s/x-dTLowXcqysnLxZ0BGA2g

在之前的文章中已经陆续介绍了 stb_image、libpng 的使用，相关链接如下：


【[简单易用的图像解码库介绍 —— stb_image](https://glumes.com/post/android/stb-image-introduce/)】

[https://glumes.com/post/android/stb-image-introduce/](https://glumes.com/post/android/stb-image-introduce/)


【[图像库 libpng 编译与实践](https://glumes.com/post/opengl/libpng-compile-and-practice/)】

[https://glumes.com/post/opengl/libpng-compile-and-practice/](https://glumes.com/post/opengl/libpng-compile-and-practice/)


而今天的主题就是 libjpeg-turbo 。

它的官网地址如下：

> [https://libjpeg-turbo.org/](https://libjpeg-turbo.org/)

它的 github 地址如下：

> [https://github.com/libjpeg-turbo/libjpeg-turbo](https://github.com/libjpeg-turbo/libjpeg-turbo)




## 编译

在 libjpeg-turbo 的源码中就已经有了讲述如何编译的 BUILDING.md 文件，还是使用 CMake 进行编译，大体方法和参数设置都大同小异了。

参考源码给出的编译代码：

```cpp
# Set these variables to suit your needs
# 设置交叉编译的变量
NDK_PATH={full path to the NDK directory-- for example,
  /opt/android/android-ndk-r16b}
TOOLCHAIN={"gcc" or "clang"-- "gcc" must be used with NDK r16b and earlier,
  and "clang" must be used with NDK r17c and later}
ANDROID_VERSION={the minimum version of Android to support-- for example,
  "16", "19", etc.}

cd {build_directory}
cmake -G"Unix Makefiles" \
  -DANDROID_ABI=armeabi-v7a \
  -DANDROID_ARM_MODE=arm \
  -DANDROID_PLATFORM=android-${ANDROID_VERSION} \
  -DANDROID_TOOLCHAIN=${TOOLCHAIN} \
  -DCMAKE_ASM_FLAGS="--target=arm-linux-androideabi${ANDROID_VERSION}" \
  -DCMAKE_TOOLCHAIN_FILE=${NDK_PATH}/build/cmake/android.toolchain.cmake \
  [additional CMake flags] {source_directory}
make
```

由于 CMake 跨平台编译的特性，在进行交叉编译时要设置很多相关参数，比如编译的目标系统平台、交叉编译工具链、NDK 目录等。

详细的编译脚本可以参考项目中的：

> [https://github.com/glumes/InstantGLSL/tree/master/libjpeg_turbo_source/build_script](https://github.com/glumes/InstantGLSL/tree/master/libjpeg_turbo_source/build_script)

该目录下给出了编译 armeabi、armeabi-v7a、arm64-v8a、x86、x86_64 等平台的编译脚本。修改一下相应文件路径，就能在 MAC 系统上编译 so 了。

另外如果是在 Android Studio 中用 CMake 编译 so，你会发现很少要设置那些参数，这是因为 Android Studio 中的 CMake 默认就设置好了那些参数。

因此还有一种更简单的方式进行编译，直接将 libjpeg-turbo 源码内容复制到 Android Studio 工程目录的 cpp 文件夹下，然后把 app 的 build.gradle 中 cmake path 改成 libjpeg-turbo 的 CMakeLists.txt 路径，如下所示：

```cpp
    externalNativeBuild {
        cmake {
            path "src/main/cpp/libjpeg_turbo_source/CMakeLists.txt"
//            path "CMakeLists.txt"
        }
    }
```

然后直接编译，在 build/intermediates/cmake 路径下依旧可以找到编译好的 so 文件。

以上两种方式都可以实现 libjpeg-turbo 的编译，看个人喜好了。而且这种库一旦编译好了，以后也很少去更改，一劳永逸~~~

## 实践

在 libjpeg-turbo 的源码中有个 example.txt 文件，详细讲述了如何利用该库进行图片压缩和解压缩。

基本上照着文件内容看一遍就懂了，在这里还会大概讲述下，并且会用另一个实例来演示，也就是之前常用的，获取 jpeg 图像文件像素内容并且上传纹理。

### 压缩

在 Android 中通过 Java 方法也可以实现 Jpeg 的文件，因为底层就是基于 libjpeg 的。而 libjpeg-turbo 的压缩速度会比 Android 原生的速度更快了。

Android 中 Jpeg 文件压缩的方法如下：

```java
compress(CompressFormat format, int quality, OutputStream stream)
```

其中重要的参数就是 quality ，代表要压缩的质量，而在 libjpeg-turbo 也会有这样的参数要设置。

libjpeg-turbo 的使用逻辑和 libpng 有点类似，首先都是要设置一个错误返回点，并且有一个结构体来存储信息。

在 libjpeg-turbo 进行压缩时，用到的结构体是 `jpeg_compress_struct` ，解压则是 `jpeg_decompress_struct`，两者名字上有着单词的不同。 

而在 libpng 中，创建结构体的方法 `png_create_write_struct` 和 `png_create_read_struct` 相配对，一个 write，一个 read 。

使用 libjpeg-turbo 的主要步骤如下：

1. 设置压缩后的输出方式，可以的是文件的形式，也可以是内存数据格式
2. 配置压缩的相关设置项，比如压缩后的图像宽高、压缩质量等
3. 进行压缩，逐行读取数据源像素内容
4. 压缩结束，得到压缩后的数据

对应到代码的逻辑如下：

```cpp
    struct jpeg_compress_struct jpegCompressStruct;
    // 创建代表压缩的结构体
    jpeg_create_compress(&jpegCompressStruct);
    
    // 文件方式输出 还有一种是内存方式
    jpeg_stdio_dest(&jpegCompressStruct, fp);
    // 设置压缩的相关参数信息
    jpegCompressStruct.image_width = w;
    jpegCompressStruct.image_height = h;
    jpegCompressStruct.arith_code = false;
    jpegCompressStruct.input_components = nComponent;
    // 设置解压的颜色
    jpegCompressStruct.in_color_space = JCS_RGB;
    jpeg_set_defaults(&jpegCompressStruct);
    // 压缩的质量
    jpegCompressStruct.optimize_coding = quality;
    jpeg_set_quality(&jpegCompressStruct, quality, true);
```

首先是创建结构体 jpeg_compress_struct ，通过该结构体来完成压缩数据输出、配置压缩选项操作。

压缩数据输出有两种方式：

```cpp
    // 以文件的方式
    jpeg_stdio_dest(j_compress_ptr cinfo, FILE *outfile);
    // 以内存的方式
    jpeg_mem_dest(j_compress_ptr cinfo, unsigned char **outbuffer,
                           unsigned long *outsize)
```


另外，压缩选项除了常见的宽高信息、颜色类型，还有最重要的图像质量参数，通过专门的方法进行设置。

```cpp
    jpeg_set_quality(j_compress_ptr cinfo, int quality,
                                boolean force_baseline)
```

设置完要压缩的相关信息后，就可以开始压缩了。

```cpp
    // 开始压缩
    jpeg_start_compress(&jpegCompressStruct, true);
    // JSAMPROW 代表每行的数据
    JSAMPROW row_point[1];
    int row_stride = jpegCompressStruct.image_width * nComponent;

    while (jpegCompressStruct.next_scanline < jpegCompressStruct.image_height) {
        // data 参数就是要压缩的数据源
        // 逐行读取像素内容
        row_point[0] = &data[jpegCompressStruct.next_scanline * row_stride];
        // 写入数据
        jpeg_write_scanlines(&jpegCompressStruct, row_point, 1);
    }
    
   // 完成压缩 
   jpeg_finish_compress(&jpegCompressStruct);
   // 释放相应的结构体
   jpeg_destroy_compress(&jpegCompressStruct);

```

主要的代码就在于 while 循环中。

`next_scanline` 类似于一个状态变量，需要逐行去扫描图像内容并写入，每次 jpeg_write_scanlines 方法之后，`next_scanline` 就会递增，直到退出循环。


### 解压缩

解压和压缩的代码结构大致相同了。

```cpp
    jpeg_create_decompress(&cinfo);
    // 设置数据源数据方式 这里以文件的方式，也可以以内存数据的方式
    jpeg_stdio_src(&cinfo, fp);
    // 读取文件信息，比如宽高之类的
    jpeg_read_header(&cinfo, TRUE);
```

其中 jpeg_read_header 方法可以获取要解压的文件相关信息。

接下来设置解压的相关参数：

```cpp
    jpeg_start_decompress(&cinfo);
    unsigned long width = cinfo.output_width;
    unsigned long height = cinfo.output_height;
    unsigned short depth = cinfo.output_components;
    row_stride = cinfo.output_width * cinfo.output_components;
```

定义相关的数据变量，来保存解压的数据。





```cpp
    // 保存每行解压的数据内容    
    JSAMPARRAY buffer;
    // 初始化
    buffer = (*cinfo.mem->alloc_sarray)((j_common_ptr) &cinfo, JPOOL_IMAGE, row_stride, 1);
    
    // 保存图像的所有数据
    unsigned char *src_buff;
    // 初始化并置空
    src_buff = static_cast<unsigned char *>(malloc(width * height * depth));
    memset(src_buff, 0, sizeof(unsigned char) * width * height * depth);
```

这里使用的是 `JSAMPARRAY` 来保存，在 libjpeg-turbo 中其实有多种结构来表示图像数据类型，压缩中用到的就是 `JSAMPROW` 。

```cpp
/* Data structures for images (arrays of samples and of DCT coefficients).
 */
typedef unsigned char JSAMPLE;
typedef JSAMPLE *JSAMPROW;      /* ptr to one image row of pixel samples. */
typedef JSAMPROW *JSAMPARRAY;   /* ptr to some rows (a 2-D sample array) */
typedef JSAMPARRAY *JSAMPIMAGE; /* a 3-D sample array: top index is color */
```

通过上述代码不难看出，`JSAMPROW` 其实就是一维数组，而 `JSAMPARRAY` 就是二维数组。以下两行代码可以说是等价的：

```cpp
    JSAMPARRAY buffer = nullptr;
    JSAMPROW *row_pointer = nullptr;
```

因为 `JSAMPARRAY` 的定义就是 `JSAMPROW *` 。

具体用哪个更好，要看调用方法需要的参数类型了，`jpeg_write_scanlines` 和 `jpeg_read_scanlines` 这两个方法需要的都是 `JSAMPARRAY` 类型。

另外，代码中还声明并初始化了 `src_buff` 变量，该变量就是用来表示解压后的图像数据。

具体解压的逻辑也比较清楚了，逐行扫描图像，用 `buffer` 变量去存储图像每行解压的数据，然后把这个数据给到 `src_buff` 变量，如下代码所示：

```cpp
    unsigned char *point = src_buff;

    while (cinfo.output_scanline < height) {
        jpeg_read_scanlines(&cinfo, buffer, 1);
        memcpy(point, *buffer, width * depth);
        point += width * depth;
    }

    jpeg_finish_decompress(&cinfo);
    jpeg_destroy_decompress(&cinfo);
```

这里用到了 `point` 这样的临时变量，实际上，这样的用法在 C++ 开发中应该算比较常见的。

因为把 `buffer` 的数据传到 `src_buff` 后，`src_buff` 指针要移动到下一个点去接收数据，这样一来，指针指向的位置就不是原始位置了，所以才需要临时变量去做移动操作，保证 `src_buff` 指向的位置为起始点。

最后，别忘了释放相应的变量，做一些收尾工作，解压就完成了。

### jpeg 上传纹理渲染


说完了压缩和解压缩，最后以一个例子来实际应用，也是之前文章中常用的例子，通过 libjpeg-turbo 读取 jpeg 文件图像内容并上传纹理渲染。

```cpp
    // 封装 jpeg 相关操作
    JpegHelper jpegHelper;
    // 读取的图像内容
    unsigned char *jpegData;
    int jpegSize;
    int jpegWidth;
    int jpegHeight;
    // 读取操作
    jpegHelper.read_jpeg_file(filePath, &jpegData, &jpegSize, &jpegWidth, &jpegHeight);
```

最终要获取的就是 `jpegData` 图像内容，并通过 `glTexImage2D` 等方法渲染到纹理上。

`read_jpeg_file` 方法的声明如下：

```cpp
    int read_jpeg_file(const char *jpeg_file, unsigned char **rgb_buffer, 
                       int *size, int *width,int *height)
```

`rgb_buffer` 参数的数据类型，既可以声明成 `unsigned char **` 这样指针的地址，也可以声明成引用 `unsigned char *&` ，大同小异。

至于具体的读取操作，和上面的解压缩过程大致相同，就不在阐述一遍了，可以查看我的项目代码实践：

> [https://github.com/glumes/InstantGLSL](https://github.com/glumes/InstantGLSL)

## 总结


至此，总结了常用的三种图像库的编译和使用。

这三种图像库各有特点，要根据实际需要，选择最合适的。但实际我们用到的无非就是图像的读写操作。读取特定格式图像的像素内容，或者将像素内容写入特定格式文件。

平时写 demo ，追求简单方便的，就用 `stb_image`，对性能有要求的，针对特定格式的选择 `libpng` 或者 `libjpeg-turbo`。




