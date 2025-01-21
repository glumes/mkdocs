---
title: "C++ 模板系列小结01-函数模板和类模板"
date: 2021-02-08T17:57:04+08:00
subtitle: ""
tags: ["C++"]
categories: ["code"]
toc: true
 
draft: false
original: true
slug: "c++-template-1"
---


现如今，掌握 C++ 模板技巧并且熟练使用可以说是能力进阶的必备内容了。

在一些优秀的开源项目中经常能看到模板的使用，要是不了解其使用方法，对分析源码都会有些阻碍。

推荐阅读《C++ Templates 中文版》一书，或许可以让你对 C++ 模板有个更加系统的概念，同时辅助阅读网上相关的博客文章加深理解，在代码实践中去掌握提高。

C++ 模板主要可以分为函数模板和类模板，这次就是介绍它们两个。

<!--more-->

## 函数模板

函数模板主要就是将它的参数用类型带代替，对每个不同的类型都会生成对应的函数，所以函数模板可以用多种不同的类型进行调用，它代表的其实是一个函数家族而不是一个函数。

以下是个简单的函数模板示例：

```cpp
#include <iostream>

template <typename T>
T plus(T a,T b){
    return a+b;
}

// 使用引用类型，减少拷贝操作
template<typename T>
inline T const &max(T const &a, T const &b) {
    return a < b ? b : a;
}

int main() {
    plus(1.0, 1.0);
    plus(static_cast<int>(1.0), 2);
    plus<int>(2.0, 3.0);

    max(2,1);
    max(static_cast<int>(2.0),1);
    max<double>(2.0,3.0);
    return 0;
}
```

通常而言，并不是把模板编译成一个可以处理任何单一类型的单一实体，而是针对实例模板参数的每种类型，都从模板产生一个不同的实体。

针对上面的使用，整型和浮点型都生成了一个对应的函数。

对于函数模板的参数，是可以在调用时显示指定的，另外可以在调用时对类型做强制类型转换，这是因为模板不允许自动类型转换，所以需要我们手动来完成。

### 函数模板的重载

和普通函数一样，函数模板也可以重载，也就是相同函数名称可以具有不同的函数定义。

```cpp
inline int const &max(int const &a, int const &b) {
    return a < b ? b : a;
}

template<typename T>
inline T const &max(T const &a, T const &b) {
    return a < b ? b : a;
}

template<typename T>
inline T const &max(T const &a, T const &b, T const &c) {
    return ::max(::max(a,b),c);
}
```

如上所示，一个非函数模板可以合一个同名的函数模板同时存在，而且该函数模板还可以被实例化为这个非函数模板。

对于非函数模板和同名的函数模板，如果其他条件都相同的话，那么在调用的时候，重载解析过程通常会调用非函数模板，而不会从该模板产生一个实例。

对于函数的重载，一般来说改变的内容最好是下面两种情况：

* 改变参数的数目
* 显示地指定模板参数

遵循以上两点，避免在重载函数调用过于复杂，从而导致匹配出差。

## 类模板

与函数模板相似，类可以被一种或者多种类型参数化。

以下代码是一个简单的类模板示例：

```cpp
template<typename T>
class Stack {
private:
    std::vector<T> elems;
public:
        void push(T const&);
        void pop();
        T top() const;
        bool empty() const{
            return elems.empty();
        }
};

template<typename T>
void Stack<T>::push(const T & elem) {
    elems.push_back(elem);
}

template<typename T>
void Stack<T>::pop() {
    if (!elems.empty()){
        elems.pop_back();
    }
}

template<typename T>
T Stack<T>::top() const {
    if (!elems.empty()){
        return elems.back();
    }
    throw std::out_of_range("Stack::top(); empty Stack");
}

int main() {
    Stack<int> intStack;
    intStack.push(1);
    intStack.top();
    intStack.pop();
    return 0;
}
```

类模板的声明和函数模板声明很类似，在声明之前都用 `typename` 声明作为类型参数的标识符。

在定义类模板的成员函数时，关于成员函数的实现，可以在类声明里面去实现，也可以放到外面去实现。

如果是在类外面声明的，那么必现要指定该成员函数是一个函数模板，而且还需要使用这个类模板的完整类型限定符。

另外，只有那些被调用的成员函数，才会产生这些函数的实例化代码。

### 类模板的特化

可以使用模板实参来特化类模板。

和函数模板的重载类似，通过特化类模板，可以优化基于某种特定类型的实现，或者克服某种特定类型在实例化类模板时所出现的不足。


另外，如果要特化一个类模板，还要特化该类模板的所有成员函数。虽然也可以只特化某个成员函数，但这个做法并没有特化整个类，也就没有特化整个类模板。

为了特化一个类模板，必须在起始处声明一个 `template<>` ，接下来声明用特化类模板的类型。

如下代码所示：

```cpp
template<>
class Stack<std::string> {
private:
        std::deque<std::string> elems;
public:
        void push(std::string const&);
        void pop();
        std::string top() const;
        bool empty() const{
            return elems.empty();
        }
};

void Stack<std::string>::push(const std::string &elem) {
    elems.push_back(elem);
}

void Stack<std::string>::pop() {
    if (!elems.empty()){
        elems.pop_back();
    }
}

std::string Stack<std::string>::top() const {
    if (!elems.empty()){
        return elems.back();
    }
    throw std::out_of_range("Stack<std::string>::top(); empty Stack");
}

int main() {
    Stack<std::string> stringStack;
    intStack.push("string");
    intStack.top();
    intStack.pop();
    return 0;
}
```

进行类模板的特化时，每个成员函数都必须重新定义为普通函数，原来函数模板中的每个 T 也相应地被进行特化的类型取代。


### 局部特化

上面的特化可以称之为全特化，另外类模板还可以局部特化，在特定的环境下制定类模板的实现，并且要求某些模板参数仍然必须由用户来定义。

如下代码所示：

```cpp
template<typename T1,typename T2>
class MyClass{

};

// 局部特化，两个模板参数具有相同的类型
template<typename T>
class MyClass<T,T>{

};

// 局部特化， 第二个模板参数的类型是 int
template<typename T>
class MyClass<T,int>{

};

// 局部特化，两个模板参数都是指针类型
template<typename T1,typename T2>
class MyClass<T1*,T2*>{

};
```

对于局部特化，其成员函数同样要指定是模板函数，并且要使用特化后的类模板完整类型限定符。

### 缺省模板实参

对于类模板，可以为类模板参数定义默认值。这些值就被称为缺省模板实参，而且它们还可以引用之前的目标参数。

如下代码所示：

```cpp
template <typename T,typename CONT = std::vector<T> >
class MyStack{
private:
    CONT elems;
public:
    void push(T const&);
    void pop();
    T top() const;
    bool empty() const{
        return elems.empty();
    }
};

template<typename T, typename CONT>
void MyStack<T, CONT>::push(const T &elem) {
    elems.push_back(elem);
}

template<typename T, typename CONT>
void MyStack<T, CONT>::pop() {
    if (!elems.empty()){
        elems.pop_back();
    }
}

template<typename T, typename CONT>
T MyStack<T, CONT>::top() const {
    if (!elems.empty()){
        return elems.back();
    }
    throw std::out_of_range("MyStack<>::top(); empty Stack");
}

int main() {
    MyStack<int> intMyStack;
    intMyStack.push(1);
    intMyStack.top();
    intMyStack.pop();

    MyStack<double, std::deque<double>> doubleMyStack;
    doubleMyStack.push(0.1);
    doubleMyStack.top();
    doubleMyStack.pop();
    return 0;
}
```

以上代码中，如果不传递第二个模板参数，那么就使用默认的 vector 来管理元素。如果传了，就使用指定的类型来管理元素。


## 小结

以上就是关于 C++ 模板中函数模板和类模板的小总结，基本上还是很容易理解的，就是用模板参数 T 代替具体的类型，并且还可以对函数模板进行重载，对类模板进行特化，类模板还可以指定默认类型参数。

