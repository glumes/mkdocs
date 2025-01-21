---
title: "rust 开发编译 Android 动态库实践"
date: 2019-05-19T23:25:31+08:00
subtitle: ""
tags: ["rust","so"]
categories: ["Android"]
toc: true
draft: false
original: true
 
slug: "rust-compile-so-library"
---

![](https://image.glumes.com/images/2019/05/19/rust-mm.png)


最近关注了一波 rust，一门目前还比较小众但却很强大的编程语言，官网地址如下：

> [https://www.rust-lang.org/](https://www.rust-lang.org/)


rust 的学习曲线比较陡峭，在开始学习之前建议看看王垠的这篇文章 《如何掌握所有的编程语言》，地址如下:

> [https://www.yinwang.org/blog-cn/2017/07/06/master-pl](https://www.yinwang.org/blog-cn/2017/07/06/master-pl)

学习语言，重要的是掌握其语言特性。

王垠举了一些语言特性的例子：

* 变量定义
* 算术运算
* for 循环语句，while 循环语句
* 函数定义，函数调用
* 递归
* 静态类型系统
* 类型推导
* lambda 函数
* 面向对象
* 垃圾回收
* 指针算术
* goto 语句

看着这些特性是不是很像一些编程语言书的目录🤔

在学习 rust 的时候也可以照着这些语言特性去对比自己是否掌握了。

<!--more-->

那么 rust 到底强大在哪里？在 Kotlin 刚出的时候宣传的点就是空安全 ，弥补 Java 在这方面的不足。而 rust 可以说对比的是 C++，弥补 C++ 在空指针和野指针（悬垂指针）方面的不足，当然 rust 的优势还不足如此。

以下是来自维基百科的介绍，有些特性我暂时还没体验过，先摘录一波：

> Rust 是由 Mozilla 主导开发的通用、编译型编程语言。。设计准则为“安全、并发、实用”，支持函数式、并发式、过程式以及面向对象的编程风格。

目前国内也已经有一些团队在用 rust 进行开发了，可以在观望一波后，再决定是否投入精力入坑~~~

---


## rust 编译 so 实践

下面是用 rust 编译 Android 动态库实践，主要参考了 Mozilla 给的例子，具体链接在后面。


### 安装 rust 

首先是安装 rust ，通过如下命令安装：

```sh
curl https://sh.rustup.rs -sSf | sh
```

然后通过如下命令输出 rust 版本以及验证是否安装成功。

```sh
rustc --version
```

### 配置 rust 交叉编译

确保 Android 的 sdk 目录配置到了系统环境中。

```sh
export ANDROID_HOME=/Users/$USER/Library/Android/sdk
export NDK_HOME=$ANDROID_HOME/ndk-bundle
```

然后执行如下命令：

```sh
mkdir NDK
${NDK_HOME}/build/tools/make_standalone_toolchain.py --api 26 --arch arm64 --install-dir NDK/arm64
${NDK_HOME}/build/tools/make_standalone_toolchain.py --api 26 --arch arm --install-dir NDK/arm
${NDK_HOME}/build/tools/make_standalone_toolchain.py --api 26 --arch x86 --install-dir NDK/x86
```

主要是执行 ndk-bundle 目录下的 python 脚本，该脚本在 NDK 目录中去创建 arm64、arm、x86 架构平台的编译环境。

在把环境添加到 rust 的配置中。

创建 `cargo-config.toml` 文件，输入如下内容：

```sh
[target.aarch64-linux-android]
ar = "<path>/NDK/arm64/bin/aarch64-linux-android-ar"
linker = "<path>/NDK/arm64/bin/aarch64-linux-android-clang"

[target.armv7-linux-androideabi]
ar = "<path>/NDK/arm/bin/arm-linux-androideabi-ar"
linker = "<path>/NDK/arm/bin/arm-linux-androideabi-clang"

[target.i686-linux-android]
ar = "<path>/NDK/x86/bin/i686-linux-android-ar"
linker = "<path>/NDK/x86/bin/i686-linux-android-clang"
```

记得把 <path> 替换成 NDK 文件夹所在的路径。

然后执行：

```sh
cp cargo-config.toml ~/.cargo/config
```

把 cargo-config.toml 移动到 `~/.cargo/` 目录下。

再执行如下命令：

```sh
rustup target add aarch64-linux-android armv7-linux-androideabi i686-linux-android
```

以上就完成了交叉编译环境的配置。

### rust 开发及编译

现在要涉及到具体的 rust 开发了，推荐使用 JetBrains 系列的 IntelliJ IDEA ，无需激活，使用社区版就行，安装 rust 插件就可以愉快地编写代码了。

执行以下命令创建工程目录：

```sh
mkdir rust-android
cd rust-android
```

然后创建 rust 项目：
```sh
cargo new rust --lib
cd rust
```

`rust` 代表创建的项目所在文件夹名字。

`--lib` 代表我们创建的是一个库项目，而不是一个执行的二进制文件。

项目结构图如下：

![](https://image.glumes.com/images/2019/05/18/-2019-05-18-4.32.01.png)

结构很简单，就两个文件，本篇文章也不会自己去新增文件，当然肯定是会有编译文件出现的。

`Cargo.toml` 就相当于 Android 中的 build.gradle ，一些第三方库都是通过这里去引用的。

`lib.rs` 就是具体编写代码的地方了。

首先要在 Cargo.toml 中去添加一些内容：

```sh
[dependencies]
# 添加 jni 依赖，并指定版本
jni = { version = "0.10.2", default-features = false }

[profile.release]
lto = true

[lib]
# 编译的动态库名字
name = "rust"
# 编译类型 cdylib 指定为动态库
crate-type = ["cdylib"]
```

关于添加的内容含义如上注释，主要是添加了 jni 依赖和指定编译，有了 jni 依赖才能够让 Android 项目代码调用到 so 动态库的内容。

接下来就是编写 rust 代码了。

```rust
// 指定 os 类型
#![cfg(target_os = "android")]
// 允许不是 snake_case 的命令方式
#![allow(non_snake_case)]

// 引用标准库的一些内容
use std::ffi::{CString, CStr};
// 引用 jni 库的一些内容，就是上面添加的 jni 依赖
use jni::JNIEnv;
use jni::objects::{JObject, JString};
use jni::sys::jstring;

#[no_mangle]
// jni 函数调用的头
pub unsafe extern fn Java_com_glumes_rust_MainActivity_hello(
    env: JNIEnv, _: JObject, j_recipient: JString) -> jstring {
    // 将 Android 上层传来的字符串转出 rust 中的字符串变量
    let recipient = CString::from(
        CStr::from_ptr(
            env.get_string(j_recipient).unwrap().as_ptr()
        )
    );
    // 返回一个新的字符串
    let output = env.new_string("hello".to_owned() + recipient.to_str().unwrap()).unwrap();
    output.into_inner()
}
```

抛开 rust 具体语法不看，代码内容也很简单，传一个字符串，返回一个新的字符串。


接下来就是执行具体的编译：

```sh
cargo build --target aarch64-linux-android --release
cargo build --target armv7-linux-androideabi --release
cargo build --target i686-linux-android --release
```

分别执行上面三个命令，会分别构建三个不同平台的 so 库。

然后把 so 库替换到 Android 项目的 jniLibs 具体文件夹就好了。

![](https://image.glumes.com/images/2019/05/18/-2019-05-18-4.56.37.png)

这样就完成了用 rust 编译 Android 平台的 so 动态库，并且每次编译后的时候就要进行 so 的替换，当然也可以想办法把 rust so 的编译放在 Android gradle 的编译过程中，就和用 CMake 编译一样。

## 问题和思考

以上只是一个小小的例子，想用 rust 实现像 C++ 那样去开发动态库，可能还一些坑要去探索。仅仅是实现 jni 的调用还是远不够的，在 NDK 开发里面还有很多头文件，如何去在 rust 里面去实现调用？

如果想要 rust 去打印 Android NDK 中的 log ，倒是可以参考 [android_logger-rs](https://github.com/Nercury/android_logger-rs) 项目，至于其他的慢慢摸索中。

另外还有个问题，如何在 Android 中去断点调试 rust 的代码，在网上搜索了一番，还没找到合适的答案，有知道的朋友可以指点我一二 🥺


## 参考

具体的例子可以参考我的 GitHub 项目：

> [https://github.com/glumes/rust-android-example](https://github.com/glumes/rust-android-example)



1. https://www.yinwang.org/blog-cn/2017/07/06/master-pl
2. https://mozilla.github.io/firefox-browser-architecture/experiments/2017-09-21-rust-on-android.html


