---
title: "C++ 模板系列小结07-尾置返回类型"
date: 2021-02-20T21:04:42+08:00
tags: ["C++"]
categories: ["code"]
toc: true
 
draft: false
original: true
slug: "c++-template-7"
---
在使用模板时可以显示指定模板类型，尤其是针对有返回类型的模板，显示指定可以避免类型转换带来的困扰。

但有时候显示指定模板实参类型会给用户增添额外负担，而且不会带来什么好处。

比如如下代码，接受表示序列的一对迭代器和返回序列中的一个元素的引用：

```cpp
template<typename It>
??? &fcn(It beg,It end){
    return *beg;
}
```

我们并不知道返回结果的准确类型，但知道所需类型是所处理的序列的元素类型。

```cpp
vector<int> vi = {1,2,3,4,5};
auto &i =fcn(vi.begin,vi.end());
```

如上代码，知道函数应该返回 *beg，而且知道我们可以用 decltype(*beg) 来获取此表达式的类型。

但是，在编译器遇到函数的参数列表之前，beg 都是不存在的。为此，我们需要使用**尾置返回类型**。

<!--more-->

由于尾置返回出现在参数列表之后，它可以使用函数的参数：

```cpp
// 尾置返回允许在参数列表之后声明返回类型
template<typename It>
auto fcn(It beg,It end) -> decltype(*beg){
    return *beg;
}
```

在上面的例子中，fcn 的返回类型和解引用 beg 参数的结果类型相同。

解引用运算符返回一个左值，因此通过 decltype 推断的类型为 beg 表示的元素的类型的引用。

比如，如果对一个 string 序列调用 fcn，返回类型是 string& ，如果是 int 序列，返回类型是 int& 。

关于为何返回的类型是引用类型，可以回顾一下 decltype 的使用：

## decltype 的使用

当希望从表达式的类型推断出要定义的变量的类型，但是不想用该表达式的值初始化变量。为了满足这一要求，C++11 新标准引入了类型说明符 decltype ，它的作用是选择并返回操作数的数据类型。

在此过程中，编译器分析表达式并得到它的类型，却不实际计算表达式的值。


```cpp
int f(){
    return 1;
}

int main(){
    int x = 2;
    decltype(f()) sum = x;
    return 0;
}
```

如上代码所示，编译器并不实际调用函数 f ，而是使用当调用发生时 f 的返回值类型作为 sum 的类型。换句话说，编译器为 sum 指定的类型就是 f 被调用时返回的那个类型。

如果 decltype 使用的表达式是一个变量，则 decltype 返回该变量的类型（包括顶层 const 和引用在内）。

```cpp
const int ci = 0, &cj = ci;
decltype(ci) x = 0;         // x 的类型是 const int
decltype(cj) y = x;         // y 的类型是 const int & ,y 绑定到变量 x
decltype(cj) z;             // z 是一个引用，必须初始化
```


如果 decltype 返回的变量类型是一个引用，那么对应的变量就必须初始化才行。

另外，如果 decltype 使用的表达式不是一个变量，则 decltype 返回表达式结果对应的类型。

有的表达式将向 decltype 返回一个引用类型。一般来说当这种情况发生时，意味着该表达式的结果对象能作为一条赋值语句的左值。

```cpp
int i = 42,*p = &i, &r = i;
decltype(r + 0) b;      //正确：加法的结果是 int，因此 b 是一个未初始化的 int
decltype(*p) c;         //错误：c 是 int&,必须初始化
```

因为 r 是一个引用，因此 decltype(r) 必然是引用类型。如果想让结果是 r 所指的类型，可以把 r 作为表达式的一部分，如 r+0 ，显然这个表达式的结果将是一个具体指而非一个引用。

另外，如果一个表达式的内容是解引用操作，则 decltype 将得到引用类型。因为，解引用指针可以得到指针所指的对象，而且还能给这个对象赋值。因此，decltype(*p) 的结果就是 int& ,而非 int 。


另外，对于 decltype 所用的表达式来说，如果变量名加上了一对括号，则得到的类型与不加括号时会有不同。

如果 decltype 使用的是一个不加括号的变量，则得到的结果就是该变量的类型。如果给变量加上了一层或多层括号，编译器会把它当成一个表达式，而变量是一种可以作为赋值语句左值的特殊表达式，这是 decltype 就会得到引用类型。

```cpp
decltype((i)) d;  // 错误，d 是引用类型， int& ,必须要初始化
decltype(i) e;      //正确，e 是未初始化的 int 
```


## 尾置返回元素为值而非引用

在前面的例子中，返回的是引用类型，但是有时候希望返回一个元素的值，而非引用。

```cpp
template<typename It>
auto fcn(It beg,It end) -> decltype(*beg){
    return *beg;
}
```

但问题在于，无法知道传递的参数类型，唯一可以使用的操作是迭代器操作，而所有迭代器操作都不会生成元素，只能生成元素的引用。

为了获得元素类型，要使用标准库的`类型转换模板`。

这些模板定义在头文件 type_traits 中，主要是用于模板元编程设计的，这次主要用到 remove_reference 模板，它的源码如下：

```cpp
template <class _Tp> struct _LIBCPP_TEMPLATE_VIS remove_reference        {typedef _LIBCPP_NODEBUG_TYPE _Tp type;};
```

如同它的名字一样，就是移除引用，可以通过它来获得元素类型。remove_reference 模板有一个模板类型参数和一个名为 type 的类型成员。

> 如果用一个引用类型实例化 remove_reference，则 type 将表示被引用的类型。

例如，实例化 remove_reference<int&> ，则 type 成员将是 int 。类似，如果实例化 remove_reference<string&> ，则 type 成员将是 string 。

如果给定一个迭代器 beg: 

```cpp
remove_reference<decltype(*beg)>::type
```

将获得 beg 引用的元素的类型：decltype(*beg) 返回元素类型的引用类型，remove_reference::type 脱去引用，剩下元素类型本身。

组合使用 remove_reference、尾置返回以及 decltype ，就可以在函数中返回元素值的拷贝。

```cpp
template<typename It>
auto fcn2(It beg,It end) -> typename remove_reference<decltype(*beg)>::type{
    return *beg; // 返回序列中一个元素的拷贝
}
```

要注意的是，type 是一个类的成员，而该类依赖于一个模板参数。因此，要在返回类型的声明中使用 typename 来告知编译器，type 表示一个类型，关于这个知识点在之前的文章中提到过了，参见：[C++ 模板系列小结03-在模板中指定变量类型](https://glumes.com/post/c++/c++-template-3/) 。

## 小结

尾置返回类型主要还是 decltype 的使用了，而 decltype 要注意的是返回的类型是元素的类型，还是引用类型了，在不同场景下注意区分。



