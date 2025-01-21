---
title: "C++ 模板系列小结02-非类型模板参数"
date: 2021-02-08T20:14:18+08:00
subtitle: ""
tags: ["C++"]
categories: ["code"]
toc: true
 
draft: false
original: true
slug: "c++-template-2"
---

前面已经介绍了函数模板和类模板，还介绍了类模板的默认参数，在代码示例中都是用具体类型来作为模板参数的。

实际上，模板参数不局限于类型，普通值也可以作为模板参数，也就是本篇要讲的内容：非类型模板参数。

<!--more-->

## 非类型模板参数

非类型模板参数的使用和类型模板参数有些不一样，不需要 `typename` 关键字了，直接指定具体类型。

如下代码所示：

```cpp
template<typename T, int MAXSIZE = 20>
class STACK {
private:
    T elems[MAXSIZE];
    int numElems;

public:
    STACK();

    void push(T const &);
    void pop();
    T top() const;
    bool empty() const {
        return numElems == 0;
    }

    bool full() const {
        return numElems == MAXSIZE;
    }
};

template<typename T, int MAXSIZE>
STACK<T, MAXSIZE>::STACK() :numElems(0) {

}

template<typename T, int MAXSIZE>
void STACK<T, MAXSIZE>::push(const T &elem) {
    if (numElems == MAXSIZE) {
        throw std::out_of_range("Stack<>::push(): stack is full");
    }
    elems[numElems] = elem;
    numElems++;
}

template<typename T, int MAXSIZE>
void STACK<T, MAXSIZE>::pop() {
    if (numElems <= 0) {
        throw std::out_of_range("Stack<>::pop(): empty stack");
    }
    numElems--;
}

template<typename T, int MAXSIZE>
T STACK<T, MAXSIZE>::top() const {
    if (numElems <= 0) {
        throw std::out_of_range("Stack<>::top(): empty stack");
    }
    return elems[numElems - 1];
}

int main() {
    STACK<int> intStack;
    intStack.push(1);
    intStack.top();
    intStack.pop();

    STACK<int,10> int10Stack;
    int10Stack.push(1);
    int10Stack.top();
    int10Stack.pop();
    return 0;
}
```

在使用非类型模板参数的时候，也就是基于值的模板参数，要么显示地指定这些值，要么作为默认参数有默认值。

## 非类型的函数模板参数

除了上面的类模板，函数模板同样可以用非类型的模板参数。

```cpp
template <typename T,int VAL = 3 >
T addValue(T const&x){
    return x + VAL;
}

int main{
    addValue(10);
    return 0;
}
```

可以看到函数模板同样是可以用默认参数值的。

## 非类型模板参数的限制

不是所有的类型都可以作为非类型模板参数，它是有限制的。

> 通常而言，只有常整数（包括枚举值）或者指向外包链接对象的指针 才可以，而浮点数和类对象是不允许作为非类型模板参数的。


```cpp
template<double VAT>
double process(double v){
    return v * VAT;
}
```

以上的代码示例就是不行的。


## 小结

非类型模板参数只是 C++ 模板知识点里面一个小方面了，但在某些场景还在能发挥大作用的。