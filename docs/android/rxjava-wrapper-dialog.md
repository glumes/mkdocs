---
title: "用 Rxjava 封装 Dialog 以及 RxBinding 实现简要分析"
date: 2017-12-22T16:00:05+08:00
categories: ["Android"]
tags: ["Android","RxJava"]
toc: true
original: true
addwechat: true
 
slug: "rxjava-wrapper-dialog"
---
 

之前有写过一篇文章：[用 RxJava 封装回调方法 CallBack](http://www.glumes.com/wrapper-callback-by-rxjava/)。

RxJava 封装回调方法的大体思路就是：使用 Observable 的 create 方法来返回一个 Observable，在 create 方法内给事物设置回调接口，用 Observable 的 onNext 方法来接受回调接口所产生的内容。

这样一来，通过 onNext 方法就把事物的回调方法转换到 Rxjava 对应的事件流里面了，再可以通过其他操作符，如 Map、FlatMap 等对事件流进行相应的转换。

<!--more-->

下来展示具体代码的操作，用的是 RxJava 2.x 的版本：

``` java
 private Observable<Integer> handleRxJavaDialog() {
        return Observable.create(emitter -> {
            final AlertDialog dialog = new AlertDialog.Builder(mContext)
                    .setMessage("RxJava封装Dialog事件流")
                    .setNegativeButton("取消", (dialog12, which) -> emitter.onNext(which)) 
                    .setPositiveButton("确认", (dialog1, which) -> emitter.onComplete())
                    .create();
            dialog.show();
            // 当被取消订阅，unsubscribed 时调用的方法
            emitter.setCancellable(() -> {
                if (dialog.isShowing()) {
                    dialog.dismiss();
                }
            });
        });
    }
```

如此封装之后，直接通过 RxJava 的 onNext 方法将事件封装到 RxJava 事件流里面去。

弄到这，想起了一个开源的 RxJava 系列框架：[RxBinding](https://github.com/JakeWharton/RxBinding)，和本文的封装有相同之处。

## RxBinding 实现简要分析

RxBinding 是用来处理控件异步调用的一个框架，它也已经更新到了 RxJava 2.x 的版本。

在它的源码里对许多控件都进行了封装，可以看到如：`RxView`、`RxTextView`、`RxSeekBar`、`RxProgressBar`、`RxPopupMenu`等等。

它们的使用也是大同小异，比如要对一个 View 封装其点击事件，就可以这样写：
``` java
  RxView.clicks(mBtn)
		.subscribe(new Consumer<Object>() {
            @Override
            public void accept(@NonNull Object o) throws Exception {
                Timber.d("RxView");
            }
        });
```
在代码的前半部分`RxView.clicks(mBtn)`返回的正好是一个 Observable 对象，然后就可以像平时使用 RxJava 一样添加各种操作符，最后通过 subscribe 进行订阅了。

在 RxView.clicks 的源码中，最后返回的是一个 ViewClickObservable，从名字可见，这个 Observable 专门是用来处理点击事件的，在 RxBinding 中还有很多专门处理其他事件的 Observable，例如：`ViewDragObservable`、`ViewLongClickObservable`、`ViewScrollChangeEventObservable`等等，他们的实现方法也都大同小异了。

``` java
  @CheckResult @NonNull
  public static Observable<Object> clicks(@NonNull View view) {
    checkNotNull(view, "view == null");
    return new ViewClickObservable(view);
  }
```

在专门处理各种事件的 Observable 类中，都会有一个实现 `Disposable`接口和具体事件所对应的接口的 Listener 类，而这个 Listener 类的作用，就是给我们所需要的 View 设置对应事件回调接口的，并且在回调接口中调用 Observable 的 onNext 方法来将事件转换到 RxJava 的事件流中。

``` java
	ViewClickObservable(View view) {
    this.view = view;
	}

  @Override protected void subscribeActual(Observer<? super Object> observer) {
    if (!checkMainThread(observer)) {
      return;
    }
    Listener listener = new Listener(view, observer);
	// onSubscribe 设置了 Disposable 接口，当取消订阅时该执行的操作。
    observer.onSubscribe(listener);
    view.setOnClickListener(listener);
  }
// ViewClickObservable 的 Listener 类实现了 OnClickListener 接口
  static final class Listener extends MainThreadDisposable implements OnClickListener {
    private final View view;
    private final Observer<? super Object> observer;

    Listener(View view, Observer<? super Object> observer) {
      this.view = view;
      this.observer = observer;
    }
	// 在 onClick 方法内，调用 onNext 方法，将 onClick 的事件转移到了 RxJava 的事件流里面去了
    @Override public void onClick(View v) {
      if (!isDisposed()) {
        observer.onNext(Notification.INSTANCE);
      }
    }
    // 当被取消订阅时，就设置接口为 null.
    @Override protected void onDispose() {
      view.setOnClickListener(null);
    }
  }
``` 
而在开始订阅时，就调用`subscribeActual`方法来给 View 设置回调接口以便转换事件流。并且还设置了`Disposable`以便响应当结束订阅时的操作。

这一逻辑和本文上半部分将的有相同之处，核心都是用 onNext 方法转换事件流。

而 RxBinding 则封装的更多了，没有在 create 方法里面进行操作， 而是针对 View 控件不同的事件来生成对应的 Observable 对象，对象里面持有 View 控件的实例，并且也完成了给 View 控件设置回调接口以及取消订阅时的方法。

但毫无疑问，针对 View 控件的事件来生成 Observable 会使得代码的兼容性更强了吧，毕竟大多数控件都有公共事件，而只需要针对这个功能事件写一遍代码就行了，不需要针对每个控件来实现一套代码了。

总结经验，写代码时还是要多想想，谋定而后动！！！

## 参考

1、http://adelnizamutdinov.github.io/blog/2014/11/23/advanced-rxjava-on-android-popupmenus-and-dialogs/



