---
title: "Kotlin 中的 run、let、with、apply、also、takeIf、takeUnless 语法糖使用和原理分析"
date: 2018-06-29T20:31:53+08:00
subtitle: ""
draft: false
categories: ["code"]
tags: ["Android","Kotlin"]
toc: true
original: true
addwechat: true
 
slug: "kotlin-syntactic-sugar-usage-and-analysis"
---


在 Kotlin 有一些可以简化代码的语法糖，比如 run、let、with、apply、also、takeIf、takeUnless  等。

再不明白这些语法糖的情况下去看 Kotlin 代码就会一脸懵逼，可当明白之后就会觉得原来可以这样简化。

<!--more-->

## 带接收者的函数字面值

使用这些语法糖之前回顾一下 Kotlin 的函数式编程，在分析 [Kotlin 使用 Anko 构建布局](https://glumes.com/post/android/kotlin-functional-programming-whth-anko/) 文章中有提到 **带接收者的函数字面值**。

它的形式是这样的：

```kotlin
// 定义一个类
class ReceiveObject 
// 定义一个函数
fun exec(invoke: ReceiveObject.()-> Int){}
```

在 Kotlin 中，函数也可以当做变量传参，例如：

```kotlin
fun funAsArg(args:()->Int){}
// 调用
funAsArg { 2 }
```

**args** 是变量名，它的类型就是函数，函数形式在变量名后面约定：`()->Int`，函数没有参数，但是会返回一个 Int 类型的值。

而带接收者的函数字面值，就是在作为传入参数的函数变量的具体函数形式的参数前面多了接收者对象，简单说就是在 `()`前面多了一个点和一个对象，成了如下的形式：

```kotlin
fun exec(invoke: ReceiveObject.()-> Int){}
```

就是这多了的一个点和一个对象，让它有了不一样的功能。

简单的说，invoke 变量是一个函数作为变量，需要传递一个具体函数实现作为形参给 invoke，那么在具体函数实现里面就可以调用接收者对象 ReceiveObject 的相关方法，如下：

```kotlin
	// 接收者对象，有个 show 方法
	class ReceiveObject{
	    fun show(){
	        println("call")
	    }
	}
	// 具体函数实现
	val invoke: ReceiveObject.() -> Int = {
        this.show() // 用 this 指代 接收者对象 ReceiveObject
        2
    }
    exec(invoke)
```

如上，在 invoke 方法里面使用 `this` 指代 ReceiveObject 对象，可以调用它的方法。

而 invoke 变量是作为参数传递给 exec 函数的，如果 exec 函数为空，那么 inkoke 具体实现的 show 方法也不会被调用的，在 exec 中调用 invoke 的方法如下：

```kotlin
fun exec(invoke: ReceiveObject.() -> Int){
    val receObj = ReceiveObject()
    // 两种调用形式
	// 类似于 ReceiceObject 拓展函数一样的调用
    receObj.invoke() 
    // 把 ReceiceObject 作为参数传递给 invoke 调用
    invoke(receObj)
}
```

在 exec 的具体调用中，我们需要构造一个 ReceiveObject 对象实例，不然怎么去调用它的 show 方法呢。


在上面的例子中，还需要构造一个指定的接收者对象实例才能完成 invoke 的调用，而 Kotlin 的语法糖中还有一种叫做 `拓展函数`。

## 拓展函数

拓展函数相当于给某个类添加函数，但这个函数并不属于这个类的函数，和 static 方法是两码事。

```kotlin
	fun Context.showToast(msg: String) {
	    Toast.makeText(this, msg, Toast.LENGTH_SHORT).show()
	}
```

在拓展函数中，使用 this 指代被拓展的类实例，上面代码中 this 指代就是 Context 。

有了 拓展函数和带接收者的函数字面值，就可以实现文章标题提到的那些语法糖了。

例如，针对 ReceiveObject 对象添加它的拓展函数，拓展函数的参数又是一个函数，函数是带接收者的函数字面值，这个接收者对象就是 ReceiveObject 对象它本身，这样调用 invoke 方法就不用再构造 ReceiveObject 对象了。

```kotlin
fun ReceiveObject.exec(invoke: ReceiveObject.() -> Int){
    invoke()
}
```

### 语法糖

下面介绍的语法糖都是位于 Kotlin Standard.kt 文件中的。

### run 语法糖

run 的语法糖有两种：

```kotlin
public inline fun <R> run(block: () -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}
```

这种语法糖传递的参数就仅仅是一个函数，不是带接收者对象的函数字面值，它的返回结果就是 block 函数调用后的结果。

调用示例：

```kotlin
    var result = kotlin.run { 
            "value"
        }
```

相对于给 arg 变量赋值为 value 字符串。

run 的另一种语法糖：

```kotlin
public inline fun <T, R> T.run(block: T.() -> R): R {
    contract {
        callsInPlace(block, InvocationKind.EXACTLY_ONCE)
    }
    return block()
}
```

首先，这个语法糖是一个拓展函数，而且用到了泛型 `<T,R>`，T 类型的拓展函数，返回的是 R 类型，T 和 R 可以相同。

其次，传递的参数是带接收者对象的函数字面值，也就是说可以在 block 函数里面调用 T 的相关方法，通过 this 来指代 T ，在 run 方法内部就是调用了 block 方法，返回 block 函数调用后的结果。

调用示例：

```kotlin
            val result = "a".run {
                this.plus("b")
            }
```

### Contracts DSL

在 run 的语法糖里面还出现了如下一段代码：

```kotlin
 contract {
      callsInPlace(block, InvocationKind.EXACTLY_ONCE)
   }
```

Google 了一番之后

*	https://discuss.kotlinlang.org/t/status-of-kotlin-internal-contracts/6392/2
*	https://stackoverflow.com/questions/49729037/how-does-kotlin-internal-contracts-contractbuilderktcontract-work-in-kotlin
*	https://aisia.moe/2018/03/25/kotlin-contracts-dsl/

得出原来这是 Kotlin 1.2.x 版本中出现的，但实际并没有用，是 Kotlin 后续发展用来解决如下代码问题的：

```kotlin
if (!x.isNullOrEmpty()) { 
  // we know that 'x != null' here 
  println(x.length)
}
```

假设 x 是可以为 null 的，经过 isNullOrEmpty 函数判断之后，再执行 println 函数，那么它肯定就不是 null 了，就不需要再加两个 `!!` 来表示 x 不为 null 了，而现在的情况是要添加 `!!` 。

从 Google 来的信息得知， contract 这段代码就是为了这样的问题的。

由于语法糖都有那样一段代码，所以就先把它们去掉了。

### let 语法糖

```kotlin
public inline fun <T, R> T.let(block: (T) -> R): R {
    return block(this)
}
```

let 语法糖传递的参数是一个函数，不是带接收者的函数字面值，但 block 函数的参数就是 T 类型，所以可以在 block 里面调用 T 类型的方法，但不能通过 this 来指代 T 了，通过 it 来指代 T 类型。

调用示例：

```kotlin
            val result = "a".let {
                it.plus("b")
            }
```


### with 语法糖

```kotlin
public inline fun <T, R> with(receiver: T, block: T.() -> R): R {
    return receiver.block()
}
```

with 语法糖不再是一个拓展函数了，而是需要在语法糖的第一个参数里面传入接收者对象的实例，第二个参数就是带接收者的函数字面值实例，返回的也是 block 调用的结果，这一点和 run 语法糖类似。

调用示例：

```kotlin
            val result = with("a") {
                this.plus("b")
            }
```

### apply 语法糖

```kotlin
public inline fun <T> T.apply(block: T.() -> Unit): T {
    block()
    return this
}
```

apply 语法糖和 run 语法糖都类似，只不过它返回的不是 block 函数调用的结果，而是返回调用者本身，返回 T 类型。



### also 语法糖

```kotlin
public inline fun <T> T.also(block: (T) -> Unit): T {
    block(this)
    return this
}
```

also 语法糖和 let 语法糖有点类似，只不过返回的结果不是 block 调用结果，而是返回它本身，返回 T 类型。

调用示例：

```kotlin
            var result = "a".also {
                it.plus("b")
            }
```

### takeIf 语法糖

```kotlin
public inline fun <T> T.takeIf(predicate: (T) -> Boolean): T? {
    return if (predicate(this)) this else null
}
```

takeIf 语法糖会调用 predicate 函数进行判断，如果为 true 就返回它本身，否则返回 null 。

### takeUnless 语法糖

```kotlin
public inline fun <T> T.takeUnless(predicate: (T) -> Boolean): T? {
    return if (!predicate(this)) this else null
}
```

takeUnless 和 takeIf 正好相反，如果 predicate 返回 false 就返回它本身，否则返回 null 。


## 总结

这么多的语法糖，其实他们的原理都是类似的，共同点在于都是有返回值的，而区别就在于对原有的值进行了哪些操作，然后如何返回最终的值。

最后，光是了解他们的原理和调用情况还是不够的，再不影响代码阅读的情况下要把它们引入到我们的代码中去，灵活地使用它们。

