---
title: "C++ 模板系列小结05-模板类型作为模板参数"
date: 2021-02-09T12:11:34+08:00
subtitle: ""
tags: ["C++"]
categories: ["code"]
toc: true
 
draft: false
original: true
slug: "c++-template-5"
---

在前面的文章中，模板参数除了是类型之外，还可以是非类型参数，但只有整型和指向外部链接对象的指针才可以。

除此之外，模板类型同样可以作为类型参数，并且还很有用处。

<!--more-->

在之前定义过如下的类模板：

```cpp
template <typename T,typename CONT = std::vector<T> >
class Stack{

};

int main() {
    Stack<int> stack1;
    Stack<int,std::vector<int>> stack2;
    return 0;
}
```

要使用如上的类模板，并且不使用默认参数的话，那么就需要指定容器类型和容器所包含的元素类型。

但如果借助模板参数的话，就可以只指定容器的类型而不需要指定所包含元素的类型。

为了达到上述目标，就需要把第二个模板参数指定为模板的模板参数，声明方式如下：

```cpp
template <typename T, template<typename ELEM> class CONT = std::deque>
class Stack{
private:
    CONT<T> elems;
public:
    void push(T const&);
    void pop();
    T top() const;
    bool empty() const{
        return elems.empty();
    }
};
```

不同之处在于，第二个模板参数选择被声明为一个类模板。

```cpp
template<typename ELEM> class CONT = std::deque
```

在使用时，第二个参数必现得是一个类模板，并且由第一个模板参数传递进来的类型进行实例化，如下代码所示：

```cpp
CONT<T> elems;
```

这也是比较特别的地方，使用第一个模板参数作为第二个模板参数的实例化类型。

一般来说，可以使用类模板内部的任何类型来实例化模板的模板参数。

## 使用 Class 关键字

作为模板参数的声明，通常可以使用 typename 关键字来替换 class 。但是上面的 CONT 是为了定义一个类，一个模板类，因此只能使用关键字 `class` 。

所以如下的声明反而是错误的：

```cpp
template <typename T, template<typename ELEM> typename CONT = std::deque>
```

但也不是绝对的，在 C++17 中也是允许上面声明方式了。

另外，由于代码中实际上用不到 "模板的模板参数" 中的模板参数，也就是 `ELEM` ，所以它是可以省略不写的。

省略之后的声明方式如下：

```cpp
template <typename T, template<typename> class CONT = std::deque>
class Stack{
private:
    CONT<T> elems;
public:
    void push(T const&);
    void pop();
    T top() const;
    bool empty() const{
        return elems.empty();
    }
};
```

接下来在定义函数实现时，也要做相应的修改：

```cpp
template<typename T, template<typename> class CONT>
void Stack<T>::push(const T &elem) {
    elems.push_back(elem);
}

template<typename T, template<typename> class CONT>
void Stack<T>::pop() {
    if (!elems.empty()) {
        elems.pop_back();
    }
}

template<typename T, template<typename> class CONT>
T Stack<T>::top() const {
    if (!elems.empty()) {
        return elems.back();
    }
    throw std::out_of_range("Stack::top(); empty Stack");
}
```

成员函数的声明要和类模板的声明一致。

> 另外，函数模板是不支持模板类型作为模板参数的。

## 模板的模板参数实参匹配

完成了以上的声明定义之后，还不能直接使用 Stack 类模板，因为 CONT 的默认值 std::deque 和模板的模板参数 CONT 并不匹配。

这是因为模板的模板实参（比如 std::deque）是一个具有参数 A 的模板，它将替换模板的模板参数（比如 CONT），而模板的模板参数是一个具有参数 B 的模板。

std::deque 的声明如下：

```cpp
template <class _Tp, class _Allocator = allocator<_Tp> > class _LIBCPP_TEMPLATE_VIS deque;
```

匹配过程要求参数 A 和参数 B 必现完全匹配。然后在这里，并没有考虑到模板的模板实参的默认模板参数，也就是上面的 allocator ，从而也就是使 B 中缺少了这些默认参数值，当然就不能获得精确的匹配。

std::deque 还有一个参数，也就是第二个参数，内存分配器 allocator ，它有一个默认值，但在匹配 std::deque 的参数和 CONT 的参数时，并没有考虑到这个参数值。

因此修改类的声明，让 CONT 的参数是具有两个模板参数的容器。

```cpp
template <typename T, template<typename ELEM,typename ALLOC = std::allocator<ELEM>> class CONT = std::deque>
class Stack{
private:
    CONT<T> elems;
public:
    void push(T const&);
    void pop();
    T top() const;
    bool empty() const{
        return elems.empty();
    }
};
```

由于 ALLOC 用不到，实际上它也是可以省略的。

```cpp
template <typename T, template<typename ELEM,typename = std::allocator<ELEM>> class CONT = std::deque>
class Stack{
private:
    CONT<T> elems;
public:
    void push(T const&);
    void pop();
    T top() const;
    bool empty() const{
        return elems.empty();
    }
};
```

相应的，也要将成员函数修改了，凡是用不到的都可以省略。

```cpp
template <typename T, template<typename ,typename > class CONT>
void Stack<T,CONT>::push(const T &elem) {
    elems.push_back(elem);
}

template <typename T, template<typename ,typename > class CONT>
void Stack<T,CONT>::pop() {
    if (!elems.empty()) {
        elems.pop_back();
    }
}
template <typename T, template<typename ,typename > class CONT>
T Stack<T,CONT>::top() const {
    if (!elems.empty()) {
        return elems.back();
    }
    throw std::out_of_range("Stack::top(); empty Stack");
}
```

将 Stack 的函数声明和实现拼起来，就是最终的版本，使用起来也不需要再指定容器所包含的类型了。

```cpp
int main() {
    Stack<int,std::vector> intStack;
    intStack.push(2);
    intStack.top();
    intStack.pop();
    return 0;
}
```


## 小结

在本篇中模板的模板参数声明时，使用到了 class 关键字，要注意它的使用场景，因为模板的模板参数是一个类，所以要用 class 关键字。

参考 std::deque 的声明，前面两个 class 关键字可以用 typename 替换，后面一个就不行了。

```cpp
template <class _Tp, class _Allocator = allocator<_Tp> > class _LIBCPP_TEMPLATE_VIS deque;
```

另外，在 C++ 17 之后，也是可以用 typename 关键字来声明模板的模板参数类型了，但考虑到 C++ 17 普及使用程度不高，还是按照 C++ 11 的标准来吧。


