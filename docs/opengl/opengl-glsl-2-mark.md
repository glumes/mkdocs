---
title: "OpenGL ES 2.0 着色器语言 GLSL 学习 Mark "
date: 2018-09-09T15:07:20+08:00
subtitle: ""
tags: ["OpenGL","GLSL"]
categories: ["OpenGL"]
toc: true
 
draft: false
original: true
addwechat: true
slug: "opengl-glsl-2-mark"
---

要想发挥 OpenGL ES 自定义渲染管线的功能，就得学会写 GLSL 着色器脚本。

本文中的内容来自于 《Android 3D 游戏开发技术宝典 OpenGL ES 2.0》。

<!--more-->

## 数据类型

OpenGL ES 着色器语言和 C/C++ 语法相近，但还是有很大不同，比如 GLSL 不支持 double、byte、short、long、union、enum、unsigned 等。

GLSL 的数据类型可以分为标量、向量、矩阵、采样器、结构体以及数组等几类。

### 标量

标量也被称为 "无向量"，其值只有大小，并不具有方向。

标量之间的运算遵循简单的代数法则。

GLSL 支持的的变量类型有如下：

*	布尔型 bool
*	整型 int
*	浮点型 float


### 向量

在 GLSL 中，向量可以看做是同样类型的标量组成的，其基本类型也分为 bool、int 及 float 3 种。

每个向量可以由 2 个、3 个或者 4 个相同的标量组成。

|向量类型|说明|向量类型|说明|
|---|---|---|---|
|vec2|包含了 2 个浮点数的向量|ivec4|包含了 4 个整数的向量|
|vec3|包含了 3 个浮点数的向量|bvec2|包含了 2 个布尔数的向量|
|vec4|包含了 4 个浮点数的向量|bvec3|包含了 3 个布尔数的向量|
|ivec2|包含了 2 个整数的向量|bvec4|包含了 4 个布尔数的向量|
|ivec3|包含了 3 个整数的向量|||


### 矩阵

矩阵按尺寸分为 2*2 矩阵、3*3 矩阵以及 4*4 矩阵。

|矩阵类型|说明|
|---|---|
|mat2|2x2 的浮点数矩阵|
|mat3|3x3 的浮点数矩阵|
|mat4|4x4 的浮点数矩阵|

### 采样器

采样器是专门进行纹理采样的相关操作。

|采样器类型|说明|采样器类型|说明|
|---|---|---|---|
|sampler2D|用于访问二维纹理|sampleCube|用于访问立方贴图纹理|
|sampler3D|用于访问三维纹理|

### 结构体

使用 `struct` 关键字声明结构体。

```glsl
struct info{
	vec3 color;
	vec3 position;
	vec2 textureCoor;
}

info CubeInfo;
```

### 数组

在 GLSL 中，可以声明任何类型的数组。声明数组的方式主要有两种，具体如下：

*	在声明数组的同时，指定数组大小。

```glsl
vec3 position[20];

```
*	在声明数组时，也可以不指定数组的大小，但必须符合下列两种情况之一：

引用数组之前，要再次使用第一种声明方式来声明该数组

```glsl
vec3 position[];
vec3 position[5];
```
	
代码中访问数组的下标都是编译时常量（如：字面常量），这时编译器会自动创建适当大小的数组，使得数组尺寸组织存储编译器看到的最大索引值对应的元素。


```glsl
vec3 position[];
position[3] = vec3(3.0); // 需要一个大小为 4 的数组
position[20] = vec3(6.0); // 需要一个大小为 21 的数组
```


### 空类型

空类型使用 `void` 表示，仅用来声明不返回任何值的函数，比如着色器中的 main 函数就是一个返回值为 `void` 类型的函数。

### 限定符

着色器对变量由很多可选的限定符，这些限定符中大部分只能用来修饰全局变量，如下表：

|限定符|说明|
|---|---|
|attribute|一般用于每个顶点都各不相同的量，如顶点位置、颜色等|
|uniform|一般用于对同一组顶点组成的单个 3D 物体中所有顶点都相同的量，如当前的光源位置|
|varying|用于从顶点着色器传递到片元着色器的量|
|const|用于声明常量|

```glsl
uniform mat4 uMVPMatrix;
attribute vec3 aPosition;
varying vec4 aaColor;
const int lightsCount = 4;
```

限定符的使用应该放在变量前面，且使用 attribute、uniform 以及  varying 限定符修饰的变量必须为全局变量。

着色器语言中没有默认限定符的概念，因此如果有需要，必须为全局变量明确指定需要的限定符。

*	attribute 限定符

attribute 限定符顾名思义为属性限定符，其修饰的变量用来接收渲染管线传递进顶点着色器的当前待处理顶点的各种属性值。这些属性值每个顶点各自拥有独立的副本，用于描述顶点的各项特征，如顶点坐标、法向量、颜色、纹理坐标等。

用 attribute 限定符修饰的变量其值是由应用程序批量传入渲染管线的，管线进行基本处理后再传递给顶点着色器。数据中有多少个顶点，管线就调用多少次顶点着色器，每次将一个顶点的各种属性数据传递给顶点着色器中对应的 attribute 变量。因此，顶点着色器每次执行将完成对一个顶点各项属性数据的处理。

attribute 限定符只能用于顶点着色器中，不能再片元着色器中使用。而且，attribute 限定符只能用来修饰浮点数标量、浮点数向量以及矩阵变量，不能用来修饰其他类型的变量。

* uniform 限定符

uniform 为一致变量限定符，一致变量指的是对于同一组顶点组成的单个 3D 物体中所有顶点都相同的量。

uniform 变量可以用在顶点着色器中或片元着色器中，其支持用来修饰所有的基本数据类型，与 attribute 变量类似，一致变量的值也是从应用程序传入的。

* varying 限定符

要想将顶点着色器中的信息传入到片元着色器，则必须使用 varying 限定符。用 varying 限定符修饰的全局变量又称为易变变量，易变变量可以看成是顶点着色器以及片元着色器之间的动态接口，方便顶点着色器与片元着色器之间信息的传递。

物体在渲染管线中会进行光栅化，光栅化之后会产生许多个片元，有多少个片元，就会插值计算出多少易变变量，同时，渲染管线就会调用多少次片元着色器。

一般来说，渲染一个 3D 物体，片元着色器执行的次数会大大超过顶点着色器，因此，GPU 硬件配置中片元着色器的硬件数量往往多余顶点着色器硬件数量。

* const 限定符

用 const 限定符修饰的变量其值是不可以变的，也就是常量，又称为编译时常量。

编译时常量在声明的时候必须进行初始化，同时这些常量在着色器外部是完全不可见的。



### 函数声明与使用

GLSL 可以开发自定义的函数：

```
 <返回类型> 函数名称 ([<参数序列>]) {/*函数体*/}
```

在参数序列中，除了可以指定类型外，还可以指定用途，具体方法为用`参数用途修饰符`进行修饰，常用的参数用途修饰符如下：

*	`in` 修饰符，用其修饰的参数为输入参数，仅供函数接收外界传入的值。若某个参数没有明确给出用途修饰符，则等同于使用了 `in` 修饰符。
*	`out`修饰符，用其修饰的参数为输出参数，在函数体中对输出参数赋值可以将值传递到调用其的外界变量中。对于输出参数，要注意的是在调用时不可以使用字面常量。
*	`inout` 修饰符，用其修饰的参数为输入输出参数，具有输入与输出两种参数的功能。


### 顶点着色器中的内建变量

顶点着色器中的内建变量主要是输出变量，包括 gl_Position、gl_PointSize 等。

在顶点着色器中应该根据需要给这些内建变量赋值，以便由渲染管线中的图元装配与光栅化等后续固定功能阶段进行进一步的处理。

*	gl_Position

物体的顶点位置经过平移、旋转、缩放等数学变换后，生成新的顶点位置。新的顶点位置通过在顶点着色器中写入 gl_Position 传递到渲染管线的后续阶段继续处理。

gl_Position 的类型是 vec4 ，写入的顶点位置数据也必须与其类型一致。

*	gl_PointSize

顶点着色器中可以计算一个点的大小（单位为像素），并将其赋值给 gl_PointSize（标量 float 类型）以传递给渲染管线，默认为 1 。

gl_PointSize 的值一般只有在采用了点绘制方式后才有意义。




### 片元着色器中的内建变量

片元着色器中的内建变量分为输入变量以及输出变量两种。

#### 内建输入变量

内建输入变量主要有 `gl_FragCoord` 和 `gl_FrontFacing`，这两个都是只读的，由渲染管线中片元着色器之前的阶段生成。

*	gl_FragCoord

内建变量 gl_FragCoord （vec4 类型）中含有当前片元相对于窗口位置的坐标值 x、y、z 与 1/w 。

其中 x 与 y 分别为片元相对于窗口的二维坐标，如果窗口的大小为 800 * 480（单位为像素），那么 x 的取值范围为 0 ~ 800，y 的取值范围为 0 ~ 480，z 部分为该片元的深度值。

通过该内建变量可以实现与窗口位置相关的操作，例如，仅绘制窗口中指定区域内容等。

*	gl_FrontFacing

gl_FrontFacing 是一个布尔类型的内建变量，通过读取该内建变量的值可以判断正在处理的片元是否属于在光栅化阶段生成此片元的对应图元的正面。如果属于正面，gl_FrontFacing 的值为 true，反之为 false 。

#### 内建输出变量


内建输出变量主要有 `gl_FragColor` 和 `gl_FragData`，在片元着色器中根据具体情况需要给这两个内建变量写入值。


*	gl_FragColor

gl_FragColor（vec4 类型）内建变量用来由片元着色器写入计算完成的片元颜色值，此颜色值将送入渲染管线的后继阶段进行处理。

*	gl_FragData

gl_FragData 内建变量本身是一个 vec4 类型的数组，写入时要给出下标，如 gl_FragData$[0$]，通过其写入的信息将供渲染管线的后继过程使用。


在实际开发中，对上述两个内建输出变量选用其中一个就好了，不用两个都赋值，若执行了 discard 操作，则两个内建变量都不需要写入值了。


### 内置函数


OpenGL ES 着色器语言提供了很多内置函数，这些函数大都已经被重载，一般具有 4 种变体，分别用来接收和返回 `genType`、`genIType`、`genUType`和`genBType`类型的值。

四种变体如下：

|变体类型|说明|变体类型|说明|
|---|---|---|---|
|genType|float,vec2,vec3,vec4|genUType|uint,uvec2,uvec3,uvec4|
|genIType|int,ivec2,ivec3,ivec4|genBType|bool,bvec2,bvec4,bvec4|

从表中可以看到，genType、genIType、genUType、genBType 分别代表的是浮点型类型、整型类型、无符号整型类型和布尔型类型。


内置函数通常是以最优的方式来实现的，有部分函数甚至由硬件直接支持。

内置函数按照设计目的分为 3 个类别：

*	提供独特硬件功能的访问接口，如纹理采样系列函数
*	简单的数学函数，如 abs（求绝对值）、floor（向下取整） 等
*	一些复杂的函数，如三角函数等

### 角度转换和三角函数

角度转换与三角函数同时适用于顶点着色器与片元着色器，并且每个角度转换与三角函数都有 4 种重载变体。

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftsbhy79vmj20uw0maal9.jpg)

上述表中 genType 代表的数据类型有 float、vec2、vec3 以及 vec4 。其中 float 指的是浮点数标量，vec2、vec3 和 vec4 指的是浮点数向量。


### 指数函数

指数函数同样适用于顶点着色器和片段着色器。

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftsbj79nwxj20ux0h7dol.jpg)

上述表中 genType 代表的数据类型有 float、vec2、vec3 以及 vec4 。

### 常见函数

常见函数也可同时用于顶点着色器和片段着色器。

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftsbkmljcxj20v2037myd.jpg)

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftsblexf4zj20vy0hcn7w.jpg)

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftsbm3lm8qj20vw0cown4.jpg)

genType 、 genIType 、genUType 代表的数据类型参考上面提到的类型。

### 几何函数

几何函数适用于顶点着色器和片段着色器，主要用于对向量进行操作。

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftsbniw3l7j20w00kgwp0.jpg)
![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftsbnhc1e7j20vz03p0u3.jpg)


### 矩阵函数

矩阵函数目前只有一个，可以用于顶点着色器，也可以用于片段着色器。

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftsbp9m0bfj20w103775p.jpg)

### 向量关系函数

向量关系函数主要功能为将向量的各分量进行关系比较运算。

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftsbqaqunej20vu0ietht.jpg)


### 纹理采样函数

纹理采样函数主要用于根据指定的纹理坐标从采样器对应的纹理中进行采样，返回采样得到的颜色值。

大部分纹理采样函数即可以用于顶点着色器也可以用于片段着色器，但又个别的仅适用于片元着色器。

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftsbr5jq5tj20vu0f97fp.jpg)


## 小结

关于 OpenGL ES 2.0 的语法就先认识到这吧，主要还是在日后的开发中多去实践，在实践中更好地去运用。

## 参考

1. 《Android 3D 游戏开发技术宝典 OpenGL ES 2.0》

