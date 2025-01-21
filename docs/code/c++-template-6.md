---
title: "C++ 模板系列小结06-可变参数模板特性"
date: 2021-02-20T21:03:04+08:00
subtitle: ""
tags: ["C++"]
categories: ["code"]
toc: true
 
draft: false
original: true
slug: "c++-template-6"
---

C++11 中引入了可变参数模板的特性，可变参数模板就是一个接受可变数目参数的模板函数或者模板类。

可变数目的参数被称为参数包，存在如下两种参数包：

1. 模板参数包：表示零个或多个模板参数
2. 函数参数包：表示零个或多个函数参数

具体如下所示，声明了一个可变参数函数模板。

```cpp
// Args 是一个模板参数包；rest 是一个函数参数包
// Args 表示零个或多个模板类型参数
// rest 表示零个或多个函数参数
template <typename T,typename... Args>
void foo(const T &t, const Args&... rest);
```


<!--more-->

可变参数模板的表示形式和正常可变参数函数类似，都是通过省略号 `...` 来表达零个或者多个的含义。

在一个模板参数列表中，typename... 或者 class... 指出接下来的参数表示零个或多个类型的列表。

比如 foo 就是一个可变参数函数模板，除了一个名为 T 的类型参数，还有一个名为 Args 的模板参数包，这个包表示零个或多个额外的类型参数。

```cpp
template <typename T,typename... Args>
```

一个类型名后面跟一个省略号表示零个或多个给定类型的非类型参数的列表。

在函数参数列表中，如果一个参数的类型是一个模板参数包，则此函数也是一个函数参数包。

对于可变参数模板，编译器会从实参推断模板参数类型，对于一个可变参数模板，编译器还会推断包中参数的数目。

比如如下的调用：

```cpp
int main(){
    int i =0;
    double d = 3.14;
    string s = "variadic template";

    foo(i,s,42,d);
    foo(s,42,"hi");
    foo(d,s);
    foo("hi");
}
```

编译器会为 foo 模板实例化四个不同的版本：

```cpp
    void foo(const int&,const string&,const int&,const double);
    void foo(const string&,const int&,const char[3] &);
    void foo(const double&,const string&);
    void foo(const char[3] &);
```

## sizeof... 运算符获取参数数量

当需要知道参数包中有多少元素时，可以使用 sizeof... 运算符。

sizeof 也返回一个常量表达式，而且不会对其实参求值。

```cpp
template <typename... Args>
void count(Args... args){
    cout << "template parameter packet size is " << sizeof...(Args) << endl;
    cout << "function parameter packet size is " << sizeof...(args) << endl;
}

int main(){
    int i =0;
    double d = 3.14;
    string s = "variadic template";
    
    count(i,s,42,d);
    count(d,s);
    return 0;    
}    
```

## 可变参数函数的递归展开

可变参数函数通常是递归调用的。第一步调用处理包中的第一个实参，然后用剩余实参调用自身。

如下 print 函数所示：

```cpp
// 用来终止递归并打印最后一个元素的函数
// 此函数必须在可变函数参数版本的 print 定义之前声明
template<typename T>
ostream &print(ostream &os,const T &t){
    return os << t ;
}

// 参数包中除了最后一个元素之外的其他元素都会调用这个版本的 print
template<typename T,typename... Args>
ostream &print(ostream &os,const T &t,const Args&... rest){
    os << t << ", ";
    return print(os,rest...);
}

int main(){
    int i =0;
    double d = 3.14;
    string s = "variadic template";
    
    print(cout,i,s,42);
}
```

定义的第一个版本 print 负责终止递归并打印初始调用中的最后一个实参。

第二个版本的 print 是可变参数版本，它打印绑定到 t 的实参，并调用自身来打印函数参数包中剩下的剩余值。

上面程序的关键部分是可变参数函数中对 print 的调用：

```cpp
return print(os,rest...); 
```

可变参数版本的 print 函数接受三个参数：一个 ostream& ，一个 const T& 和一个参数包。

而上面的调用只传递了两个实参，结果就是 rest 中的第一个实参并绑定到 `t` ，剩余实参形成下一个 print 调用的参数包。

因此，在每个调用中，包中的第一个实参被移除，成为绑定 t 的实参，所以上面的调用会递归如下执行：

```cpp
print(cout,i,s,42);

递归展开：

print(cout,i,s,42);     
print(cout,s,42);
print(cout,42);
```

前两个调用只能与可变参数版本的 print 匹配，非可变参数版本是不行的。

最后一个调用，两个 print 版本都是可行的。但非可变参数模板比可变参数模板更特例化，因此编译器会选择非可变参数模板。

可变参数模板的递归调用特性还可以用来做其他的运算，比如求和运算:

```cpp
template<typename T>
T sum(T t){
    return t;
}

template<typename T, typename ...Args>
T sum(T first,Args... rest){
    return first + sum<T>(rest...);
}

int main(){
    sum(1,2,3,4)
}
```

递归展开后的调用顺序如下：

```cpp
sum(4);
sum(3,4);
sum(2,3,4);
sum(1,2,3,4);
```

递归展开的这一特性，在后续模板元编程中还会继续用到的。

## 参考 

1. https://www.cnblogs.com/qicosmos/p/4325949.html