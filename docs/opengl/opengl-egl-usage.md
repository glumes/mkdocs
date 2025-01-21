---
title: "OpenGL 之 EGL 使用实践"
date: 2018-09-02T21:26:55+08:00
subtitle: ""
tags: ["OpenGL","EGL"]
categories: ["OpenGL"]
toc: true
 
draft: false
original: true
addwechat: true
slug: "opengl-egl-usage"
---

> 避免图片丢失，建议阅读微信原文：
> 
> https://mp.weixin.qq.com/s/AGMA4xvynzmdCNo-Caur-g

OpenGL 是跨平台的、专业的图形编程接口，而接口的实现是由厂商来完成的。

而当我们使用这组接口完成绘制之后，要把结果显示在屏幕上，就要用到 `EGL` 来完成这个转换工作。

<!--more-->

EGL 是 OpenGL ES 渲染 API 和本地窗口系统（native platform window system）之间的一个中间接口层，它也主要由厂商来实现。EGL 提供了如下机制：

*	与设备的原生窗口系统通信
*	查询绘图表面的可用类型和配置
*	创建绘图表面
*	在 OpenGL ES 和其他图形渲染 API 之间同步渲染
*	管理纹理贴图等渲染资源

为了让 OpenGL ES 能够绘制在当前设备上，我们需要 EGL 作为 OpenGL ES 与设备的桥梁。

我们可以直接用 GLSurfaceView 来进行 OpenGL 的渲染，就是因为在 GLSurfaceView 的内部已经完成了对 EGL 的使用封装，当然我们也可以封装自己的 EGL 环境。


## EGL 使用实践


EGL 的使用要遵循一些固定的步骤，按照这些步骤去配置、创建、渲染、释放。

*	创建与本地窗口系统的连接
	*	调用 eglGetDisplay 方法得到 EGLDisplay
*	初始化 EGL 方法
	*	调用 eglInitialize 方法初始化
*	确定渲染表面的配置信息
	*	调用 eglChooseConfig 方法得到 EGLConfig
*	创建渲染上下文
	*	通过 EGLDisplay 和 EGLConfig ，调用 eglCreateContext 方法创建渲染上下文，得到 EGLContext
*	创建渲染表面
	*	通过 EGLDisplay 和 EGLConfig ，调用 eglCreateWindowSurface 方法创建渲染表面，得到 EGLSurface
*	绑定上下文
	*	通过 eglMakeCurrent 方法将 EGLSurface、EGLContext、EGLDisplay 三者绑定，接下来就可以使用 OpenGL 进行绘制了。
*	交换缓冲
	*	当用 OpenGL 绘制结束后，使用 eglSwapBuffers 方法交换前后缓冲，将绘制内容显示到屏幕上
*	释放 EGL 环境
	*	绘制结束，不再需要使用 EGL 时，取消 eglMakeCurrent 的绑定，销毁  EGLDisplay、EGLSurface、EGLContext。


如果对 EGLDisplay、EGLSurface 、EGLContext 这些抽象概念傻傻分不清楚，可以参考这幅图：

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1fu8hoo85znj20g008q78w.jpg)

其中：

*	Display(EGLDisplay) 是对实际显示设备的抽象
*	Surface（EGLSurface）是对用来存储图像的内存区域
*	FrameBuffer 的抽象，包括 Color Buffer， Stencil Buffer ，Depth Buffer
*	Context (EGLContext) 存储 OpenGL ES绘图的一些状态信息


使用 EGL 的具体步骤如下：

### 创建与本地窗口系统的连接

通过 `eglGetDisplay` 方法创建与本地窗口系统的连接，返回的是 `EGLDisplay` 类型对象，可以把它抽象理解成设备的显示屏幕。

```java
    private EGLDisplay mEGLDisplay = EGL14.EGL_NO_DISPLAY;
    // 创建与本地窗口系统的连接
    mEGLDisplay = EGL14.eglGetDisplay(EGL14.EGL_DEFAULT_DISPLAY);
    // 如果创建之后还是 EGL_NO_DISPLAY ，表示创建失败
    if (mEGLDisplay == EGL14.EGL_NO_DISPLAY) {
        // failed
    }
```

这一步就可以看成是选择显示设备，一般都是选择默认的显示设备，也就是手机屏幕。

### 初始化 EGL 

创建 `EGLDisplay` 对象之后，要将它进行初始化。

```java
    // EGL 的主版本号
    private int[] mMajorVersion = new int[1];
    // EGL 的次版本号
    private int[] mMinorVersion = new int[1];
    boolean result = EGL14.eglInitialize(mEGLDisplay, mMajorVersion, 0, mMajorVersion, 0);
    if (!result) {
        // failed
    }
```

初始化的时候可以获得 EGL 的主版本号与次版本号。

### 确定可用的渲染表面（Surface）配置

把 Surface 看成是一个渲染表面，渲染表面包含一些配置信息，也就是 `EGLConfig` 属性：

比如：

*	EGL_RED_SIZE：颜色缓冲区中红色用几位来表示
*	EGL_BLUE_SIZE：颜色缓冲区中蓝色用几位来表示

更多的 EGLConfig 属性参考这里：

[https://www.jianshu.com/p/9db986365cda](https://www.jianshu.com/p/9db986365cda)。


有两种方式用来确定可用的渲染表面配置。

通过 `eglGetConfigs` 的方法获取底层窗口系统支持的所有 EGL 渲染表面配置，再用 `eglGetConfigAttrib` 查询每个 `EGLConfig` 的信息。
```java
    public static native boolean eglGetConfigs(
        EGLDisplay dpy,
        EGLConfig[] configs,
        int configsOffset,
        int config_size,
        int[] num_config,
        int num_configOffset
    );
	
    public static native boolean eglGetConfigAttrib(
        EGLDisplay dpy,
        EGLConfig config,
        int attribute,
        int[] value,
        int offset
    );
```

另一种是创建好渲染表面配置列表，通过 `eglChooseConfig` 的方法，找到符合要求的 EGLConfig 配置。

```java
    public static native boolean eglChooseConfig(
        EGLDisplay dpy,
        int[] attrib_list,
        int attrib_listOffset,
        EGLConfig[] configs,
        int configsOffset,
        int config_size,
        int[] num_config,
        int num_configOffset
    );
```

第二种方法相对于第一种来说，不用再查询每一个 EGLConfig 属性了。

具体使用如下：

```java
	// 定义 EGLConfig 属性配置
    private static final int[] EGL_CONFIG = {
            EGL14.EGL_RED_SIZE, CFG_RED_SIZE,
            EGL14.EGL_GREEN_SIZE, CFG_GREEN_SIZE,
            EGL14.EGL_BLUE_SIZE, CFG_BLUE_SIZE,
            EGL14.EGL_ALPHA_SIZE, CFG_ALPHA_SIZE,
            EGL14.EGL_DEPTH_SIZE, CFG_DEPTH_SIZE,
            EGL14.EGL_STENCIL_SIZE, CFG_STENCIL_SIZE,
            EGL14.EGL_RENDERABLE_TYPE, EGL14.EGL_OPENGL_ES2_BIT,
            EGL14.EGL_NONE,
    };
```

首先定义 EGLConfig 属性配置数组，定义红、绿、蓝、透明度、深度、模板缓冲的位数，最后要以 `EGL14.EGL_NONE` 结尾。

```java
        // 所有符合配置的 EGLConfig 个数
        int[] numConfigs = new int[1];
        // 所有符合配置的 EGLConfig
        EGLConfig[] configs = new EGLConfig[1];

        // configs 参数为 null,会在 numConfigs 中输出所有满足 EGL_CONFIG 条件的 config 个数
        EGL14.eglChooseConfig(mEGLDisplay, EGL_CONFIG, 0, null, 0, 0, numConfigs, 0);
        // 得到满足条件的个数
        int num = numConfigs[0];
        if (num != 0) {
            // 会获取所有满足 EGL_CONFIG 的 config
            EGL14.eglChooseConfig(mEGLDisplay, EGL_CONFIG, 0, configs, 0, configs.length, numConfigs, 0);
            // 去第一个
            mEGLConfig = configs[0];
        }
```

首先通过给 eglChooseConfig 相应参数设置为 null，找到符合条件的 EGLConfig 个数，如果不为空，则再一次调用，取第一个，就是想要的 `EGLConfig` 配置。


### 创建渲染上下文

有了 `EGLDisplay` 和 `EGLConfig` 对象，就可以创建 渲染表面 `EGLSurface`和 渲染上下文 `EGLContext`。

在创建渲染上下文时，同样要创建属性信息，主要是指定 OpenGL 使用版本。

```java
    private static final int[] EGL_ATTRIBUTE = {
            EGL14.EGL_CONTEXT_CLIENT_VERSION, 2,
            EGL14.EGL_NONE,
    };
```

同样，还是以 EGL14.EGL_NONE 结尾。
```java
    private EGLContext mEGLContext = EGL14.EGL_NO_CONTEXT;
    // 创建上下文
    mEGLContext = EGL14.eglCreateContext(mEGLDisplay, mEGLConfig, EGL14.EGL_NO_CONTEXT, EGL_ATTRIBUTE, 0);
    if (mEGLContext == EGL14.EGL_NO_CONTEXT) {
        // failed
    }
```

### 创建渲染表面

EGL 提供了两种方式创建渲染表面，一种是可见的，渲染到屏幕上，一种是不可见的，也就离屏的。

*	eglCreatePbufferSurface：创建离屏的渲染表面
*	eglCreateWindowSurface：创建渲染到屏幕的渲染表面

无论使用哪种创建方式，也都需要创建配置信息。

```java
	// 创建渲染表面的配置信息
    private static final int[] EGL_SURFACE = {
            EGL14.EGL_NONE,
    };
    private EGLSurface mEGLSurface = EGL14.EGL_NO_SURFACE;
    // surface 参数是有 SurfaceView.getHolder 传递过来的
    mEGLSurface = EGL14.eglCreateWindowSurface(mEGLDisplay, mEGLConfig, surface, EGL_SURFACE, 0);
```

对于渲染到屏幕上的创建，配置信息可以不添加什么，但还是要以 `EGL14.EGL_NONE` 结尾。

而对于离屏的渲染表面创建，就还需要提供宽、高等信息，但是却不需要提供 `surface` 的参数了。

```java
        // 使用 eglCreatePbufferSurface 就需要指定宽和高
        int[] EGL_SURFACE = {
                EGL10.EGL_WIDTH, w,
                EGL10.EGL_HEIGHT, h,
                EGL10.EGL_NONE,
        };
        mEGLSurface = mEGL.eglCreatePbufferSurface(mEGLDisplay, mEGLConfig, EGL_SURFACE,0);
```

### 绑定上下文

当有了 `EGLDisplay`、`EGLSurface`、`EGLContext` 对象之后，就可以为 `EGLContext` 绑定上下文了。

```java
 EGL14.eglMakeCurrent(mEGLDisplay, mEGLSurface, mEGLSurface, mEGLContext);
```

当绑定上下文之后，就可以执行具体的绘制操作了，调用 OpenGL 相关的方法绘制图形。

### 交换缓冲，进行显示

当绘制结束之后，就该把绘制结果显示在屏幕上了。

```java
        EGL14.eglSwapBuffers(mEGLDisplay, mEGLSurface);
```

通过 `eglSwapBuffers` 方法，将后台绘制的缓冲显示到前台。

### 释放操作

当绘制结束时，要进行相应的释放操作。

```java
        EGL14.eglMakeCurrent(mEGLDisplay, EGL14.EGL_NO_SURFACE, EGL14.EGL_NO_SURFACE, EGL14.EGL_NO_CONTEXT);
        EGL14.eglDestroyContext(mEGLDisplay, mEGLContext);
        EGL14.eglDestroySurface(mEGLDisplay, mEGLSurface);
        EGL14.eglReleaseThread();
        EGL14.eglTerminate(mEGLDisplay);

        mEGLContext = EGL14.EGL_NO_CONTEXT;
        mEGLDisplay = EGL14.EGL_NO_DISPLAY;
        mEGLConfig = null;
```

相当于就是把前面操作的 EGL 相关对象都释放掉。

当完成这样的一个流程之后，就基本上掌握了 EGL 的大部分使用。

 EGL 操作的很多流程都是固定的，可以把它们再单独做一层封装，这里可以参考 [grafika](https://github.com/google/grafika) 的封装。

文章中具体代码部分，可以参考我的 Github 项目，欢迎 Star 。

> [https://github.com/glumes/AndroidOpenGLTutorial](https://github.com/glumes/AndroidOpenGLTutorial)

## 参考

1. [Android 系统图形栈（一）： OpenGL ES 和 EGL 介绍](https://woshijpf.github.io/android/2017/09/04/Android%E7%B3%BB%E7%BB%9F%E5%9B%BE%E5%BD%A2%E6%A0%88OpenGLES%E5%92%8CEGL%E4%BB%8B%E7%BB%8D.html)
2. [Understaing Android EGL](https://www.slideshare.net/SuhanLee2/understaing-android-egl)
3. [OpenGL ES EGL介绍](https://blog.csdn.net/cauchyweierstrass/article/details/53189449)
4. [引入EGL](https://www.jianshu.com/p/9db986365cda)