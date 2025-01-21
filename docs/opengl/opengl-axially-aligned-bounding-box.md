---
title: "《OpenGL ES 3.x 游戏开发》碰撞检测之 AABB 包围盒"
date: 2018-07-26T12:47:49+08:00
subtitle: "学习笔记内容摘录"
draft: false
slug: "opengl-axially-aligned-bounding-box"
categories: ["OpenGL"]
tags: ["OpenGL"]
toc: true
 
---

在 OpenGL 的世界模型中，同时绘制了多个物体，那么怎么去检测物体之间是否触碰了，不同于在平面之间的触碰，OpenGL 是在三维世界里面的触碰，接下来就继续深入理解 OpenGL 中的碰撞检测相关知识~~~


<!--more-->

## 碰撞检测实现原理

进行碰撞检测时最直观和最精确的方式就是采用组成物体的三角形来进行，将待检测的两个物体的三角形组中的三角形两两进行相交性检测，若有任意一对三角形相交则物体构成碰撞，否则物体不构成碰撞。

这样的思路很直观，但是计算量大，在一般设备上是难以实施的。


通过一种简化的思路来间接实现碰撞检测，采用 `AABB （Axially Aligned Bounding Box）`包围盒的方式实现。

AABB 包围盒就是采用一个长方体将物体包裹起来，进行两个物体的相交性检测时仅检测物体对应包围盒（包裹物体的长方体）的相交性。

另外，AABB 包围盒有一个重要特性，那就是包围盒对应的长方体每一个面都是与某个坐标轴平面平行的，因此，AABB 包围盒又称了 `轴对齐包围盒` 。

对于不同物体包围盒直接示例，如下图：

![不同物体的 AABB 包围盒](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftn0tnab4pj209p03yweu.jpg)




有了上述约定后，确定 AABB 包围盒就简单多了，仅需要记录 6 个值即可，这 6 个值分别代表包围盒在每个坐标轴上的最小值与最大值，即 $x\_{min}$、$x\_{max}$、$y\_{min}$、$y\_{max}$、$z\_{min}$、$z\_{max}$。

也就是说，实际物体上所有的点都必须满足以下条件：

$$ x\_{min}  \le x  \le x\_{max} $$
$$ y\_{min}  \le y  \le y\_{max} $$
$$ z\_{min}  \le z  \le z\_{max} $$


另外，可以将表示 AABB 包围盒的 6 个参数分为以下两组：

$$P\_{min} = [x\_{min} , y\_{min} , z\_{min}]$$
$$P\_{max} = [x\_{max} , y\_{max} , z\_{max}]$$

其中，$P\_{min}$ 是 3 个轴坐标最小值的集合，$P_{max} $ 是 3 个轴坐标最大值的集合。

知道了表示 AABB 包围盒的 6 个参数后就可以非常方便地求得 AABB 包围盒的几何中心，公式如下：

$$c=(P\_{min} + P\_{max})  /  2$$

## AABB 包围盒计算


了解了 AABB 包围盒的原理之后，求 AABB 包围盒 6 个参数的方法就简单多了，只需要对物体中的所有顶点坐标进行扫描，求出各个轴分量的最大值与最小值即可。


首先引入一个辅助类 `Vector3f`，其对象可以记录 3 个浮点数组成的三元组，用来表示物体的位置、速度等。

```java
public class Vector3f {
    float x;//三维变量中的x值
    float y;//三维变量中的y值
    float z;//三维变量中的z值
    public Vector3f(float x,float y,float z)
    {
        this.x=x;
        this.y=y;
        this.z=z;
    }
    public void add(Vector3f temp)
    {
        this.x+=temp.x;
        this.y+=temp.y;
        this.z+=temp.z;
    }
}
```

有了此辅助类之后，物体的位置、速度等就可以直接用此类的对象表示了。

接下来就是计算物体的 AABB 包围盒，定义一个对象 AABBBox 来表示包围盒。


```java
public class AABBBox {
    float minX;//x轴最小位置
    float maxX;//x轴最大位置
    float minY;//y轴最小位置
    float maxY;//y轴最大位置
    float minZ;//z轴最小位置
    float maxZ;//z轴最大位置

    public AABBBox(float[] vertices)
    {
        init();
        findMinAndMax(vertices);
    }

    public AABBBox(float minX,float maxX,float minY,float maxY,float minZ,float maxZ)
    {
        this.minX=minX;
        this.maxX=maxX;
        this.minY=minY;
        this.maxY=maxY;
        this.minZ=minZ;
        this.maxZ=maxZ;
    }
    //初始化包围盒的最小以及最大顶点坐标
    public void init()
    {
        minX=Float.POSITIVE_INFINITY;
        maxX=Float.NEGATIVE_INFINITY;
        minY=Float.POSITIVE_INFINITY;
        maxY=Float.NEGATIVE_INFINITY;
        minZ=Float.POSITIVE_INFINITY;
        maxZ=Float.NEGATIVE_INFINITY;
    }
    //获取包围盒的实际最小以及最大顶点坐标
    public void findMinAndMax(float[] vertices)
    {
        for(int i=0;i<vertices.length/3;i++)
        {
            //判断X轴的最小和最大位置
            if(vertices[i*3]<minX)
            {
                minX=vertices[i*3];
            }
            if(vertices[i*3]>maxX)
            {
                maxX=vertices[i*3];
            }
            //判断Y轴的最小和最大位置
            if(vertices[i*3+1]<minY)
            {
                minY=vertices[i*3+1];
            }
            if(vertices[i*3+1]>maxY)
            {
                maxY=vertices[i*3+1];
            }
            //判断Z轴的最小和最大位置
            if(vertices[i*3+2]<minZ)
            {
                minZ=vertices[i*3+2];
            }
            if(vertices[i*3+2]>maxZ)
            {
                maxZ=vertices[i*3+2];
            }
        }
    }
    //获得物体平移后的AABB包围盒
    public AABBBox getCurrAABBBox(Vector3f currPosition)
    {
        AABBBox result=new AABBBox
                (
                        this.minX+currPosition.x,
                        this.maxX+currPosition.x,
                        this.minY+currPosition.y,
                        this.maxY+currPosition.y,
                        this.minZ+currPosition.z,
                        this.maxZ+currPosition.z
                );
        return result;
    }
}
```

在 AABBBox 的构造函数中需要传入物体的顶点序列，在这些序列中分别找出 x 、y、z 坐标的最小值和最大值，也可以直接传入包围盒的 6 个顶点参数。

如果在程序运行过程中，物体发生了移动，那就需要根据移动后的位置计算产生新包围盒对象，`getCurrAABBBox`方法需要传递物体移动后位置的 3 个坐标分量，用于与原始包围盒的 6 个 参数进行运算，产生移动后包围盒的 6 个参数。


## AABB 包围盒的碰撞检测

在上面求得了物体的包围盒，求包围盒的目的就是为了简化物体运动过程中的碰撞检测，接下来就是介绍 AABB 包围盒碰撞检测的策略，原理如下图所示：


![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftn2ce177mj20ie04p0tu.jpg)


由于任何一个 AABB 包围盒的各个面都平行于坐标平面，因此判断两个 AABB 包围盒是否发生碰撞仅需要分别判断 3 个轴方向的交叠部分大小是否大于设定的阈值，若大于则发生了碰撞，否则没有发生碰撞。


具体碰撞检测代码如下：

```java
    public boolean check(RigidBody ra, RigidBody rb)//true为撞上
    {
        float[] over = calOverTotal
                (
	                // 两个物体的 AABB 包围盒
                        ra.collObject.getCurrAABBBox(ra.currLocation),
                        rb.collObject.getCurrAABBBox(rb.currLocation)
                );
        // 三个方向的交叠值与设定的阈值进行比较
        return over[0] > V_UNIT && over[1] > V_UNIT && over[2] > V_UNIT;
    }
	
	// 传入两个物体的 AABB 包围盒
    public float[] calOverTotal(AABBBox a, AABBBox b) {
        float xOver = calOverOne(a.maxX, a.minX, b.maxX, b.minX);
        float yOver = calOverOne(a.maxY, a.minY, b.maxY, b.minY);
        float zOver = calOverOne(a.maxZ, a.minZ, b.maxZ, b.minZ);
        return new float[]{xOver, yOver, zOver};
    }
	// 计算每个轴方向的交叠值
    public float calOverOne(float amax, float amin, float bmax, float bmin) {
        float minMax = 0;
        float maxMin = 0;
        if (amax < bmax)//a物体在b物体左侧
        {
            minMax = amax;
            maxMin = bmin;
        } else //a物体在b物体右侧
        {
            minMax = bmax;
            maxMin = amin;
        }

        if (minMax > maxMin) {
            return minMax - maxMin;
        } else {
            return 0;
        }
    }
```

在 calOverTotal 方法里面要传入两个物体的 AABB 包围盒。

然后在 calOverOne 方法里面分别计算两个包围盒 3 个轴的交叠值，思路就是先比较两个AABB 包围盒在对应轴的最大分量，此比较是为了得出两个物体的方位，哪个物体在此轴的正方向那一侧。然后用在轴正方向的那一侧物体的坐标最小值减去在轴负方向那一侧的物体的最大值，如果为负数，则说明没有发生交叠，如果其值为正数，则说明两个物体发生了交叠，交叠值和设定的阈值进行比较。


## 实践

具体的实践效果如下：

![碰撞检测示例](https://res.cloudinary.com/glumes-com/image/upload/v1532576353/code/collision_demo.gif)

在效果图中，有个两个静止不同的物体，一个来回运动碰撞的物体。


在开发上述效果中，会定义一个 RigidBody 刚体类，它由两个重要的成员，就是我们的绘制物体和它的 AABB 包围盒，如下图所示：

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftn452gxq7j20fa03rjs2.jpg)

渲染者指的就是用来渲染的物体，碰撞者指的就是物体的 AABB 包围盒，它们共同组成了刚体 RigidBody 。

RigidBody 代码如下：

```java
public class RigidBody {
    CollisionObject renderObject;//渲染者
    AABBBox collObject;//碰撞者
    boolean isStatic;//是否静止的标志位
    Vector3f currLocation;//位置三维变量
    Vector3f currV;//速度三维变量
    final float V_UNIT = 0.02f;//阈值

    public RigidBody(CollisionObject renderObject, boolean isStatic, Vector3f currLocation, Vector3f currV) {
        this.renderObject = renderObject;
        collObject = new AABBBox(renderObject.vertices);
        this.isStatic = isStatic;
        this.currLocation = currLocation;
        this.currV = currV;
    }

    public void drawSelf() {
        MatrixState.pushMatrix();//保护现场
        MatrixState.translate(currLocation.x, currLocation.y, currLocation.z);
        renderObject.drawSelf();//绘制物体
        MatrixState.popMatrix();//恢复现场
    }

    public void go(ArrayList<RigidBody> al) {
        if (isStatic) return;
        currLocation.add(currV);
        for (int i = 0; i < al.size(); i++) {

            RigidBody rb = al.get(i);
            if (rb != this) {
                if (check(this, rb))//检验碰撞
                {
                    this.currV.x = -this.currV.x;//哪个方向的有速度，该方向上的速度置反
                }
            }
        }
    }
// 省略碰撞检测的代码，在上面已经有了。
}
```

在 RigidBody 的构造函数中，需要传入用来渲染的物体对象，以及是否是静止的标志位，还有表示该物体的位置和移动的向量，向量类型都是之前提到的 `Vector3f`。

当传入要绘制的对象时，会根据它的顶点计算对应的包围盒，当物体移动后，也会根据新的坐标位置重新计算包围盒。

物体发生移动时，只需要将物体的位置向量加上物体的速度向量，然后调用 `Matrix.translateM` 方法就可以改变物体位置了。

```java
    Vector3f currLocation;//位置三维变量
    Vector3f currV;//速度三维变量
    // 位置向量加上速度向量，就改变了物体的位置
    currLocation.add(currV);
    // translate 方法让改变生效
    MatrixState.translate(currLocation.x, currLocation.y, currLocation.z);
```


如果碰撞发生了，并且物体要反向移动，那么就改变速度向量的方向就好了。

```java
                if (check(this, rb))//检验碰撞，如果发生碰撞了，改变方向
                {
                    this.currV.x = -this.currV.x;//哪个方向的有速度，该方向上的速度置反
                }
```

具体绘制时，在 GLSurfaceView.Renderer 方法里面就采用刚体 RigidBody 类来绘制就好了。

```java
    var aList = ArrayList<RigidBody>() // 定义刚体集合
	    // onSurfaceCreated 方法里面添加要绘制的物体，
        override fun onSurfaceCreated(gl: GL10?, config: EGLConfig?) {
            aList.add(RigidBody(ch, true, Vector3f(-13f, 0f, 0f), Vector3f(0f, 0f, 0f)))
            aList.add(RigidBody(ch, true, Vector3f(13f, 0f, 0f), Vector3f(0f, 0f, 0f)))
            aList.add(RigidBody(ch, false, Vector3f(0f, 0f, 0f), Vector3f(0.1f, 0f, 0f)))
			// 开启一个线程来改变刚体的移动
            lgt = LovoGoThread(aList)
            lgt.start()
        }
        // 绘制刚体
        override fun onDrawFrame(gl: GL10?) {
            GLES30.glClear(GLES30.GL_DEPTH_BUFFER_BIT or GLES30.GL_COLOR_BUFFER_BIT)
            for (it in 0 until aList.size) {
                aList[it].drawSelf()
            }
            pm.drawSelf()
        }
```

在物体移动时，开启了一个线程去改变物体的坐标。

```java
public class LovoGoThread extends Thread {
    ArrayList<RigidBody> al;//控制列表
    boolean flag = true;//线程控制标志位
    public LovoGoThread(ArrayList<RigidBody> al) {
        this.al = al;
    }
    public void run() {
        while (flag) {
            int size = al.size();
            for (int i = 0; i < size; i++) {
                al.get(i).go(al);
            }
            try {
                sleep(20);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }
}
```


具体的代码，可以参考我的 Github 项目：

> [https://github.com/glumes/AndroidOpenGLTutorial](https://github.com/glumes/AndroidOpenGLTutorial)

## 注意点

AABB 包围盒的碰撞检测其实还是一种间接检测方案，所以肯定还是会有误差的。

AABB 包围盒对于本身横平竖直的物体在平行于坐标轴摆放的情况下下，计算碰撞检测的误差很小，但对于不规则形状的物体或本身横平竖直的物体在随意倾斜摆放时，计算碰撞检测的误差就会比较大，如下图所示：

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1ftn5uds87ej20h604vt96.jpg)

从图中可以看到，对于同一物体在不同姿态下的 AABB 包围盒以及不同形状的物体的 AABB 包围盒，其物体本身区域占 AABB 包围盒区域的比例是大不相同的。

物体本身区域所占比例越大，则碰撞检测的误差越小，因此在实际开发中应该尽量提高物体本身区域所在的比例，以降低计算误差。


## 参考

1. 《OpenGL ES 3.x 游戏开发》



