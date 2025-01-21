---
title: "C++ 模板系列小结04-类模板中的成员模板"
date: 2021-02-09T12:11:01+08:00
subtitle: ""
tags: ["C++"]
categories: ["code"]
toc: true
 
draft: false
original: true
slug: "c++-template-4"
---

在之前的类模板中，只在类声明时用了 typename 指定模板参数类型，之后的成员函数复用模板参数类型。

但实际上，类成员也可以是模板，嵌套类和成员函数都可以作为模板。

<!--more-->

如下代码所示：

```cpp
template<typename T>
class Stack {
private:
    std::deque<T> elems;

public:
    void push(T const &elem);

    void pop();

    T top() const;

    bool empty() const {
        return elems.empty();
    }

    template<typename T2>
    Stack<T> &operator=(Stack<T2> const &);
};

template<typename T>
void Stack<T>::push(const T &elem) {
    elems.push_back(elem);
}

template<typename T>
void Stack<T>::pop() {
    if (!elems.empty()) {
        elems.pop_back();
    }
}

template<typename T>
T Stack<T>::top() const {
    if (!elems.empty()) {
        return elems.back();
    }
    throw std::out_of_range("Stack::top(); empty Stack");
}

template<typename T>
template<typename T2>
Stack<T> &Stack<T>::operator=(const Stack<T2> &op2) {
    if ((void *) this == (void *) &op2) {
        return *this;
    }

    Stack<T2> tmp(op2);
    elems.clear();
    while (!tmp.empty()) {
        elems.push_front(tmp.top());
        tmp.pop();
    }
    return *this;
}
```

以上类和之前定义的 Stack 类略有差别，主要是自定义了赋值运算符，可以将两个不同类型的 Stack 容器内容进行转换。

在自定义的赋值运算符中，除了类模板声明的类型参数 T 之外，还多了一个 T2 的类型参数，此时的成员函数也成了一个模板函数。


在类模板中，成员函数的模板参数可以和类的模板参数不同，但在定义中必须添加两个模板参数列表。

其中，第一个为类的模板参数，第二个为成员函数的。

```cpp
template<typename T>
class MyClass {
public:
    template<typename T2> 
    void func(T2 t2);
};

template<typename T>
template<typename T2>
void MyClass<T>::func(T2 t2) {

}
```

同样的，类模板的成员模板也只有在调用时才会实例化，否则不会。

成员模板既可以在类模板中，也可以在普通类中。

```cpp
class MyClass2{
public:
    template<typename T>
    void func(T t);
};

template<typename T>
void MyClass2::func(T t) {

}
```



