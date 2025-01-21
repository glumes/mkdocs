---
title: "C++ 模板系列小结03-在模板中指定变量类型"
date: 2021-02-09T10:19:51+08:00
subtitle: ""
tags: ["C++"]
categories: ["code"]
toc: true
 
draft: false
original: true
slug: "c++-template-3"
header_img: "/img/banner.jpg"
---

在之前的代码示例中，频繁用到 `typename` 关键字。

它的作用就是声明模板参数是类型参数（对于非类型参数，之前的文章也有提到了），也可以用 `class` 关键字来代替，但为了避免歧义，大多还是使用 `typename` 了。

除此之外，在模板的定义也可以使用 `typename` 关键字，用来指定变量的类型。

<!--more-->

举个例子：

```cpp
class Foo{
public:
    typedef int num;
    static int order;
};

int main(){
    Foo::num a = 10;
    Foo::order = 10;
    return 0;
}

```

以上定义了一个类 Foo ，并且在类中定义了一个类型别名 `Foo::num` ，后续我们可以使用该类型别名。

同时也定义了一个类变量 `order` ，后续也可以直接使用该类变量。

类变量和类中声明的别名使用方式有相似之处，都是通过 `XX::XX` 的形式使用，但一个是变量，一个是类型名称，这是两者的根本差别。

假设现在定义了如下的 MyClass 类模板：

```cpp
template<typename T>
class MyClass {
private:
    T::num * a;
};
```

由于类变量和类中别名的相似之处，此时编译器将不知道 T::num 到底是变量还是类型，如果是变量的话，加上后面的 * 号会被当成是一个乘法操作。

这时就需要在类模板定义中使用 `typename` 关键字指定 T::num 是一个类型而不是变量。

如下所示:

```cpp
template<typename T>
class MyClass {
private:
   typename T::num * a;
};
```

通常而言，当某个依赖于模板参数的名称是一个类型时，就应该使用 `typename` 。

一个典型的应用就是在模板代码中访问 STL 容器的迭代器：

```cpp
template<typename T>
void printcoll(T const &coll) {
    typename T::const_iterator pos;
    typename T::const_iterator end(coll.end());

    for (pos = coll.begin(); pos != end; pos++) {
        std::cout << *pos << std::endl;
    }
}
```

