---
title: "OpenGL ES 3.0 着色器语言 GLSL 学习 Mark "
date: 2018-09-09T15:20:42+08:00
subtitle: ""
tags: ["OpenGL","GLSL"]
categories: ["OpenGL"]
toc: true
 
draft: false
original: true
addwechat: true
slug: "opengl-glsl-3-mark"
---


在之前的[文章](https://glumes.com/post/opengl/opengl-glsl-2-mark/)中， 主要介绍了 OpenGL ES 2.0 的 GLSL 语法，在 OpenGL ES 3.0 中语法又有了一些变化。

本文的内容来自于《OpenGL ES 3.x 游戏开发 上卷》。

<!--more-->

## 数据类型


### 标量

标量也被称为 "无向量"，其值只有大小，并不具有方向。

OpenGL ES 着色器语言支持的标量类型如下：

*	 布尔型    bool
*	 有符号整型和无符号整型    int/uint
*	  浮点型   float

> 相对于 OpenGL ES 2.0 ，多了无符号整型 uint 。

### 向量

在 GLSL 中，向量可以看做是同样类型的标量组成的，其基本类型也分为 bool、int 、uint 及 float 4 种。

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftsvggbzsej20ou06njsm.jpg)

### 矩阵

OpenGL ES 3.0 支持的矩阵类型比 OpenGL ES 2.0 多了一些。

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftsvi91fqnj20ot06l0tq.jpg)

在 OpenGL 中，矩阵是按列的顺序组织的，也就是一个矩阵可以看作由几个列向量组成。

### 采样器

采样器是专门进行纹理采样的相关操作。一般情况下，一个采样器变量代表一幅或一套纹理贴图。

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftsvkyfrhsj20ov08j412.jpg)

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

*	在声明数组并初始化的同时，可以不指定数组的大小。


```glsl
float x[] = float[2](1.0,2.0);
float y[] = float[](1.0,2.0,3.0);
```

### 空类型

空类型使用 `void` 表示，仅用来声明不返回任何值的函数，比如着色器中的 main 函数就是一个返回值为 `void` 类型的函数。


## 存储限定符


![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftsvoph5x8j20ow060q49.jpg)

限定符的使用应该放在变量前面，且使用 in、uniform 以及 out 限定符修饰的变量必须为全局变量。

着色器语言中没有默认限定符的概念，因此如果有需要，必须为全局变量明确指定需要的限定符。

### in / centroid in 限定符

in / centroid in 限定符修饰的全局变量又称为输入变量。

对于顶点着色器的输入变量只能用 in 限定符来修饰，不能使用 centroid in 限定符和 imterpolation 限定符来修饰。

in 修饰符有点类似于 OpenGL ES 2.0 中的 attribute 限定符。

对于片元着色器可以使用 in 或 centroid in 限定符来修饰全局变量。

### out / centroid out 限定符

out / centroid out 限定符修饰的全局变量又称为输出变量。使用情况分为如下两种情况：

*	顶点着色器的输出变量

顶点着色器中可以使用 out 或者 centroid out 限定符修饰全局变量，其变量用于向渲染管线后继阶段传递当前顶点的数据。

*	片元着色器的输出变量

在片元着色器中只能使用 out 限定符来修饰全局变量，而不能使用 centroid out 限定符。

片元着色器中的 out 变量一般指的是由片元着色器写入计算完成片元颜色值的变量。



> 对于顶点着色器而言，一般是既声明 out 变量，又对 out 变量进行赋值用以传递给片元着色器。而片元着色器中声明 in 变量用于接收顶点着色器传过来的值即可，是不可以对 in 变量赋值的。另外，在 OpenGL ES 3.0 中片元着色器内的内建输出变量 gl_FragColor 不存在了，需要自己声明 out (vec4) 变量，用声明的 out 变量替代 gl_FragColor 内建变量。


### uniform 限定符

uniform 为一致变量限定符，一致变量指的是对于同一组顶点组成的单个 3D 物体中所有顶点都相同的量。

### const 限定符

用 const 限定符修饰的变量其值是不可以变的，也就是常量，又称为编译时常量。

### 插值限定符

插值限定符，其主要用于控制顶点着色器传递到片元着色器数据的插值方式。

插值限定符包含如下两种：

*	smooth
*	flat

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftsw70uh21j20ou02vdgc.jpg)


> 若使用插值限定符，则该限定符应该在 in、centroid in 、out 或者 centroid out 之前使用，且只能用来修饰顶点着色器的 out 变量与片元着色器中对应的 in 变量。当未使用任何插值限定符时，默认的插值方式为 smooth 。

*	smooth 限定符

如果顶点着色器中 out 变量之前含有 smooth 限定符或者不含有任何限定符，则传递到后继片元着色器对应的 in 变量的值，是在光栅化阶段由管线根据片元所属图元各个顶点对应的顶点着色器对此 out 变量的赋值情况，及片元与各顶点的位置关系插值产生。

* flat 限定符

如果顶点着色器中 out 变量之前含有 flat 限定符，则传递到后继片元着色器中对应的 in 变量的值不是在光栅化阶段插值产生的，一般是由图元的最后一个顶点对应的顶点着色器对此 out 变量所赋的值决定的。此时，图元中每个片元的此项值均相同。


若顶点着色器中的输出变量的类型为整型标量或者整型向量，则变量必须使用 flat 限定符修饰。与之对应，若片元着色器中的输入变量的类型为整型标量或整型向量，变量必须用 flat 限定符修饰。


> 无论顶点着色器中的 out 全局变量被哪种插值限定符修饰，后继片元着色器中必须含有与之对应的修饰符修饰的 in 全局变量。



## 一致块

多个一致变量的声明可以通过类似结构体形式的接口块实现，该形式的接口块又称为一致块（uniform block）。

一致块的数据是通过缓冲对象送入渲染管线的，以一致块的形式批量传送数据比单个传送效率高，其基本语法为：

```glsl
[<layout 限定符>] unifrom 一致块名称 {<成员变量列表>} [<实例名>]
```

声明一致块时可能包含 5 个组成部分，分别是：

*	layout 限定符
*	uniform 修饰符
*	一致块名称
*	成员变量列表
*	实例名


示例如下：

```glsl
uniform Transform{
	float radius;
	mat4 modelViewMatrix;
	uniform mat3 normalMatrix;
} block_Transform;
```

创建一致块时，可以声明实例名，也可以不声明实例名。

*	未声明实例名

如果在创建一致块时未声明实例名，则一致块的成员变量与在块外一样，其作用域是全局的，既可以通过一致块的成员变量名称访问对应变量，也可以通过 `<一致块名称>.<成员变量名>` 的形式访问一致块的成员变量。

* 声明实例名

如果在创建一致块时声明了实例名，则一致块内成员变量的作用域从声明开始到一致块结束，都通过`<一致块名称>.<成员变量名>` 访问成员变量，而着色器语言中需要通过`<实例名>.<成员变量名>`访问一致块的成员变量。


## layout 限定符

layout 限定符从 OpenGL ES 3.0 开始出现的，其主要用于设置变量的存储索引（即引用）值，声明有几种不同的形式。

*	作为接口块定义的一部分或者接口块的成员
*	可以仅仅修饰 uniform，用于建立其他一致变量声明的参照

```glsl
<layout 限定符> uniform
```


*	用于修饰被接口限定符修饰的单独变量


```glsl
<layout 限定符> <接口限定符> <变量声明>
```


接口限定符有 in、out、uniform 三种选择。

着色器中的 layout 限定符必须在存储限定符之前使用，且 layout 限定符修饰的变量或接口块的作用域必须是全局的。

*	layout 输入限定符

顶点着色器允许 layout 输入限定符修饰输入变量的声明。

```glsl
layout (location = 0) in vec3 aPosition;
layout (location = 1) in vec4 aColor;
```

在片元着色器中不允许有 layout 输入限定符。


*	layout 输出限定符

片元着色器中，layout 限定符通过 location 值将输出变量和指定编号的绘制缓冲绑定起来。每 一个输出变量的索引(引用)值都会对应到一个相应编号的绘制缓冲，而这个输出变量的值将写 入相应缓冲。


> layout 限定符的 location 值是有范围的，其范围为 $[0, MAX_DRAW_BUFFERS-1$]。 不同手持设备的范围有可能不同，最基本的范围是 $[0,3$] 。


```glsl
layout (location = 0) out vec4 fragColor;
layout (location = 1) out vec4 colors[2];
```


在顶点着色器中不允许有 layout 输出限定符。


> 顶点着色器不允许有 layout 输出限定符。如果在片元着色器中只有一个输出变 量，则不需要用 layout 修饰符说明其对应绘制缓冲，在这种情况下，默认值为 0。如 果片元着色器中有多个输出变量，则不允许重复使用相同的 location 值。



## 内建变量

### 顶点着色器中的内建变量

#### 内建输入变量

顶点着色器中的内建输入变量主要有 gl_VertexID 以及 gl_InstanceID，这两个变量分别为顶点整数索引和实例 ID，都只在顶点着色器中使用。



#### 内建输出变量

顶点着色器中的内建输出变量主要有 gl_Position 和 gl_PointSize 。


### 片元着色器中的内建变量

#### 内建输出变量

内建输出变量指 gl_FragDepth，其值为片元深度值，从 OpenGL ES 3.0 开始，可以对其赋值，然后送进深度缓冲区内参与后续计算。

#### 内建输入变量

内建输入变量包括 gl_FragCoord、gl_FrontFacing 和 gl_PointCoord 。



## 用 invariant 修饰符避免值变问题

值变问题是指在同样的着色器程序多次运行时，同一个表达式在同样输入值的情况下多次运行，结果不精确一致的现象，在大部分情况下，这并不影响最终效果的正确性。

如果在某些特定情况下需要避免值变问题，可以用 invariant 修饰符来修饰变量。

采用 invariant 修饰符修饰变量主要有如下两种方式：

*	在声明变量时加上 invariant 修饰符

```glsl
invariant out vec3 color;
```

*	对已经声明的变量补充使用 invariant 修饰符进行修饰

```glsl
out vec3 color;
invariant color;
```

若有多个已经声明的变量需要用 invariant 修饰符补充修饰，则可以在 invariant 修饰符后把这些变量名用逗号隔开，一次完成。另外，用 invariant 修饰符补充修饰变量必须在变量第一次被使用前完成。

如果希望所有的输出变量都是 invariant 的，用如下语句完成：

```glsl
#pragma STDGL invariant(all)
```

上述代码只能添加在顶点着色器程序的最前面，不能用在片段着色器中。

不是所有变量都可以用 invariant 修饰，只有如下几种才可以：

*	顶点着色器的内建输出变量，如 gl_Position 。
*	顶点着色器中声明的以 out 修饰符修饰的变量。
*	片元着色器中内建的输出变量。
*	片元着色器中声明的以 out 修饰符修饰的变量。

invariant 使用时要注意放在其他的修饰符之前，同时，invariant 修饰符只能用来修饰全局变量。

## 函数
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

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftpatv2l1gj20x505itag.jpg)

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftpauq1lk3j20qu0m8jxq.jpg)


上述表中 genType 代表的数据类型有 float、vec2、vec3 以及 vec4 。其中 float 指的是浮点数标量，vec2、vec3 和 vec4 指的是浮点数向量。

### 指数函数

指数函数同样适用于顶点着色器和片段着色器。

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftpayhnwydj20r003djs4.jpg)

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftpays9rloj20qz0c4adp.jpg)

上述表中 genType 代表的数据类型有 float、vec2、vec3 以及 vec4 。

### 常见函数

常见函数也可同时用于顶点着色器和片段着色器。

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftpb0u5pf5j20r00i6tf2.jpg)

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftpb13iad9j20qx0nywno.jpg)


genType 、 genIType 、genUType 代表的数据类型参考上面提到的类型。



### 几何函数

几何函数适用于顶点着色器和片段着色器，主要用于对向量进行操作。

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftpbecmyi6j20r20ijgqq.jpg)

### 矩阵函数

矩阵函数适用于顶点着色器和片段着色器，主要包括生成矩阵、矩阵的转置、求矩阵的行列式以及求逆矩阵等有关矩阵的操作。

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftpbg27jyoj20r00e0n1f.jpg)

### 向量关系函数

向量关系函数主要功能为将向量的各分量进行关系比较运算。

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftpbhepfflj20qw0k8jwh.jpg)

### 纹理采样函数

纹理采样函数主要用于根据指定的纹理坐标从采样器对应的纹理中进行采样，返回采样得到的颜色值。

大部分纹理采样函数即可以用于顶点着色器也可以用于片段着色器，但又个别的仅适用于片元着色器。

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftpbm2irjuj20qw0dfte3.jpg)
![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftpbm1wlrfj20qz09hjvc.jpg)
![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftpbm2e3t9j20qv0kdqbe.jpg)
![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftpbm1xhgxj20qz075who.jpg)
![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftpbm27f72j20qx0ildn4.jpg)
![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftpbpjnyahj20r006fwgo.jpg)

### 微分函数

微分函数仅能用于片段着色器，是从 OpenGL ES 3.0 开始正式支持的。

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftpbqfuxhfj20qx04dq43.jpg)

### 浮点数的打包与解包函数

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftpbrx38zwj20r00e3n2w.jpg)
![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftpbrwvqx7j20r204rabo.jpg)



## 参考

1. 《OpenGL ES 3.x 游戏开发 上卷》