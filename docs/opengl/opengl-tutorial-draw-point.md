---
title: "OpenGL 学习系列之基础的绘制流程"
date: 2017-12-22T16:19:28+08:00
categories: ["OpenGL"]
tags: ["OpenGL"]
 
toc: true
original: true
addwechat: true
slug: "opengl-tutorial-draw-point"
---

> 避免图片丢失，建议阅读微信原文：
> 
> https://mp.weixin.qq.com/s/H6p3cLGpKys5ihF7ymBhJw

终于要开始探索奇妙的 3D 世界了，OpenGL 搞起。

<!--more-->

## OpenGL 简介

OpenGL 是一种应用程序编程接口，它是一种可以对图形硬件设备特性进行访问的软件库。

重点：OpenGL 是一种接口，既然是接口，那么就必然要有实现。

事实上，它的实现是由显示设备厂商提供的，而且依赖于厂商提供的硬件设备。

OpenGL 常用于 CAD、虚拟实境、科学可视化程序和电子游戏开发。

在 Android 上使用的是 OpenGL ES，它是 OpenGL 的子集，在 OpenGL 的基础之上裁剪掉了一些非必要的部分，主要是针对手机、PAD 和游戏主机等嵌入式设备设计的。

在 Android 上开发 OpenGL 既可以使用 Java 也可以使用 C ，话不多说，撸起袖子就是干！


## OpenGL 的绘制流程

学习 OpenGL 的绘制，最好还是先从 2D 绘制开始，逐渐过渡到 3D 绘制。

Android 为 OpenGL 的绘制提供了一个特定的视图`GLSurfaceView`，就像 SurfaceView 一样，它渲染绘制也可以在一个单独的线程中，而非主线程，毕竟 GLSurfaceView 就是继承自 SurfaceView 的。

在使用 GLSurfaceView 时，需要通过`setRenderer`方法给它设置一个渲染器，而主要的渲染工作就是由渲染器`Renderer`完成了。

通过继承`GLSurfaceView.Renderer`类来实现我们自己的渲染器程序，主要有如下三个方法：

*	onSurfaceCreated
	*	当 GLSurfaceView 创建时调用，主要做一些准备工作。
*	onSurfaceChanged
	*	当 GLSurfaceView 视图改变时调用，第一次创建时也会被调用。
*	onDrawFrame
	*	每一帧绘制时被调用。

实现渲染器程序时，首先要考虑三个问题：

*	在什么地方进行绘制？
*	绘制成什么形状？
*	用什么颜色来绘制？

而我们的程序也主要以解决上述三个问题为主，下面以 OpenGL 绘制一个点来讲解。

### OpenGL 坐标

手机屏幕的坐标是以左上角为原点（0，0），向右为 X 轴正方形，向下为 Y 轴正方向，而 OpenGL 也有着它自己的一套坐标定义。

假设我们定义了一个点的坐标（4.3，2.1），也就是它的 X 轴坐标和 Y 轴坐标，而 OpenGL 最后会把我们定义的坐标映射手机屏幕的实际物理坐标上。

无论是 X 坐标还是 Y 坐标，OpenGL 都会把手机屏幕映射到 $[-1，1$] 的范围内。也就是说：屏幕的左边对应 X 轴的 -1 ，屏幕的右边对应 +1，屏幕的底边会对应 Y 轴的 -1，而屏幕的顶边就对应 +1。

不管屏幕是什么形状和大小，这个坐标范围都是一样的，例如下图所示：

![](https://image.glumes.com/images/2019/05/12/opengl_coordinate.png)

所以，上面定义的坐标（4.3，2.1），最后是会被映射到手机屏幕之外的，处于不可见的状态。

这里，假定绘制一个位于原点的点（0，0），那么映射之后的位置就手机屏幕的中心了。

### 基本图元

解决了位置的问题，接下来就是形状和颜色的问题。

如同 Android 的 Canvas 对象提供了一些方法来完成基本的绘制：drawPoint、drawRect、drawLine 等，OpenGL 程序也提供且仅提供了三种基本的图元来完成绘制。

*	点
*	线
*	三角形

其他的所有形状都是基于这三种图元来完成的，比如矩形就可以看成是两个三角形拼成的。

由于我们要绘制的是一个点，在坐标系中，一个坐标就可以代替一个点了。假设要绘制一个三角形，那么在坐标系中就需要三个点才行了。

接下来就涉及到 OpenGL 如何把定义的点的数据绘制出来了。

### 渲染管线

首先要明白一个概念`渲染管线`。

根据百度百科的定义，渲染管线也称为`渲染流水线`或`像素流水线`或`像素管线`，是显示芯片内部（GPU）处理图形信号相互独立的并行处理单元。

显卡的渲染管线就是显示核心的重要组成部分，是负责给图形配上颜色的一组专门通道。渲染管线的数量是决定显示芯片性能和档次的最重要的参数之一。

现阶段的显卡都是分为`顶点渲染`和`像素渲染`的。在显卡，内部分为两大区域，一个区域是顶点渲染单元（也叫顶点着色），主要负责描绘图形，也就是建立模型。一个区域是像素渲染管线，主要负责把顶点绘出的图形填上颜色。


![](https://image.glumes.com/images/2019/05/12/opengl_draw_point.png)

上图就是 OpenGL 中渲染管线的一个处理流程。

可以看到，流程图从读取顶点数据开始，然后后执行两个着色器：

*	顶点着色器
	*	主要负责描绘图形，也就是根据顶点坐标，建立图形模型。
*	片段着色器
	*	主要负责把顶点绘出的图形填上颜色。


由于这两个着色器对于最后图形显示效果至关重要，并且它们还是可以通过编程来控制的，这也是为什么`可编程渲染管线`要优于固定编程管线了。

事实上，随着显示技术的发展，渲染管线将不复存在了，顶点着色器和渲染管线统一被`流处理器（Stream Processors）`所取代。

但是目前手机上 OpenGL 还是使用渲染管线中，有了渲染管线，我们就可以完成点的形状绘制和着色两大问题了，接下来的工作也是围绕这条渲染管线开始的。

### 内存拷贝

当定义完了顶点坐标，并且明确了下一步：**顶点坐标将要通过渲染管线进行一系列处理**，那么接下来就是如何把顶点坐标传递给渲染管线了。

OpenGL 的实现是由显示设备厂商提供的，它作为本地系统库直接运行在硬件上。而我们定义的顶点 Java 代码是运行在虚拟机上的，这就涉及到了如何把 Java 层的内存复制到 Native 层了。


一种方法是直接使用`JNI`开发，直接调用本地系统库，也就是用 `C++` 来开发 OpenGL，这种实现肯定要学会的。

另一种方法就是在 Java 层把内存块复制到 Native 层。

使用`ByteBuffer.allocateDirect()`方法就可以分配一块 Native 内存，这块内存不会被 Java 的垃圾回收器管理。

它的使用方法大致都一样，抽出公共的模板：

``` java
   // 声明一个字节缓冲区 FloatBuffer
   private FloatBuffer floatBuffer;
   // 定义顶点数据
   float[] vertexData = new float[16];
   // FloatBuffer 初始化工作并放入顶点数据
   floatBuffer = ByteBuffer
       .allocateDirect(vertexData.length * Constant.BYTES_PRE_FLOAT)
       .order(ByteOrder.nativeOrder())
       .asFloatBuffer()
       .put(vertexData);
```

在`allocateDirect`方法分配了内存并指定了大小之后，下一步就是告诉 ByteBuffer 按照`本地字节序`组织它的内容。本地字节序是指，当一个值占用多个字节时，比如 32 位整型数，字节按照从最重要位到最不重要位或者相反顺序排列。

接下来`asFloatBuffer`方法可以得到一个反映底层字节的 FloatBuffer 类实例，避免直接操作单独的字节，而是使用浮点数。

最后，通过`put`方法就可以把数据从 Java 层内存复制到 Native 层了，当进程结束时，这块内存就会被释放掉。


### 顶点着色器

接下来可编程的部分了，定义着色器（`Shader`）程序。

使用不同的着色器对输入的图元数据执行计算操作，判断它们的位置、颜色，以及其他渲染属性。

首先是顶点着色器。

在渲染管线中传输的每个顶点坐标位置，OpenGL 都会调用一个顶点着色器来处理顶点相关的数据，这个处理过程可以很复杂，也可以很简单。

想要定义一个着色器程序，还要通过一种特殊的语言去编写：`OpenGL Shading Language`，简称`GLSL`.

`GLSL`语言类似于 C 语言或者 Java 语言，它的程序入口也是一个名为`main`的函数。关于 GLSL 的部分，完全可以单独写一篇博客了，暂时先不详细阐述。

下面就是一个简单的顶点着色器程序：
``` java
attribute vec4 a_Position;
void main()
{
    gl_Position = a_Position;
    gl_PointSize = 30.0;
}
```
着色器类似于一个函数调用的方式——数据传输进来，经过处理，然后再传输出去。

其中，`gl_Position`和`gl_PointSize`就是着色器中的特殊全局变量，它接收输入。

`a_Position`就是我们定义的一个变量，它是`vec4`类型的。而`attribute`只能存在于顶点着色器中，一般用于保存顶点数据，它可以在数据缓冲区中读取数据。

数据缓存区中的顶点坐标会赋值给 a_Position ，a_Position 会传递给 gl_Position。

而 gl_PointSize 则固定了点的大小为 30。

有了顶点着色器，就能够为每个顶点生成最终的位置，接下来就是定义片段着色器。

根据上图的渲染管线，顶点着色器到片段着色器之间，还要经过`组装图元`和`光栅化图元`。


### 光栅化技术

移动设备的显示屏由成百上千个小的、独立的部件组成，他们称为`像素`。每个像素通常由三个单独的子组件构成，它们发出红色、绿色和蓝色的光，因为每个像素都非常小，人的眼睛会把红色、绿色和蓝色的光混合在一起，从而创造出巨量的颜色范围。

OpenGL 就是通过 *光栅化* 技术的过程把每个点、直线及三角形分解成大量的小片段，它们可以映射到移动设备显示屏的像素上，从而生成一幅图像。这些片段类似于显示屏上的像素，每一个都包含单一的纯色。

如下图所示：

![](https://image.glumes.com/images/2019/05/12/opengl_guangshanhua.png)

OpenGL 通过光栅化技术把一条直线映射为一个片段集合，显示系统通常会把这些片段直接映射到屏幕上的像素，结果一个片段就对应一个像素。

明白了这样的显示原理，就可以在其中做一些操作了，这就是片段着色器的功能了。

### 片段着色器

片段着色器的主要目的就是告诉 GPU 每个片段的最终颜色应该是什么。

对于基本图元的每个片段，片段着色器都会被调用一次，因此，如果一个三角形被映射到 10000 个片段，那么片段着色器就会被调用 10000 次。

下面就是一个简单的片段着色器程序：

``` java
precision mediump float;
uniform vec4 u_Color;
void main()
{
    gl_FragColor = u_Color;
}
```

其中，`gl_FragColor`变量就是 OpenGL 最终渲染出来的颜色的全局变量，而`u_Color`就是我们定义的变量，通过在 Java 层绑定到 `u_Color`变量并给它赋值，就会传递到 Native 层的`gl_FragColor`中。

而第一行的`mediump`指的就是片段着色器的精度了，有三种可选，这里用中等精度就行了。`uniform`则表示该变量是不可变的了，也就是固定颜色了，目前显示固定颜色就好了。


### 编译 OpenGL 程序

明白了着色器的功能和光栅化技术之后，对渲染管线的流程也就更加清楚了，接下来就是编译 OpenGL 的程序了。

编译 OpenGL 程序基本流程如下：

*	编译着色器
*	创建 OpenGL 程序和着色器链接
*	验证 OpenGL 程序
*	确定使用 OpenGL 程序

#### 编译着色器

创建新的文件编写着色器程序，然后再从文件以字符串的形式中读取文件内容。这样会比把着色器程序写成字符串的形式更加清晰。

当读取了着色器程序内容之后，就可以编译了。

``` java
   // 编译顶点着色器
    public static int compileVertexShader(String shaderCode) {
        return compileShader(GL_VERTEX_SHADER, shaderCode);
    }
    
    // 编译片段着色器
    public static int compleFragmentShader(String shaderCode) {
        return compileShader(GL_FRAGMENT_SHADER, shaderCode);
    }

    // 根据类型编译着色器
    private static int compileShader(int type, String shaderCode) {
	    // 根据不同的类型创建着色器 ID
        final int shaderObjectId = glCreateShader(type);
        if (shaderObjectId == 0) {
            return 0;
        }
        // 将着色器 ID 和着色器程序内容连接
        glShaderSource(shaderObjectId, shaderCode);
        // 编译着色器
        glCompileShader(shaderObjectId);
		// 以下为验证编译结果是否失败
        final int[] compileStatsu = new int[1];
        glGetShaderiv(shaderObjectId, GL_COMPILE_STATUS, compileStatsu, 0);
        if ((compileStatsu[0] == 0)) {
	        // 失败则删除
            glDeleteShader(shaderObjectId);
            return 0;
        }
        return shaderObjectId;
    }
```

以上程序主要就是通过`glCreateShader`方法创建了着色器 ID，然后通过`glShaderSource`连接上着色器程序内容，接下来通过`glCompileShader`编译着色器，最后通过`glGetShaderiv`验证是否失败。

`glGetShaderiv`函数比较通用，在着色器阶段和 OpenGL 程序阶段都会通过它来验证结果。


####	创建 OpenGL 程序和着色器链接

接下来就是创建 OpenGL 程序并加着色器加进来。

``` java
  public static int linkProgram(int vertexShaderId, int fragmentShaderId) {
		// 创建 OpenGL 程序 ID
        final int programObjectId = glCreateProgram();
        if (programObjectId == 0) {
            return 0;
        }
        // 链接上 顶点着色器
        glAttachShader(programObjectId, vertexShaderId);
        // 链接上 片段着色器
        glAttachShader(programObjectId, fragmentShaderId);
        // 链接着色器之后，链接 OpenGL 程序
        glLinkProgram(programObjectId);
        final int[] linkStatus = new int[1];
        // 验证链接结果是否失败
        glGetProgramiv(programObjectId, GL_LINK_STATUS, linkStatus, 0);
        if (linkStatus[0] == 0) {
	        // 失败则删除 OpenGL 程序
            glDeleteProgram(programObjectId);
            return 0;
        }
        return programObjectId;
    }
```

首先通过`glCreateProgram`程序创建 OpenGL 程序，然后通过`glAttachShader`将着色器程序 ID 添加上 OpenGL 程序，接下来通过`glLinkProgram`链接 OpenGL 程序，最后通过`glGetProgramiv`来验证链接是否失败。


#### 验证 OpenGL 程序

链接了 OpenGL 程序后，就是验证 OpenGL 是否可用。

``` java
 public static boolean validateProgram(int programObjectId) {
        glValidateProgram(programObjectId);
        final int[] validateStatus = new int[1];
        glGetProgramiv(programObjectId, GL_VALIDATE_STATUS, validateStatus, 0);
        return validateStatus[0] != 0;

    }
```

通过`glValidateProgram`函数验证，并再次通过`glGetProgramiv`函数验证是否失败。

#### 确定使用 OpenGL 程序

当一切完成后，就是确定使用该 OpenGL 程序了。

``` java
// 创建 OpenGL 程序过程
 public static int buildProgram(Context context, int vertexShaderSource, int fragmentShaderSource) {
        int program;

        int vertexShader = compileVertexShader(
                TextResourceReader.readTextFileFromResource(context, vertexShaderSource));

        int fragmentShader = compleFragmentShader(
                TextResourceReader.readTextFileFromResource(context, fragmentShaderSource));

        program = linkProgram(vertexShader, fragmentShader);

        validateProgram(program);

        return program;
    }

// 创建完毕后，确定使用
    mProgram = ShaderHelper.buildProgram(context, R.raw.point_vertex_shader
                , R.raw.point_fragment_shader);

        glUseProgram(mProgram);
```

将上述的过程合并起来，`buildProgram`函数返回的 OpenGL 程序 ID 即可，通过`glUseProgram`函数表示使用该 OpenGL 程序。


### 绘制

完成了 OpenGL 程序的编译，就是最后的绘制了，再回到渲染器 `Renderer`里面。

``` java
public class PointRenderer extends BaseRenderer {

    private Point mPoint;
    public PointRenderer(Context mContext) {
        super(mContext);
    }

    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {
        super.onSurfaceCreated(gl, config);
        glClearColor(0.0f, 0.0f, 0.0f, 0.0f);
        // 在 onSurfaceCreated 里面初始化，否则会报线程错误
        mPoint = new Point(mContext);
        // 绑定相应的顶点数据
        mPoint.bindData();
    }
    
    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        // 确定视口大小
        glViewport(0, 0, width, height);
    }

    @Override
    public void onDrawFrame(GL10 gl) {
        // 清屏
        glClear(GL_COLOR_BUFFER_BIT);
        // 绘制
        mPoint.draw();
    }
}
```

在`onSurfaceCreated`函数里面做初始化工作，绑定数据等；在`onSurfaceChanged`方法里面确定视图大小，在`onDrawFrame`里面执行绘制。

为了简化渲染流程，把所有的操作都放在放在要渲染的对象里面去了，声明一个 Point 对象，代表要绘制的点。

``` java
public class Point extends BaseShape {

    // 着色器中定义的变量，在 Java 层绑定并赋值
    private static final String U_COLOR = "u_Color";
    private static final String A_POSITION = "a_Position";
    private int aColorLocation;
    private int aPositionLocation;
    
    float[] pointVertex = {
            0f, 0f
    };

    public Point(Context context) {
        super(context);
        mProgram = ShaderHelper.buildProgram(context, R.raw.point_vertex_shader
                , R.raw.point_fragment_shader);
        glUseProgram(mProgram);
        vertexArray = new VertexArray(pointVertex);
        POSITION_COMPONENT_COUNT = 2;
    }

    @Override
    public void bindData() {
        //绑定值
        aColorLocation = glGetUniformLocation(mProgram, U_COLOR);
        aPositionLocation = glGetAttribLocation(mProgram, A_POSITION);
        // 给绑定的值赋值，也就是从顶点数据那里开始读取，每次读取间隔是多少
        vertexArray.setVertexAttribPointer(0, aPositionLocation, POSITION_COMPONENT_COUNT,
                0);
    }

    @Override
    public void draw() {
        // 给绑定的值赋值
        glUniform4f(aColorLocation, 0.0f, 0.0f, 1.0f, 1.0f);
        glDrawArrays(GL_POINTS, 0, 1);
    }
}
```

在 Point 的构造函数中，编译并使用 OpenGL 程序，而在 `bindData`函数中，通过`glGetUniformLocation`和`glGetAttribLocation`函数绑定了我们在 OpenGL 中声明的变量`u_Color`和`a_Position`，要注意的是，`attribute`类型和`uniform`类型所对应的方法是不同的，最后通过给`POSITION_COMPONENT_COUNT`变量赋值，指定每个顶点数据的个数为 2 。

绑定了变量之后，接下来就是给他们赋值了，对于`uniform`类型变量，由于是固定值，所以直接调用`glUniform4f`方法给其赋值就好了，而`attribute`类型变量，则需要对应顶点数据中的值了，`vertexArray.setVertexAttribPointer`方法就是完成这个任务的。

``` java
	// 给某个顶点数据绑定值，并 Enable 使能
    public void setVertexAttribPointer(int dataOffset, int attributeLocation, int componentCount, int stride) {
        floatBuffer.position(dataOffset);
        glVertexAttribPointer(attributeLocation, componentCount, GL_FLOAT, false, stride, floatBuffer);
        glEnableVertexAttribArray(attributeLocation);
        floatBuffer.position(0);
    }
```

在`setVertexAttribPointer`方法中，使用了`glVertexAttribPointer`方法来绑定值，它的参数释义如下所示：

![https://image.glumes.com/images/2019/05/12/opengl_canshu.png](https://image.glumes.com/images/2019/05/12/opengl_canshu.png)

通过`glEnableVertexAttribArray`方法来开启使用即可。

最后通过` glDrawArrays`方法来执行最后的绘制，`GL_POINTS`代表绘制的类型，而参数`0，1`则代表绘制的点的范围，它是一个左闭右开的区间。

以上步骤就完成了一个点的绘制，如图所示：

![](https://image.glumes.com/images/2019/05/12/opengl_draw_p.png)

具体代码详情，可以参考我的 Github 项目：

[https://github.com/glumes/AndroidOpenGLTutorial](https://github.com/glumes/AndroidOpenGLTutorial)

### 小结

使用 OpenGL 进行绘制的原理，也就是按照 GPU 的渲染管线流程，提供了顶点数据之后，执行顶点着色器，然后执行片段着色器，最后映射到手机屏幕上。

而作为可编程的阶段，我们就是在顶点着色器和片段着色器中做我们想要的处理，编写了着色器代码之后，通过编译链接成 OpenGL 程序。然后给 OpenGL 中设定的变量绑定对应的值，从顶点数据何处开始读取值。到这里，一切准备工作就做完了。

最后就在在渲染器 Renderer 中开始绘制了。



## 参考

1. 《OpenGL ES 应用开发实践指南》
2. 《OpenGL 编程指南》（原书第八版）
3. https://github.com/glumes/AndroidOpenGLTutorial



