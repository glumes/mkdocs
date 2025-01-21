---
title: "Android NDK 开发之 CMake 必知必会"
date: 2018-09-12T09:19:09+08:00
subtitle: ""
tags: ["NDK"]
categories: ["Android"]
toc: true
draft: false
original: true
addwechat: true
 
slug: "cmake-best-practices"
---

Android Studio 从 2.2 版本起开始支持 CMake ,可以通过 CMake 和 NDK 将 C/C++ 代码编译成底层的库，然后再配合 Gradle 的编译将库打包到 APK 中。

这意味就不需要再编写 `.mk` 文件来编译 `so` 动态库了。

<!--more-->

CMake 是一个跨平台构建系统，在 Android Studio 引入 CMake 之前，它就已经被广泛运用了。

Google 官方网站上有对 CMake 的使用示范，可以参考 [官方指南](https://developer.android.com/studio/projects/configure-cmake)。

总结官网对 CMake 的使用，其实也就如下的步骤：

1. add_library 指定要编译的库，并将所有的 `.c` 或 `.cpp` 文件包含指定。
2. include_directories 将头文件添加到搜索路径中
3. set_target_properties 设置库的一些属性
4. target_link_libraries 将库与其他库相关联

如果你对上面的步骤还是不了解，那么接下来就更深入了解 CMake 相关内容吧~~~

## CMake 的基本操作

以 [Clion](https://www.jetbrains.com/clion/) 作为工具来讲解 CMake 的基本使用。

![clion_cmake_build](https://image.glumes.com/images/2019/04/27/clion_cmake_build.png)

### CMake 编译可执行文件

一个打印 hello world 的 cpp 文件，通过 CMake 将它编译成可执行文件。

在 cpp 的同一目录下创建 `CMakeLists.txt` 文件，内容如下：

```CMake
# 指定 CMake 使用版本
cmake_minimum_required(VERSION 3.9)
# 工程名
project(HelloCMake)
# 编译可执行文件
add_executable(HelloCMake main.cpp )
```

其中，通过 `cmake_minimum_required` 方法指定 CMake 使用版本，通过 `project` 指定工程名。

而 `add_executable` 就是指定最后编译的可执行文件名称和需要编译的 cpp 文件，如果工程很大，有多个 cpp 文件，那么都要把它们添加进来。

定义了 CMake 文件之后，就可以开始编译构建了。

CMake 在构建工程时会生成许多临时文件，避免让这些临时文件污染代码，一般会把它们放到一个单独的目录中。

操作步骤如下：

```bash
# 在 cpp 目录下创建 build 目录
mkdir build
# 调用 cmake 命令生成 makefile 文件
cmake ..
# 编译
make
```

在 build 目录中可以找到最终生成的可执行文件。

这就是 CMake 的一个简单操作，将 cpp 编译成可执行文件，但在 Android 中，大多数场景都是把 cpp 编译成库文件。

### CMake 编译静态库和动态库

同样还是一个 cpp 文件和一个 CMake 文件，cpp 文件内容为打印字符串的函数：

```cpp
#include <iostream>
void print() {
    std::cout << "hello lib" << std::endl;
}
```

同时，CMake 文件也要做相应更改：

```cmake
cmake_minimum_required(VERSION 3.12)
# 指定编译的库和文件，SHARED 编译动态库
add_library(share_lib SHARED lib.cpp)
# STATIC 编译静态库
# add_library(share_lib STATIC lib.cpp)
```

通过 `add_library` 指定要编译的库的名称，以及动态库还是静态库，还有要编译的文件。

最后同样地执行构建，在 build 目录下可以看到生成的库文件。

到这里，就基本可以使用 CMake 来构建 C/C++ 工程了。

## CMake 基本语法

熟悉了上面的基本操作之后，就必然会遇到以下的问题了：

*   如果要参与编译的 C/C++ 文件很多，难道每个都要手动添加嘛？
*   可以把编译好的可执行文件或者库自动放到指定位置嘛?
*   可以把编译好的库指定版本号嘛？


带着这些问题，还是要继续深入学习 CMake 的相关语法，最好的学习材料就是 [官网文档](https://cmake.org/cmake/help/v3.12/index.html) 了。

为了避免直接看官方文档时一头雾水，这里列举一些常用的语法命令。

### 注释与大小写

在前面就已经用到了 CMake 注释了，每一行的开头 `#` 代表注释。

另外，CMake 的所有语法指令是不区分大小写的。

### 变量定义与消息打印

通过 `set` 来定义变量：

```cmake
# 变量名为 var，值为 hello
set(var hello) 
```

当需要引用变量时，在变量名外面加上 `${}` 符合来引用变量。
```cmake
# 引用 var 变量
${var}
```

还可以通过 `message` 在命令行中输出打印内容。

```cmake
set(var hello) 
message(${var})
```

### 数学和字符串操作

#### 数学操作

CMake 中通过 `math` 来实现数学操作。

```cmake
# math 使用，EXPR 为大小
math(EXPR <output-variable> <math-expression>)
```

```cmake
math(EXPR var "1+1")
# 输出结果为 2
message(${var})
```

math 支持 `+, -, *, /, %, |, &, ^, ~, <<, >> ` 等操作，和 C 语言中大致相同。


#### 字符串操作

CMake 通过 `string` 来实现字符串的操作，这波操作有很多，包括将字符串全部大写、全部小写、求字符串长度、查找与替换等操作。

具体查看 [官方文档](https://cmake.org/cmake/help/v3.12/command/string.html)。

```cmake
set(var "this is  string")
set(sub "this")
set(sub1 "that")
# 字符串的查找，结果保存在 result 变量中
string(FIND ${var} ${sub1} result )
# 找到了输出 0 ，否则为 -1
message(${result})

# 将字符串全部大写
string(TOUPPER ${var} result)
message(${result})

# 求字符串的长度
string(LENGTH ${var} num)
message(${num})
```

另外，通过空白或者分隔符号可以表示字符串序列。

```cmake
set(foo this is a list) // 实际内容为字符串序列
message(${foo})
```

当字符串中需要用到空白或者分隔符时，再用双括号`""`表示为同一个字符串内容。

```cmake
set(foo "this is a list") // 实际内容为一个字符串
message(${foo})
```

### 文件操作

CMake 中通过 `file` 来实现文件操作，包括文件读写、下载文件、文件重命名等。

具体查看 [官方文档](https://cmake.org/cmake/help/v3.12/command/file.html)

```cmake
# 文件重命名
file(RENAME "test.txt" "new.txt")

# 文件下载
# 把文件 URL 设定为变量
set(var "http://img.zcool.cn/community/0117e2571b8b246ac72538120dd8a4.jpg")

# 使用 DOWNLOAD 下载
file(DOWNLOAD ${var} "/Users/glumes/CLionProjects/HelloCMake/image.jpg")
```

在文件的操作中，还有两个很重要的指令 `GLOB` 和 `GLOB_RECURSE` 。

```cmake
# GLOB 的使用
file(GLOB ROOT_SOURCE *.cpp)
# GLOB_RECURSE 的使用
file(GLOB_RECURSE CORE_SOURCE ./detail/*.cpp)
```

其中，`GLOB` 指令会将所有匹配 `*.cpp` 表达式的文件组成一个列表，并保存在 `ROOT_SOURCE` 变量中。

而 `GLOB_RECURSE` 指令和 `GLOB` 类似，但是它会遍历匹配目录的所有文件以及子目录下面的文件。

使用  `GLOB` 和 `GLOB_RECURSE` 有好处，就是当添加需要编译的文件时，不用再一个一个手动添加了，同一目录下的内容都被包含在对应变量中了，但也有弊端，就是新建了文件，但是 CMake 并没有改变，导致在编译时也会重新产生构建文件，要解决这个问题，就是动一动 CMake，让编译器检测到它有改变就好了。



### 预定义的常量

在 CMake 中有许多预定义的常量，使用好这些常量能起到事半功倍的效果。

*   CMAKE_CURRENT_SOURCE_DIR
    *   指当前 CMake 文件所在的文件夹路径
*   CMAKE_SOURCE_DIR
    *   指当前工程的 CMake 文件所在路径
*   CMAKE_CURRENT_LIST_FILE
    *   指当前 CMake 文件的完整路径
*   PROJECT_SOURCE_DIR
    *   指当前工程的路径

比如，在 `add_library` 中需要指定 cpp 文件的路径，以 `CMAKE_CURRENT_SOURCE_DIR` 为基准，指定 cpp 相对它的路径就好了。


```cmake
# 利用预定义的常量来指定文件路径
add_library( # Sets the name of the library.
             openglutil
             # Sets the library as a shared library.
             SHARED
             # Provides a relative path to your source file(s).
             ${CMAKE_CURRENT_SOURCE_DIR}/opengl_util.cpp
             )
```

### 平台相关的常量

CMake 能够用来在 Window、Linux、Mac 平台下进行编译，在它的内部也定义了和这些平台相关的变量。

具体查看 [官方文档](https://cmake.org/cmake/help/v3.12/manual/cmake-variables.7.html) 。

列举一些常见的：

*   WIN32
    *   如果编译的目标系统是 Window,那么 WIN32 为 True 。
*   UNIX
    *   如果编译的目标系统是 Unix 或者类 Unix 也就是 Linux ,那么 UNIX 为 True 。
*   MSVC
    *   如果编译器是 Window 上的 Visual C++ 之类的，那么 MSVC 为 True 。
*   ANDROID
    *   如果目标系统是 Android ，那么 ANDROID 为 1 。
*   APPLE
    *   如果目标系统是 APPLE ，那么 APPLE 为 1 。

有了这些常量做区分，就可以在一份 CMake 文件中编写不同平台的编译选项。

```cmake
if(WIN32){
    # do something
}elseif(UNIX){
    # do something
}
```


### 函数、宏、流程控制和选项 等命令

具体参考[cmake-commands](https://cmake.org/cmake/help/v3.12/manual/cmake-commands.7.html) ，这里面包括了很多重要且常见的指令。

简单示例 CMake 中的函数操作：

```cmake
function(add a b)
    message("this is function call")
    math(EXPR num "${a} + ${b}" )
    message("result is ${aa}")
endfunction()

add(1 2)
```

其中，function 为定义函数，第一个参数为函数名称，后面为函数参数。

在调用函数时，参数之间用空格隔开，不要用逗号。


宏的使用与函数使用有点类似：

```cmake
macro(del a b)
    message("this is macro call")
    math(EXPR num "${a} - ${b}")
    message("num is ${num}")
endmacro()

del(1 2)
```

在流程控制方面，CMake 也提供了 if、else 这样的操作：

```cmake
set(num 0)
if (1 AND ${num})
    message("and operation")
elseif (1 OR ${num})
    message("or operation")
else ()
    message("not reach")
endif ()
```

其中，CMake 提供了 `AND`、`OR`、`NOT`、`LESS`、`EQUAL` 等等这样的操作来对数据进行判断，比如 `AND` 就是要求两边同为 True 才行。

另外 CMake 还提供了循环迭代的操作：

```cmake
set(stringList this is string list)
foreach (str ${stringList})
    message("str is ${str}")
endforeach ()
```

CMake 还提供了一个 `option` 指令。

可以通过它来给 CMake 定义一些全局选项:

```cmake
option(ENABLE_SHARED "Build shared libraries" TRUE)

if(ENABLE_SHARED)
    # do something
else()
    # do something   
endif()
```

可能会觉得 `option` 无非就是一个 True or False 的标志位，可以用变量来代替，但使用变量的话，还得添加 `${}` 来表示变量，而使用 `option` 直接引用名称就好了。


## CMake 阅读实践

明白了上述的 CMake 语法以及从官网去查找陌生的指令意思，就基本上可以看懂大部分的 CMake 文件了。

这里举两个开源库的例子：

*  [https://github.com/g-truc/glm](https://github.com/g-truc/glm)
    *  `glm` 是一个用来实现矩阵计算的，在 OpenGL 的开发中会用到。
    *  `CMakeLists.txt` 地址在 [这里](https://github.com/g-truc/glm/blob/master/CMakeLists.txt)
*  [https://github.com/libjpeg-turbo/libjpeg-turbo](https://github.com/libjpeg-turbo/libjpeg-turbo)
    *  `libjpeg-turbo` 是用来进行图片压缩的，在 Android 底层就是用的它。
    *  `CMakeLists.txt` 地址在 [这里](https://github.com/libjpeg-turbo/libjpeg-turbo/blob/master/CMakeLists.txt)

这两个例子中大量用到了前面所讲的内容，可以试着读一读增加熟练度。

## 为编译的库设置属性

接下来再回到用 CMake 编译动态库的话题上，毕竟 Android NDK 开发也主要是用来编译库了，当编译完 so 之后，我们可以对它做一些操作。

通过 `set_target_properties` 来给编译的库设定相关属性内容，函数原型如下：


```cmake
set_target_properties(target1 target2 ...
                      PROPERTIES prop1 value1
                      prop2 value2 ...)
```

比如，要将编译的库改个名称：

```cmake
set_target_properties(native-lib PROPERTIES OUTPUT_NAME "testlib" )
```


更多的属性内容可以参考 [官方文档](https://cmake.org/cmake/help/v3.9/manual/cmake-properties.7.html#target-properties)。

不过，这里面有一些属性设定无效，在 Android Studio 上试了无效，在 CLion 上反而可以，当然也可能是我使用姿势不对。

比如，实现动态库的版本号：

```cmake
set_target_properties(native-lib PROPERTIES VERSION 1.2 SOVERSION 1 )
```

对于已经编译好的动态库，想要把它导入进来，也需要用到一个属性。

比如编译的 `FFmpeg` 动态库，

```cmkae
# 使用 IMPORTED 表示导入库
add_library(avcodec-57_lib SHARED IMPORTED)
# 使用 IMPORTED_LOCATION 属性指定库的路径
set_target_properties(avcodec-57_lib PROPERTIES IMPORTED_LOCATION
                        ${CMAKE_CURRENT_SOURCE_DIR}/src/main/jniLibs/armeabi/libavcodec-57.so )
```

## 链接到其他的库


如果编译了多个库，并且想库与库之间进行链接，那么就要通过 `target_link_libraries` 。

```cmake
target_link_libraries( native-lib
                       glm
                       turbojpeg
                       log )
```

在 Android 底层也提供了一些 so 库供上层链接使用，也要通过上面的方式来链接，比如最常见的就是 `log` 库打印日志。


如果要链接自己编译的多个库文件，首先要保证每个库的代码都对应一个 `CMakeLists.txt` 文件，这个 `CMakeLists.txt` 文件指定当前要编译的库的信息。

然后在当前库的 `CMakeLists.txt` 文件中通过 `ADD_SUBDIRECTORY` 将其他库的目录添加进来，这样才能够链接到。

```cmake
ADD_SUBDIRECTORY(src/main/cpp/turbojpeg)
ADD_SUBDIRECTORY(src/main/cpp/glm)
```

## 添加头文件

在使用的时候有一个容易忽略的步骤就是添加头文件，通过 `include_directories` 指令把头文件目录包含进来。

这样就可以直接使用 `#include "header.h"` 的方式包含头文件，而不用  `#include "path/path/header.h"` 这样添加路径的方式来包含。

## 小结

以上，就是关于 CMake 的部分总结内容。




