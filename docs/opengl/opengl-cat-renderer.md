---
title: "代码吸猫 | 用 OpenGL 图像渲染的养猫计划"
date: 2021-11-09T09:52:01+08:00
subtitle: ""
categories: ["OpenGL"]
slug: "opengl-cat-renderer"
tags: ["OpenGL"]
toc: true
 
draft: false
original: true
---

在掘金上看到最近的新活动 "代码吸猫"，技术类文章只要和猫有关就行。

<!--more-->

对于没有养猫的程序员，这不是为难人嘛。

不过没关系，用 OpenGL 图像渲染给自己造一只猫吧！！！


## 模型构造

首先需要构造出猫的模型，有能力的话可以直接在三维软件里面造一个。

或者像我一样直接下载免费的猫模型，然后把它导入 Blender 3D 软件中。

![](https://image.glumes.com/blog_image/blender_cat_model.png)

在 Blender 中可以预览猫模型，或者对它做一下调整，最后在把这个模型导出。




## 模型加载

导出的  obj 文件里面就记录了模型的顶点信息，接下来就要用 OpenGL 将它绘制出来。

这里要用到 assimp 开源库，它支持多种模型文件的解析操作，通过它将模型解析成一个个 Mesh 。

Mesh 的定义如下：

```cpp
class Mesh {
public:
  /*  Mesh Data  */
  vector<Vertex> vertices;
  vector<unsigned int> indices;
  vector<Texture> textures;
  unsigned int VAO;
  // 省略部分代码
}
```

Mesh 相当于绘制模型上的一个个网格或者说面片，它包含了该网格的顶点、纹理信息和绘制索引。

而模型 Model 就是由这一系列网格 Mesh 组成的。

![](https://image.glumes.com/blog_image/cat-model-mesh-render.png)


如上图所示，猫模型是由一个个小矩阵组成的，小矩阵就可以理解成 mesh 网格了。


Model 的定义如下：

```cpp
class Model
{
public:
  /*  Model Data */
  vector<Texture> textures_loaded;    
  vector<Mesh> meshes;
  // 省略部分代码  
}  
```

在实际绘制的时候，也是由一个一个 Mesh 最终绘制成的。

```cpp
    // draws the model, and thus all its meshes
 void Draw(Shader shader)
 {
   for(unsigned int i = 0; i < meshes.size(); i++)
     meshes[i].Draw(shader);
 }
```

从图中也可以看到，猫模型的网格数量是很多的，导致加载的时候会很很慢了，加载方法如下：

```cpp
  void loadModel(string const &path)
  {
    // 使用 assimp 库进行加载
    const aiScene* scene = importer.ReadFile(path, aiProcess_Triangulate | aiProcess_FlipUVs | aiProcess_CalcTangentSpace);
    // 检查是否有错
    if(!scene || scene->mFlags & AI_SCENE_FLAGS_INCOMPLETE || !scene->mRootNode) // if is Not Zero
    {
      cout << "ERROR::ASSIMP:: " << importer.GetErrorString() << endl;
      return;
    }
    // 获取模型所在文件夹
    directory = path.substr(0, path.find_last_of('/'));

    // 从根节点一个一个节点开始处理
    processNode(scene->mRootNode, scene);
  }
```

使用 assimp 处理后会得到一个根节点，然后顺着根节点一个一个往下处理就好了。

```cpp
  void processNode(aiNode *node, const aiScene *scene)
  {
    for(unsigned int i = 0; i < node->mNumMeshes; i++)
    {
      aiMesh* mesh = scene->mMeshes[node->mMeshes[i]];
      // 处理得到的 aiMesh 并组装成定义好的 Mesh 数据结构 
      meshes.push_back(processMesh(mesh, scene));
    }
    // 处理子节点
    for(unsigned int i = 0; i < node->mNumChildren; i++)
    {
      processNode(node->mChildren[i], scene);
    }
  }
```

可以看到处理过程大量的 for 循环操作，所以后续才会针对模型文件的优化，加快其加载速度。

## 模型渲染


得到了最终的 Model 之后，就可以对它做渲染显示了。

```cpp
    // model 矩阵调整模型显示位置和方向
    glm::mat4 model = glm::mat4(1.0f);
    model = glm::translate(model, glm::vec3(tranx_x, tranx_y, tranx_z));
    model = glm::rotate(model,glm::radians(90.0f),glm::vec3(0.0,0.0,1.0));
    model = glm::scale(model, glm::vec3(0.5f, 0.5f, 0.5f));    
    shader.setMatrix4fv("model", glm::value_ptr(model));
    ourModel.Draw(shader);
```

由于模型自身就带了一个位置和方向，显示的时候不一定是我们想要的观察方位，所以还是要调整一个模型矩阵。

最后渲染就可以看到 猫模型 效果啦。


![](https://image.glumes.com/blog_image/opengl-cat-demo-1.png)

![](https://image.glumes.com/blog_image/opengl-cat-demo-2.png)

![](https://image.glumes.com/blog_image/opengl-cat-demo-3.png)


## 小结

为了便于观察，可以处理一下键盘或者鼠标事件，修改模型矩阵的值，从不同角度撸猫。

目前的猫模型还只是静态的，调整的话也只能用键盘调整，而且还只是改了 移动、缩放、旋转这些属性，猫本身是没有动的。

想要猫自身能动的话，还需要模型里面有对应的骨骼动画才可以了，等后面有了这样的模型，再继续迭代。

关注微信公众号 音视频开发进阶，看更多撸猫后续内容...

