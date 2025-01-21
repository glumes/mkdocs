---
title: "用 RxJava 封装回调方法 CallBack"
date: 2017-12-22T10:38:34+08:00
categories: ["Android"]
tags: ["Android","RxJava"]
toc: true
original: true
addwechat: true
 
slug: "rxjava-wrapper-callback"
---



在知乎上看到这样一个问题：[RxJava正确的封装callback的方式应该是怎么样的？](https://www.zhihu.com/question/39492234)。虽说已经是个一年前的问题了，自己现在才遇到 (羞愧脸) 。

<!--more-->

最近在处理蓝牙操作时，也想着如何把 RxJava 优势用到蓝牙开发中来。

使用 RxJava 能够简化我们的编程，有效的避免回调地狱 (`Callback Hell`) 的情况。将回调操作交给观察者 Observer 的 onNext 中去处理，同时还有着丰富的操作符进行各项处理。

但是有些现成的操作已经处理好回调方法了，例如蓝牙扫描，只要在 onLeScan 方法中处理返回的蓝牙设备即可，其他方法也大致如此，发出请求，在回调中处理请求。

``` java
mBluetoothAdapter.startLeScan(new BluetoothAdapter.LeScanCallback() {
            @Override
            public void onLeScan(BluetoothDevice bluetoothDevice, int i, byte[] bytes) {
                // 回调方法，处理扫描到的蓝牙设备
            }
        });
```

而现在，要做的就是用 RxJava 对蓝牙扫描过程进行封装，返回 RxJava 中的被观察者 Observable ，然后在再对这个 Observable 使用各种操作符，线程调度，最后执行订阅 subscribe 方法。

## RxJava 创建 Observable 过程

创建一个 Observable 的方法大致是这样的：
``` java
		Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {
                // 在回调方法中，对发射的数据处理 onNext、onComplete、 onError 
                // 实则是调用 subscriber 的方法进行处理
            }
        }).subscribe(new Observer());
```
使用 create 方法将会创建一个 Observable 对象并返回。而 create 方法的参数 OnSubscribe 就是一个回调方法，在执行订阅 subscribe 方法时会回调里面的 call 方法。

而 call 方法里的参数 subscriber 就是我们在 subscribe 方法中传入的观察者 Observer 。RxJava 内部会对传入的 Observer 进行处理，包装成 ObserverSubscriber 对象，继承自 Subscriber ，所以最终都是调用的 Subscriber 类型的方法。

所以假若对回调方法进包装，那么在 call 方法中就应该对回调数据进行处理了。

同时，subscribe 方法最后返回的对象是 Subscription 类型的。我们可以用  Subscription 类型的对象的 unsubscribe 方法来取消订阅。参照 RxJava 源码发现，Subscriber 类型实现了 Subscription 接口，并且最后返回的也是 call 方法中的参数 subscriber 。

``` java
        try {
            // allow the hook to intercept and/or decorate
            // 回调 call 方法
            RxJavaHooks.onObservableStart(observable, observable.onSubscribe).call(subscriber);
            // 返回 Subscription 
            return RxJavaHooks.onObservableReturn(subscriber);
        } catch (Throwable e) { // 省略
        }
```

对于使用 from 方法创建的 Observable ，最后的调用过程也是一致的，RxJava 内部将传入的参数封装成了 OnSubscribeFromIterable 对象，其他的大同小异了。

## RxJava 封装蓝牙扫描过程

对蓝牙扫描的包装：

有了上面的思路，就可以简单的实现包装了，使用 create 方法返回了一个 `Observable<BluetoothDevice>`对象，然后在对其订阅。

``` java
Subscription subscription = Observable.create(new Observable.OnSubscribe<BluetoothDevice>() {
            @Override
            public void call(final Subscriber<? super BluetoothDevice> subscriber) {
                BluetoothAdapter.LeScanCallback leScanCallback =  new BluetoothAdapter.LeScanCallback() {
                    @Override
                    public void onLeScan(BluetoothDevice bluetoothDevice, int i, byte[] bytes) {
                        //
                        subscriber.onNext(bluetoothDevice);
                    }
                } ;
                mBluetoothAdapter.startLeScan(leScanCallback) ;
            }
        }).subscribe(new Observer()) ;
```
简单地说，已经实现了对 BluetoothDevice 对象的封装，可依旧存在问题，我们可以使用 Subscription 来取消订阅，不接收发送数据，但却没有停止蓝牙设备的扫描，因此下一步就是在取消订阅同时终止扫描。

``` java
    Subscription subscription = Observable.create(new Observable.OnSubscribe<BluetoothDevice>() {
            @Override
            public void call(final Subscriber<? super BluetoothDevice> subscriber) {
                final BluetoothAdapter.LeScanCallback leScanCallback =  new BluetoothAdapter.LeScanCallback() {
                    @Override
                    public void onLeScan(BluetoothDevice bluetoothDevice, int i, byte[] bytes) {
                        // 判断是否还在订阅，避免发送不必要的数据
                        if (!subscriber.isUnsubscribed()){
                            subscriber.onNext(bluetoothDevice);
                        }
                    }
                } ;
                mBluetoothAdapter.startLeScan(leScanCallback) ;
                // 使用 Subscriptions 的 create 方法创建一个只有取消订阅时才调用的方法
                subscriber.add(Subscriptions.create(new Action0() {
                    @Override
                    public void call() {
                        mBluetoothAdapter.stopLeScan(leScanCallback);
                    }
                }));
            }
        }).subscribe();
```

Subscriber 类有个方法是 addSubscription ，使用 Subscriptions 类的 create 方法就可以创建一个在取消订阅时才执行的方法。

Subscriptions 类的命名方式有点类似于 Java 的 Collections 命名，工具类后缀加个 s 的方式。

这样就实现了在取消订阅的同时也停止蓝牙扫描过程。


## RxJava 自带处理方式

最后，RxJava 在后续的版本中还提供了其他的方法来针对上述问题，不需要再通过 Subscriber 添加一个 Subscription 来解决了。

``` java
Observable.fromEmitter(new Action1<Emitter<BluetoothDevice>>() {
            @Override
            public void call(final Emitter<BluetoothDevice> bluetoothDeviceEmitter) {
            final BluetoothAdapter.LeScanCallback scanCallback = new BluetoothAdapter.LeScanCallback() {
                    @Override
                    public void onLeScan(BluetoothDevice bluetoothDevice, int i, byte[] bytes) {
                        bluetoothDeviceEmitter.onNext(bluetoothDevice);
                    }
                };
                // 开始扫描
                mBluetoothAdapter.startLeScan(scanCallback) ;
                // 当 unsubscribe 时执行该方法
                bluetoothDeviceEmitter.setCancellation(new Cancellable() {
                    @Override
                    public void cancel() throws Exception {
                        mBluetoothAdapter.stopLeScan(scanCallback);
                    }
                });
            }
        }, Emitter.BackpressureMode.BUFFER); // RxJava 解决背压问题的方式
```

通过 Observable 的 fromEmitter 方法来创建 Observable ，同时通过 Emitter 的 setCancellation 方法设置取消订阅时的动作。

同时，还可以通过 Emitter 的不同的 `BackpressureMode` 来处理背压问题，也就是当生产者的生产速度比消费者的消费速度快的情况，Emmiter 提供了如下方法：

*	BUFFER               缓存
*	LATEST                使用最新的
*	DROP                   直接丢弃
*	ERROR/NONE      抛出异常 MissingBackpressureException 

如此一来就可以使用 RxJava 对蓝牙扫描过程进行封装了。

用 RxJava 封装其他回调方法也大致如此了，把原本的回调方法处理用 onNext 来处理，异常用 onError 处理，需要取消回调的就在取消订阅时处理。

## 参考
1、http://ryanharter.com/blog/2015/07/07/wrapping-existing-libraries-with-rxjava/
2、http://blog.chengyunfeng.com/?p=1019