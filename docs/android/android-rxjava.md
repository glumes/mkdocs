---
title: "安卓异步之RxJava"
date: 2018-01-31T15:26:16+08:00
subtitle: ""
tags: ["Android","RxJava"]
draft: false
categories: ["Android"]
toc: true
original: true
addwechat: true
 
slug: "android-rxjava"
categoriessss: true
author: "glumes"
---


不学习 RxJava 简直太落后了，参照网上的博客以及英文书籍《RxJavaEssentials》，开始了 RxJava 之旅 。

<!--more-->

## 添加 RxJava

RxJava 的 Github 地址是 ： https://github.com/ReactiveX/RxJava

根据 Github 上面的 README 提示，需要在 http://search.maven.org 网站上查找需要的 RxJava 的版本信息，然后添加到 Android Studio 的 App Module 的 gradle 脚本中去。

当前的最新的版本是 1.1.9 ，所以添加：

```gradle
compile 'io.reactivex:rxjava:1.1.9'
```

## 观察者模式

观察者模式是设计模式中一种比较常见的模式了，而 RxJava 也是基于此拓展而来的。

Observable 就是我们观察者模式中的被观察者，而 Observe 就是观察者模式中的观察者。一个被观察者可以持有好几个观察者的引用，一旦被观察者的状态发生改变时，就可以通知观察者执行相应的操作。

![observable-pattern](http://7xqe3m.com1.z0.glb.clouddn.com/blog-observable-pattern.png)

## 基础讲解

在 RxJava 中，主要有四个角色：

*    Observable
*    Observer
*    Subscriber
*    Subjects

其中，Observable 和 Subject 是事件的生产者，而 Observe 和 Subscriber 是事件的消费者。

### 热启动和冷启动

从发送消息的角度来看，有两种不同类型的被观察者，分别是热启动观察者和冷启动被观察者。

* Hot Observable 

热启动被观察者 在它被创建时就开始发送消息了，所以，任何订阅了该消息的观察者会在消息序列的中间某个地方开始观察，而不是从起始位置开始。

* Cold Observable

冷启动被观察者 在至少有一个观察者订阅了它之后才会发送消息，所以，观察者可以保证会从消息序列的起始位置处开始观察。


### 创建被观察者对象 Observable

Observable 类提供了方法来创建 Observable 对象。

#### Observable.create() 

使用 Observable.create() 方法来创建一个 Observable 对象：

```java
/**
 * Observable.create() 方法创建一个被观察者
 */
Observable<Integer> observable = Observable.create(new Observable.OnSubscribe<Integer>() {
    @Override
    public void call(Subscriber<? super Integer> subscriber) {
        for (int i = 0; i < 5; i++) {
            subscriber.onNext(i);
        }
        subscriber.onCompleted();
    }
});
```
其中，create() 方法的参数是一个匿名内部类，也就是创建了 ```OnSubscirbe 接口``` 类型的对象，而 OnSubsribe 接口又是继承 ```Action1``` 接口的，其中的 ```call()``` 方法也是在 Action1 接口中的，查看代码如下所示：

Action1 接口：
```java
/**
 * A one-argument action.
 * @param <T> the first argument type
 */
public interface Action1<T> extends Action {
    void call(T t);
}
```

OnSubscribe 接口
```java
/**
 * Invoked when Observable.subscribe is called.
 * @param <T> the output value type
 */
public interface OnSubscribe<T> extends Action1<Subscriber<? super T>> {
    // cover for generics insanity
}
```

### 创建观察者对象 Observer / Subscriber 
创建了被观察者之后，就可以创建一个观察者，用来消费被观察者产生的事件。
```java
/**
 * 创建一个观察者,Observer 接口类型实现类的对象
 */
Observer<Integer> observer = new Observer<Integer>() {
    @Override
    public void onCompleted() {

    }

    @Override
    public void onError(Throwable e) {

    }

    @Override
    public void onNext(Integer integer) {
        Logger.v(String.valueOf(integer));
    }
};
```

```Observer``` 也是一个接口类型，声明了如上的三个方法：
*    onNext()：Observable 每发送一次事件都会调用一次该方法消费事件。
*    onCompleted()：当 Observabel 的事件发送完毕后，就会调用该方法。
*    onError()：如果 Observable 发送事件的过程中出现了错误，则会调用该方法。

有了被观察者 Observable 和观察者 Observer 之后，就可以进行事件的订阅了。
```java
/**
 * Observable 订阅 Observer
 */
observable.subscribe(observer);
```
调用 Observable 对象的 subscribe 方法即可实现订阅。

就这样，一个简单的并没有什么卵用的被观察者、观察者以及它们之间的订阅关系就已经实现了。

> 在 OnSubscribe 接口的注释中可以看到，当调用了 Observable.subscribe 方法时，OnSubscribe 的接口方法将会被调用，也就是继承的 Action1 的 call 方法。而 call 则回调执行了观察者的 onNext、onCompleted、onError 方法。

通过查看 subscribe 执行的源代码也可以得知，部分源码：
```java
static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
    // new Subscriber so onStart it
    subscriber.onStart();
try {
    // allow the hook to intercept and/or decorate
    RxJavaHooks.onObservableStart(observable, observable.onSubscribe).call(subscriber);
    return RxJavaHooks.onObservableReturn(subscriber);
}
}
```
subscribe 方法接收的参数是 Subscriber 类型的，而且还返回了一个 Subscription 类型的结果。

> 在 subscribe 方法中，调用了 OnSubscribe 的 call 方法，而 call 方法又将 subscriber/observer 作为参数传入，在创建 Observable 对象的 call 方法实现中，调用了 subscriber/observer 的 onNext、onCompleted、onError 方法，这样就形成了一个回调，回调了 subscriber/observer 的方法。

而在执行到最终的 subscribe 方法之前，还有一系列的转换过程，用于**将 Observer 对象转换成 Subscriber 对象**。
```java
public final Subscription subscribe(final Observer<? super T> observer) {
    if (observer instanceof Subscriber) {  // 如果是 Subscriber 实例则直接调用了
        return subscribe((Subscriber<? super T>)observer);
    }
    if (observer == null) {
        throw new NullPointerException("observer is null");
    }
    // 转换成一个包装类
    return subscribe(new ObserverSubscriber<T>(observer)); 
}
```
ObserverSubscriber 是一个 Observer 的包装类，部分代码如下：
```java
/**
 * Wraps an Observer and forwards the onXXX method calls to it.
 * @param <T> the value type
 */
public final class ObserverSubscriber<T> extends Subscriber<T> {
    final Observer<? super T> observer;

    public ObserverSubscriber(Observer<? super T> observer) {
        this.observer = observer;
    }
}
```
所以，在能使用 Subscriber 的地方还是尽量使用 Subscriber 。并且，相比 Observer 对象，Subscriber 还多了 onStart 方法和 unsubscribe 方法，分别用来在 subscribe 方法调用之前做一些准备工作和取消订阅。

### 其他创建被观察者的方法


### Observable.from() 方法创建 Observable 

当我们需要监听的对象是一个列表 List  或者 数组 Array 时，我们还可以使用 Observable.from() 方法来创建一个被观察者。

```java
/**
 * Observable.from() 方法创建 Observable
 */
List<Integer> items = new ArrayList<Integer>();
items.add(1);
items.add(10);
items.add(100);
items.add(1000);
Observable<Integer> integerObservable = Observable.from(items);
```

查看 from() 方法的源码发现，在内部还是调用的 create() 方法来创建的被观察者。

所以，当我们通过 from() 方法来创建一个 Observable 时，就不需要像 create() 方法考虑订阅观察者时的回调了，直接 subscribe 订阅观察者即可。

```java
public static <T> Observable<T> from(Iterable<? extends T> iterable) {
    return create(new OnSubscribeFromIterable<T>(iterable));
}
```

### Observable.just() 方法创建 Observable 

当我们需要监听的是一个 Java 方法时，我们可以使用 just() 方法来将其转化成一个被观察者。
```java
/**
 * Observable.just() 方法将一个函数转化成 Observable
 */
Observable<String> stringObservable = Observable.just(helloWorld());
private String helloWorld(){
    return "Hello World" ;
}
```
当我们创建 Observable 对象时， just() 就会执行需要转化的方法；当订阅观察者时，就会将方法返回的值发送出去。

just 方法能够接收 1~9 个参数，并且会按照它们的参数顺序发送它们。同时，just() 还能接收列表 List 和数组 Array ，但是 just() 方法并不会迭代列表中的每一个值，而是将它们作为一个整体发送出来。

### Observable 对象的 empty() 方法、never() 方法和 thorw() 方法

如果我们想要 Observable 对象不发送任何东西，但是正常的结束，可以使用 empty() 方法来创建被观察者。

我们可以使用 never() 方法来创建一个 Observable 对象，它不会发送任何东西，也不会终止。

我们可以使用 throw() 方法来创建一个 Observable 对象，它不会发送任何东西，但是会抛出异常。


### Subject 对象

Subject 对象也是 RxJava 中四大重要对象之一。

在同一时刻，Subject 既能是 Observable 对象也能是 Observer 对象，它就像一个桥梁，联系着两者。一个 Subject 对象能像 Observer 对象那样订阅被观察者，也能像 Observable 那样去发送消息。

RxJava 提供了四种类型的 subject ：

####    PublishSubject

PublishSubject 是最基本的 Subject 对象。

```java
/**
 * 创建一个 PublishSubject,从订阅后的地方开始接收
 */
PublishSubject<String> stringPublishSubject = PublishSubject.create();
/**
 * 像 Observable 一样的订阅
 */
Subscription subscriptionPrint = stringPublishSubject.subscribe(new Observer<String>() {
    @Override
    public void onCompleted() {
        Logger.e("PublishSubject Observable completed");
    }

    @Override
    public void onError(Throwable e) {

    }

    @Override
    public void onNext(String s) {
        Logger.e(s);
    }
});

/**
 * 像 Observer 一样的执行方法,从此处开始接收订阅
 */
stringPublishSubject.onNext("this is PublishSubject");
```
通过 create() 方法来创建一个 PublishSubject ，它会发送一个 String 类型的值，然后订阅了该 PublishSubject 。

这个时候，还没有任何元素被发送出来，所以我们的观察者也会一直在等待，但是这个等待是无需我们担心的，系统会自动响应的，我们只需关注响应时执行哪些操作就好了。

最后一行代码，则是手动触发了 Observer 的 onNext() ，然后打印字符串，而在 subscribe() 方法之前执行的 onNext() 则不会打印字符串。

> PublishSubject 只有在订阅了之后，才会发送数据。

#### BehaivorSubject 

```java
/**
 * BehaivorSubject 会发送离订阅最近的上一值,如果没有则发送默认值
 */
BehaviorSubject<String> behaviorSubject = BehaviorSubject.create("init item");

behaviorSubject.onNext("1");

/**
 * 订阅前的上一个值,将会被打印出来
 */
behaviorSubject.onNext("2");

behaviorSubject.subscribe(new Action1<String>() {
    @Override
    public void call(String s) {
        Logger.e(s);
    }
});

/**
 * 在此方法之前订阅,则会发送订阅前的上一个值,以及这次订阅的值
 */
behaviorSubject.onNext("3");

behaviorSubject.onCompleted();
```
>    BehaviorSubject 会发送离订阅最近是上一个值，如果没有则发送默认值。

#### ReplaySubject 

```java
/**
 * ReplaySubject 会将所有的订阅都缓存起来,并将它们一并发送给 Observer .
 */
ReplaySubject<String> replaySubject = ReplaySubject.create();
/**
 * 在订阅之前发送的元素,也会发送给观察者
 */
replaySubject.onNext("1");
replaySubject.onNext("2");
/**
 * 执行订阅操作
 */
replaySubject.subscribe(new Action1<String>() {
    @Override
    public void call(String s) {
        Logger.e(s);
    }
});
/**
 * 在订阅之后发送的元素,也会发送给观察者
 */
replaySubject.onNext("3");
replaySubject.onNext("4");
```

> ReplaySubject 会将订阅前后发送的元素缓存起来，并且一并发送给 Observer 。

#### AsyncSubject 

```java
/**
 * AsyncSubject 当 Observable 完成之后,只会向 Observer 发送订阅的最后一个元素
 */

AsyncSubject<String> asyncSubject = AsyncSubject.create();


asyncSubject.subscribe(new Action1<String>() {
    @Override
    public void call(String s) {
        Logger.e(s);
    }
});

/**
 * 1 和 2 将不会被打印出来
 */
asyncSubject.onNext("1");
asyncSubject.onNext("2");
/**
 * 最后一个发送的将会被打印出来
 */
asyncSubject.onNext("this is last item");
/**
 * 此方法必须得有,表明发送完成
 */
asyncSubject.onCompleted();
```

> AsyncSubject 将会发送订阅的最后一个元素 。

## 参考

1. https://gank.io/post/560e15be2dca930e00da1083
2. http://www.jianshu.com/p/1257c8ba7c0c



