---
title: "C++ 标准容器库小结"
date: 2018-11-19T10:23:00+08:00
subtitle: ""
tags: ["C++","STL"]
categories: ["code"]
toc: true
draft: false
slug: "c++-stl-container-overview"
original: false
addwechat: false
 
author: "星陨"
---


《C++ Primer》 第五版笔记摘录~~~

<!--more-->

C++ 容器的种类如下：

*	顺序容器
	*	vector
	*	list
	*	deque
*	关联容器
	*	map
	*	set
	*	multimap
	*	multiset
*	容器适配器
	*	stack
	*	queue
	*	priority_queue



## 顺序容器

顺序容器提供了控制元素存储和访问顺序的能力，这种顺序不依赖于元素的值，而是与元素加入容器的位置相对应。

以下表格是标准库中的顺序容器，它们都提供快速顺序访问元素的能力，但是这些容器在以下方面有不同的性能这种：

*	向容器添加或从容器中删除元素的代价
*	非顺序访问容器中元素的代价


|容器类型|介绍|
|---|---|
|vector|可变大小数组。支持快速随机访问。在尾部之外的位置插入或删除元素可能很慢|
|deque|双端队列。支持快速随机访问。在头尾位置插入/删除速度很快|
|list|双向链表。只支持双向顺序访问。在 list 中任何位置进行插入/删除操作速度都很快|
|forward_list|单向链表。只支持单向顺序访问。在链表任何位置进行插入/删除操作速度都很快|
|array|固定数组大小。支持快速随机访问。不能添加或删除元素|
|string|与 vector 相似的容器，但专门用于保存字符。随机访问快。在尾部插入/删除速度快|

所有容器有一些共有的操作。

### 顺序容器操作

顺序容器几乎可以保存任意类型的元素，特别是，我们可以定义一个容器，其元素的类型是另一个容器。

```cpp
vector<vector<string>> lines ; // vector 的 vector
```
### 容器的一些通用操作

#### 容器定义和初始化

每个容器类型都定义了一个默认构造函数。除 array 之外，其他容器的默认构造函数都会创建一个指定类型的空容器，且都可以接受指定容器大小和元素初始值的参数。

容器定义和初始化方法见下图：

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1fwovi7exigj20pj0fnwp1.jpg)

#### 将一个容器初始化为另一个容器的拷贝

将一个容器创建为另一个容器的拷贝的方法有两种：

1. 可以直接拷贝整个容器。
2. 拷贝由一个迭代器对指定的元素范围（除 array 外）。

```cpp
    list<string> authors = {"milton", "shake", "audio"};
    list<string> list2(authors);
    cout << *list2.begin() << endl;

    vector<const char *> articles = {"a", "an", "use"};
    forward_list<string> word(articles.begin(), articles.end());
    cout << *word.begin() << endl;
```

当将一个容器初始化为另一个容器的拷贝时，两个容器的容器类型和元素类型都必须相同。

不过，当传递迭代器参数来拷贝一个范围时，就不要求容器类型是相同的了，而且，新容器和原容器的元素类型也可以不同，只要能将要拷贝的元素转换为要初始化的容器的元素类型即可。

#### 迭代器

所有标准库容器都可以使用迭代器，但是其中只有少数几种才同时支持下标运算符。

和指针类似，迭代器也提供了对对象的间接访问。就迭代器而言，其对象是容器中的元素或者 string 对象中的字符。使用迭代器可以访问某个元素，迭代器也能从一个元素移动到另外一个元素。

迭代器有有效和无效之分，这一点和指针差不多。有效的迭代器或者指向某个元素，或者指向容器中尾元素的下一位置：其他所有情况都属于无效。


#### 迭代器的使用

获取迭代器不是使用取地址符，有迭代器的类型同时拥有返回迭代器的成员。

比如，这些类型都拥有名为 `begin` 和 `end` 的成员，其中 begin 成员负责返回指向第一个元素的迭代器，end 成员则负责返回指向容器尾元素的下一位置的迭代器，常称作 `尾后迭代器` ，也就是说该迭代器指示的是容器一个本不存在的 `尾后元素` 。

特殊情况下如果容器为空，则 begin 和 end 返回的就是同一个迭代器。

```cpp
    string content = "content";
    string::iterator begin = content.begin();
    auto end = content.end();
    while (begin != end) {
        cout << (*begin) << endl;
        begin++;
    }
```

#### 迭代器的类型

一般来说不需要知道迭代器的精确类型，而实际上，拥有迭代器的标准库类型使用 `iterator` 和 `const_iterator` 来表示迭代器的类型。

```cpp
    vector<int>::iterator it ;    // it 能读写 vector<int> 的元素
    string::iterator it2 ;        // it2 能读写 string 对象中的字符
    
    vector<int>::const_iterator it3;   // it3 只能读元素，不能写元素
    string::const_iterator it4;        // it4 只能读字符，不能写字符
```

begin 和 end 返回的具体类型由对象是否是常量决定，如果对象是常量，返回的是 const_iterator ，如果对象不是常量，返回的是 iterator 。

如果对象只需读操作，无需写操作，那么迭代器最好使用常量类型，为了便于专门得到 const_iterator 类型的返回值，C++ 11 新标准引入了两个新函数，分别是 `cbegin` 和 `cend` 。

类似于 begin 和 end ，上述两个新函数也分别返回指示容器第一个元素或最后元素下一位置的迭代器，有所不同的是，不论 vector 对象（或 string 对象）本身是否是常量，返回值都是 const_iterator 。

#### 迭代器解引用和成员访问操作

解引用迭代器可以获得迭代器所指的对象，如果该对象的类型恰好是类，就有可能希望进一步访问它的成员。

```cpp
    vector<string> strings(3, "content");
    auto it = strings.begin();
    // 迭代器解引用
    cout << *it << endl;
    // 迭代器解引用之后访问成员
    cout << (*it).empty() << endl;
    // 通过箭头运算符 -> 把解引用和成员访问结合在一起
    cout << it->empty() << endl;
```

C++ 语言定义了箭头运算符 `->` ，能够把解引用和成员访问两个操作结合在一起。


### 顺序容器添加和删除元素

除了 array 固定大小之外，所有标准容器库都提供灵活的内存管理。在运行时可以动态添加或删除元素来改变容器大小。

以下是向顺序容器中添加元素的操作。

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1fwo8b7wyadj20k30g4wlt.jpg)

当使用如上的操作时，必须记得不同容器使用不同的策略来分配元素空间，这些策略直接影响性能。

添加主要操作函数有如下：

*	push_back
	*	将一个元素追加到容器的尾部，除了 array 和 forward_list 之外，每个顺序容器都支持。
*	push_front
	*	将元素插入到容器头部，除了 push_back，list、forward_list 和 deque 容器还支持 push_front 操作。
*	insert
	*	在容器中任意位置插入 0 个或多个元素，vector、deque、list 和 string 都支持 insert 操作。
	*	forward_list 提供了特殊版本的 insert 成员。
	*	每个 insert 函数都接受一个迭代器作为其第一个参数，迭代器指出了在容器中什么位置放置新元素。
	*	insert 函数将元素插入到迭代器所指定的位置之前。
*	emplace
	*	C++ 新标准引入了三个新成员，emplace_front、emplace 和 emplace_back，这些操作构造而不是拷贝元素。这些操作分别对应于 push_front、insert 和 push_back ，允许我们将元素放置在容器头部、一个指定位置之前或者容器尾部。
	*	当调用 push 或 insert 成员函数时，我们将元素类型的对象传递给它们，这些对象被拷贝到容器中。
	*	而当我们调用一个 emplace 成员函数时，则是将参数传递给元素类型的构造函数，emplace 成员使用这些参数在容器管理的内存空间直接构造元素。


删除主要操作函数如下：

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1fwoydkugt2j20t10il4b7.jpg)

*	pop_front 和 pop_back
	*	pop_front 和 pop_back 成员函数分别删除首元素和尾元素。
	*	与 vector 和 string 不支持 push_front 一样，这些类型也不支持 pop_front 。
	*	forward_list 不支持 pop_back 。
	*	不能对一个空容器执行弹出操作。
*	erase
	*	从容器指定位置删除元素。
	*	可以删除由一个迭代器指定的单个元素，也可以删除由一对迭代器指定的范围内的所有元素。
	*	两种形式的 erase 都返回指向删除的（最后一个）元素之后位置的迭代器。
	*	接受一对迭代器的 erase 运行删除一个范围内的元素。


代码示例如下：

```cpp
    vector<string> str(5, "content");
    // 添加尾元素
    str.push_back("push_back");
    // 通过构造函数添加尾元素
    str.emplace_back("emplace");
    // 插入元素
    str.insert(str.cbegin(), "insert");

    auto begin = str.cbegin();
    auto end = str.cend();
    while (begin != end) {
        cout << *begin << endl;
        begin++;
    }

    cout << endl << "delete" << endl;

    // 删除尾元素
    str.pop_back();
    // 通过迭代器删除单个元素
    str.erase(str.cbegin());
    // 通过迭代器删除多个元素
    str.erase(str.cbegin() + 1, str.cend() - 1);

    auto del_begin = str.cbegin();
    auto del_end = str.end();
    while (del_begin != del_end) {
        cout << *del_begin << endl;
        del_begin++;
    }
```



## 容器适配器

除了顺序容器外，标准库还定义了三个顺序容器适配器：

*	stack
*	queue
*	priority_queue

适配器是标准库中的一个通用概念，容器、迭代器和函数都要适配器。

本质上，一个适配器是一种机制，能使某种事物的行为看起来像另外一种事物一样，一个容器适配器接受一种已有的容器类型，使其行为看起来像一种不同的类型。例如，stack 适配器接受一个顺序容器（除 array 或 forward_list 外），并使其操作起来像一个 stack 一样。


所有容器适配器都支持的操作如下：

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1fxd77po68vj20ib090whe.jpg)


### 定义适配器

每个适配器都定义两个构造函数：默认构造函数创建一个空对象，接受一个容器的构造函数拷贝该函数来初始化适配器。

例如，假如 deq 是一个 deque<int> ，可以用 deq 来初始化一个新的 stack 。

```cpp
stack<int> stk(deq);	// 从 deq 拷贝元素到 stk
```

默认情况下，stack 和 queue 是基于 deque 实现的，priority_queue 是在 vector 之上实现的。我们可以创建一个适配器时将一个命名的顺序容器作为第二个类型参数，来重载默认容器类型。

```cpp
// 在 vector 上实现的空栈
stack<string, vector<string>> str_stk;
// str_stk2 在 vector 上实现，初始化时保存 svec 的拷贝
stack<string, vector<string>> str_stk2(svec);
```

对于一个给定的适配器，可以使用哪些容器是有限制的。所有适配器都要求容器具有添加和删除元素的能力。因此，适配器不能构造在 array 之上，类似的，也不能用 forward_list 来构造适配器，因为所有适配器都要求容器具有添加、删除以及访问尾元素的能力。

stack 只要求 push_back、pop_back 和 back 操作，因此可以使用除 array 和 forward_list 之外的任何容器来构造 stack 。

queue 适配器要求 back、push_back、front 和 push_front ，因此它可以构造于 list 或 queue 之上，但不能基于 vector 构造。

priority_queue 除了 front、push_back 和 pop_back 操作之外还要求随机访问能力，因此它可以构造于 vector 或 queue 之上，但不能基于 list 构造。

### 栈适配器


stack 类型定义在 stack 头文件中，支持的操作如下表：

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1fxd78arzj4j20ig04rwga.jpg)


代码示例如下：

```cpp
stack<int> intStack; // 声明一个空栈
for(size_t ix = 0;ix != 10;ix++){
	intStack.push(ix);
}
while(!intStack.empty()){
	int value = intStack.top();
	intStack.pop();
}
```

每个容器适配器都基于底层容器类型的操作定义了自己的特殊操作。我们只可以使用适配器操作，而不能使用底层容器类型的操作。

### 队列适配器

queue 和 priority_queue 适配器定义在 queue 头文件中，支持的操作如下表：

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1fxd78arzj4j20ig04rwga.jpg)

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1fxd7dnqwazj20ib02smxx.jpg)


标准库 queue 使用先进先出的存储和访问策略。进入队列的对象被放置到队尾，而离开队列的对象从队首删除。

priority_queue 允许我们为队列中的元素建立优先级。新加入的元素会排在所有优先级比它低的已有元素之前。


## 顺序容器小结

标准库容器是模板类型，用来保存给定类型的对象。在一个顺序容器中，元素是按顺序存放的，通过位置来访问。顺序容器有公共的标准接口：如果两个顺序容器都提供一个特定的操作，那么这个操作在两个容器中具有相同的接口和含义。

所有容器（除 array 外）都提供高效的动态内存管理。我们可以向容器中添加元素，而不必担心元素存储在哪里。容器负责管理自身的存储。vector 和 string 都提供更细致的内存管理控制，这是通过它们的 reserve 和 capacity 成员函数来实现的。

很大程度上，容器只定义了极少的操作。每个容器都定义了构造函数、添加和删除元素的操作、确定容器大小的操作以及返回特定元素迭代器的操作。其他一些有用的操作，如排序或搜索，并不是由容器类型定义的，而是由标准库算法实现的。

当使用添加或删除元素的容器操作时，必须注意这些操作可能使指向容器中的元素的迭代器、指针或引用失效。很多会使迭代器失效的操作，如 insert 和 erase ，都会返回一个新的迭代器，来帮助程序员维护容器中的位置。如果循环程序中使用了改变容器大小的操作，就要尤其小心其中迭代器、指针和引用的使用。


----


## 关联容器

关联容器和顺序容器有着根本的不同：关联容器中的元素是按关键字来保存和访问的，与之相对，顺序容器中的元素是按它们在容器中的位置来顺序保存和访问的。

关联容器支持高效的关键字查找和访问，两个主要的关联容器类型是 `map` 和 `set` 。

map 中的元素是一些 关键字-值 对：关键字起到索引的作用，值则表示与索引相关联的数据。

set 中每个元素只包含一个关键字，set 支持高效的关键词查询操作，检查一个给定关键字是否在 set 中。

标准库提供了 8 个关联容器，如下图：

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1fwoym8cn2ij20t50cfn4e.jpg)

这 8 个关联容器间的不同体现在三个维度上：

*	每个容器或是一个 set 或者是一个 map 。
*	或者要求不重复的关键字，或者允许重复关键字。
*	按顺序保存元素，或无序保存。

允许重复关键字的容器的名字中都包含单词 `multi` 。

不保持关键字按顺序存储的容器的名字都以单词 `unordered` 开头。

因此，一个 `unordered_multi_set` 是一个允许重复关键字，元素无序保存的集合，而一个 set 则是要求不重复关键字，有序存储的集合。

> 类型 map 和 multimap 定义在头文件 map 中，set 和 multiset 定义在头文件 set 中，无序容器则定义在头文件 unordered_map 和 unordered_set 中。


map 的简单示例：
```cpp
    map<string, size_t> word_count;
    string word;
    while (cin >> word) {
        ++word_count[word];
    }
    for (const auto &w : word_count) {
        cout << w.first << " occurs" <<
             w.second << ((w.second > 1) ? " times" : " time") << endl;
    }
```

set 的简单示例：

```cpp
    map<string, size_t> word_count;
    set<string> exclude = {"The", "But", "And"};
    string word;
    while (cin >> word) {
        if (exclude.find(word) == exclude.end()) {
            ++word_count[word];
        }
    }
```
### pair 类型

在开始关联容器的操作之前，需要了解 `pair` 的标准库类型，它定义在头文件 `utility` 中。

一个 pair 保存两个数据成员。pair 是一个用来生成特别类型的模板。当创建一个 pair 时，我们必须提供两个类型名，pair 的数据成员将具有对应的类型，两个类型不要求一样：

```cpp
pair<string,string> anon;
pair<string,size_t> word_count;
pair<string,vector<int>> line;
pair<string,string> author{"James","Joyce"};
```

pair 的默认构造函数对数据成员进行值初始化，当然也可以为每个成员提供初始化器。

pair 的数据成员是 public 的。两个成员分别命名为 first 和 second 。可以用普通的访问符来访问它们。

具体 pair 上的一些操作如下：

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1fwr9pwm1vdj21fo0w21hy.jpg)

### 关联容器迭代器

当解引用一个关联容器迭代器时，我们会得到一个类型为容器的 value_type 的值的引用。对 map 而言，value_type 是一个 pair 类型，其 first 成员保存 const 的关键字，second 成员保存值。

```cpp
    // 获得指向 word_count 中一个元素的迭代器
    auto map_it = word_count.begin();
    // *map_it 是指向一个 pair<const string,size_t> 对象的引用
    cout << map_it->first;
    cout << map_it->second;
    // 可以通过迭代器改变元素
    ++map_it->second;
```

> 一个 map 的 value_type 是一个 pari ，我们可以改变 pair 的值，但不能改变关键字成员的值。

对于 set 而言，它的 iterator 和 const_iterator 迭代器都只允许只读访问 set 中的元素，而且，set 中的关键字也是 const 的 。

```cpp
    set<int> iset = {0, 1, 2, 3, 4, 5};
    set<int>::iterator set_it = iset.begin();
    if (set_it != iset.end()) {
        // *set_it = 42;   // 关键字只读，不能赋值
        cout << *set_it << endl;
    }
```

### 向 map 添加元素：

对一个 map 进行 insert 操作时，必须记住元素类型是 `pair` 。

通常，对于一个想要插入的数据，并没有一个现成的 pair 对象，可以在 insert 的参数列表中创建一个 `pair` 。

```cpp
    map<string, size_t> word_count;
    string word = "word";
    // 向 map 中插入数据的方式的 4 中方式
    word_count.insert({word, 1});
    word_count.insert(make_pair(word, 1));
    word_count.insert(pair<string, size_t>(word, 1));
    word_count.insert(map<string, size_t>::value_type(word, 1));
```

在新标准下，创建一个 pair 最简单的方法是在参数列表中使用花括号初始化，也可以用 `make_pair` 或显示构造 `pair` 。或者通过 `map<string,size_t>::value_type` 构造一个恰当的 pair 类型，并构造该类型的一个对象，然后插入到 map 中。

关联容器的 insert 操作如下表：

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1fwpbcg2ekoj20re0e67cn.jpg)

insert （或 emplace）返回的值依赖于容器类型和参数。对于不包含重复关键字的容器，添加单一元素的 insert 和 emplace 版本返回一个 pair ，告诉我们插入操作是否成功。

pair 的 first 成员是一个迭代器，指向具有给定关键字的元素；second 成员是一个 bool 值，指出元素是插入成功还是已经存在于容器中。如果关键字已在容器中，则 insert 什么事情也不做，且返回值中的 bool 部分为 false。如果关键字不存在，元素被插入容器，且 bool 值为 true 。

比如如下代码：

```cpp
	map<string, size_t> word_count;
    string word = "word";
    for (int i = 0; i < 5; ++i) {
	    // key 是 word ，value 是 1
        auto ret = word_count.insert({word, 1});
        if (!ret.second) {
            ++ret.first->second;
        }
    }
```

在 if 中根据 second 来判断是否插入成功，如果已经存在了，就把 pair 对应的 value 值递增。

这里要重点查看递增语句，比较难理解：


```cpp
	++ret.first->second;
	// 等价于
	++((ret.first)->second);
```

`ret.first` 是 pair 的第一个成员，是一个 map 迭代器，指向具有给定关键字的元素。
`ret.first->` 解引用此迭代器，提取 map 中的元素，元素也是一个 pair 。
`ret.first->second` map 中元素的值部分。
`++ret.first->second` 递增此值。

### 向 multiset 或者 multimap 添加元素

如果希望添加具有相同关键字的多个元素，就应该使用 `multimap` 。由于一个 multi 容器中的关键字不必唯一，在这些类型上调用 insert 总会插入一个元素。

```cpp
    multimap<string, string> authors;
    // 插入两个元素，关键字都为 Barth
    authors.insert({"Barth", "fadt"});
    authors.insert({"Barth", "asdd"});
```

对允许重复关键字的容器，接受单个元素的 insert 操作返回一个指向新元素的迭代器。这里无须返回一个 bool 值，因为 insert 总是向这类容器加入一个新元素。


###  map 移除元素

关联容器定义了三个版本的 `earse` 。

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1fwpc64o9buj20re07l79v.jpg)

与顺序容器一样，我们可以通过传递给 erase 一个迭代器或一个迭代器对来删除一个元素或者一个元素范围。另外还有一个额外的 earse 操作，它接受一个 `key_type` 参数，它会删除所有匹配给定关键字的元素（如果存在的话），返回实际删除的元素的数量。


对于不保存重复关键字的容器，erase 的返回值总是 0 或 1 。若返回值为 0 ，则表明想要删除的元素并不在容器中。对允许重复关键字的容器，删除元素的数量可能大于 1 。

### map 的下标操作和元素访问操作

#### 下标操作

map 和 unordered_map 容器提供了下标运算符和一个对应的 `at` 函数。

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1fwpiihxamnj20vi06s0xa.jpg)

set 类型不支持下标操作，因为 set 中没有与关键字相关联的 “值” ，元素本身就是关键字，因此 “获取与一个关键字相关联的值” 的操作就没意义了。

不能对一个 multimap 或一个 unordered_multimap 进行下标操作，因为这些容器中可能有多个值与一个关键字相关联。

类似于其他下标运算符，map 下标运算符接受一个索引（即：关键字），获取于此关键字相关联的值。但是，如果关键字并不在 map 中，会为它创建一个元素并插入到 map 中，关联值将进行值初始化。

```cpp
    map<string, size_t> word_count;
    // 接受索引为关键字，如果没有就创建一个并插入到 map 中
    word_count["Anna"] = 1;
```

如果下标运算符可能插入一个新元素，我们只可以对非 const 的 map 使用下标操作。

通常情况下，解引用一个迭代器所返回的类型与下标运算符返回的类型是一样的。但对于 map 则不然：当对一个 map 进行下标操作时，会获得一个 `mapped_type` 对象；当解引用一个 map 迭代器时，会得到一个 `valut_type` 对象。


与其他下标运算符相同的是，map 的下标运算符返回一个左值。由于返回的是一个左值，所以我们既可以读也可以写元素。


#### 元素访问

关联容器提供多种查找一个指定元素的方法。

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1fwpj7wig23j20yu04kn11.jpg)

![](https://image.glumes.com/images/2019/04/27/bc32fd77gy1fwpj82iljyj20yw0ai44z.jpg)


应该使用哪个操作依赖于我们要解决什么问题：

*	如果关心的是一个特定元素是否已在容器中，可能 `find` 是最佳选择。
*	对于不允许重复关键字的容器，可能使用 `find` 还是 `count` 没什么区别。
*	对于允许重复关键字的容器，`count` 还会统计有多少个元素有相同的关键字，如果不需要，最好使用 `find` 。


对 map 和 unordered_map 类型，下标运算符提供了最简单的提取元素的方法。但是，使用下标有一个严重的副作用：如果关键字还未在 map 中，下标操作会插入一个具有给定关键字的元素。这种行为是否正确完全依赖于我们的预期是是什么。

但有时，如果只是想知道一个给定关键字是否在 map 中，而不想改变 map ，那就不能使用下标运算符来检查一个元素是否存在，而是该使用 find 操作符。

如果要在允许重复关键字的容器中查找元素，在容器中可能有很多元素具有给定的关键字。如果一个 multimap 或 mulitset 中有多个元素具有给定关键字，则这些元素在容器中会相邻存储。

代码示例如下：

```cpp
    string search_item("Alain de Botton");
    // 返回关键字等于 search_item 的数量
    auto entries = authors.count(search_item);
    // 返回一个迭代器
    auto iter = authors.find(search_item);
    while (entries) {
        // pair 迭代器的 second 代表值
        cout << iter->second << endl;
        ++iter;
        --entries;
    }
```

还可以使用 `lower_bound` 和 `upper_bound` 来实现上述代码，这两个操作都接受一个关键字，返回一个迭代器。

如果关键字在容器中，`lower_bound` 返回的迭代器将指向第一个具有给定关键字的元素，而 `upper_bound` 返回的迭代器则指向最后一个匹配给定关键字的元素之后的位置，如果元素不在 multimap 中，则 lower_bound 和 upper_bound 会返回相等的迭代器，指向一个不影响排序的关键字插入位置。

因此用相同的关键字调用 lower_bound 和 upper_bound 会得到一个迭代器范围，表示所有具有该关键字的元素的范围。

```cpp
    string search_item("Alain de Botton");
    for (auto beg = authors.lower_bound(search_item),
                 end = authors.upper_bound(search_item);
         beg != end;
         ++beg
            ) {
        cout << beg->second << endl;
    }
```

另外还可以使用 `equal_range` 函数来实现。此函数接受一个关键字，返回一个迭代器 pair 。若关键字存在，则第一个迭代器指向第一个与关键字匹配的元素，第二个迭代器指向最后一个匹配元素之后的位置。若未找到匹配元素，则两个迭代器都指向关键字可以插入的位置。


```cpp
    string search_item("Alain de Botton");
    for (auto pos = authors.equal_range(search_item);
         pos.first != pos.second; ++pos.first) {
        cout << pos.first->second << endl;
    }
```

`equal_range` 返回的 pair 的 first 成员保存的迭代器与 lower_bound 返回的迭代器是一样的，second 保存的迭代器与 upper_bound 的返回值是一样的。因此在程序中，pos.first 等于上面的 beg ，pos.second 等价于 end 。

下面是一个单词转换的代码总结 map 的使用：

通过读取两个文件，一个是单词转换文件，一个是要转换的文本文件来实现。

```cpp

/**
 * 读取转换规则文件，保存每个单词到其转换内容的映射
 * @param map_file
 * @return
 */
map<string, string> buildMap(ifstream &map_file) {
    map<string, string> trans_map; // 保存转换规则
    string key;
    string value;
    // 读取第一个单词存入 key 中，行中剩余内容存入 value
    while (map_file >> key && getline(map_file, value)) {
        if (value.size() > 1) {
            trans_map[key] = value.substr(1);
        } else {
            throw runtime_error("no rule for " + key);
        }
    }
    return trans_map;
}

/**
 * 接受一个 string ，如果存在转换规则，则返回转换后的内容
 * @param s
 * @param m
 * @return
 */
const string &transfrom(const string &s, const map<string, string> &m) {
    auto map_it = m.find(s);
    // 如果单词在转换规则 map 中
    if (map_it != m.cend()) {
        // 使用替换短语
        return map_it->second;
    } else {
        return s;
    }
}

/**
 * 读取要转换的文本文件，
 * @param map_file
 * @param input
 */
void word_transform(ifstream &map_file, ifstream &input) {
    auto trans_map = buildMap(map_file); // 保存转换规则
    string text;
    while (getline(input, text)) {
        istringstream stream(text);
        string word;
        bool firstword = true;
        while (stream >> word) {
            if (firstword) {
                firstword = false;
            } else {
                cout << " ";
                cout << transfrom(word, trans_map);
            }
        }
        cout << endl;
    }
}
```

## 关联容器小结

关联容器支持通过关键字高效查找和提取元素。对关键字的使用将关联容器和顺序容器区分开，顺序容器是通过位置访问元素的。

标准库定义了 8 个关联容器，每个容器：

*	是一个 map 或是一个 set 。map 保存 关键字-值 对；set 只保存关键字。
*	要求关键字唯一或不要求。s
*	保持关键字有序或不保证有序。

有序容器使用比较函数来比较关键字，从而将元素按顺序存储。默认情况下，比较操作是采用关键字类型的 `<` 运算符。无序容器使用关键字类型的 `==` 运算符和一个 `hash<key_type>` 类型的对象来组织元素。

允许重复关键字的容器的名字都包含 multi 。而使用哈希技术的容器的名字都以 `unordered` 开头。例如：set 是一个有序集合，其中每个关键字只可以出现一次；unordered_multiset 则是一个无序的关键字集合，其中关键字可以出现多次。

关联容器和顺序容器有很多共同的元素。但是，关联容器定义了一些新操作，并对一些和顺序容器和关联容器都支持的操作重新定义了含义或返回类型。操作的不同反映出关联容器使用关键字的特点。

有序容器的迭代器通过关键字有序访问容器中的元素。无论在有序容器中还是在无序容器中，具有相同关键字的元素都是相邻存储的。

## 小结

标准容器库使用还是挺频繁的，能用标准库的时候就尽量用，包括刷题写代码之类的，《C++ Primer》要多看几遍。

