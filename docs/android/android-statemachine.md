---
title: "Android StateMachine 状态机分析"
date: 2017-12-22T15:37:04+08:00
categories: ["Android"]
tags: ["Android"]
toc: true
original: true
addwechat: true
 
slug: "android-statemachine"
---


之前就有写过一篇文章来学习状态机：[状态机学习](http://www.glumes.com/statemachine_learn/)。

在之后的工作中多次用到了 StateMachine 状态机，简单记录其原理。

<!--more-->

StateMachine 类位于 Android 源码中的，路径是`frameworks/base/core/java/com/android/internal/util/StateMachine.java`，从路径名可以看出，这是一个工具类，该类对状态机进行了封装，方便使用。

在安卓源码中进行搜索，会发现很多类用到了状态机，比如：`WifiStateMachine`、`A2dpStateMachine`、`NsdStateMachine`、`P2dStateMachine`等等，它们都直接或间接的继承了 StateMachine 类。可见，状态机的思想在安卓源码中得到了广泛使用。


在我们的应用开发中也可以使用源码中的`StateMachine`类，只要从源码中把`StateMachine`和`State`类拷贝到我们的工程目录就可以使用了，对于其 import 的一些类，在我们的开发环境中也全都有，不用担心。


如果你对状态机有那么一丝丝的了解，那么学习 StateMachine 类最好的资料就是它的注释了，解释的一清二楚。

## 简单使用阐述

Android 中的状态机是一个分层的消息处理机制，每一层都会有一到多个节点，而状态机的消息就是在这些节点之间流转处理，如下结构所示：

``` sh
	// 状态机分层结构
          mP0
         /   \
        mP1   mS0
       /   \
      mS2   mS1
     /  \    \
    mS3  mS4  mS5  ---> initial state 初始节点
```

而节点就是`State`类，它实现了`IState`接口，除了`enter`、`exit`方法外，还有`processMessage`方法，表示用来处理节点的消息。若返回`HANDLED`则表示消息处理完成，若返回`NOT_HANDLED`则表示消息没有处理。



## 构造状态机

在我们使用 StateMachine 之前，要构造好所需的状态分层结构。通过`addState`方法来向状态机中添加节点，例如如下的状态结构，mS1 和 mS2 节点有公共的父节点 mP1，同时还有一个孤立的节点 mP2。
``` java
      // 状态分层结构设定
        mP1      mP2
       /   \
      mS2   mS1
      // 构造状态机结构代码
      addState(mP1);
          addState(mS1, mP1);
          addState(mS2, mP1);
      addState(mP2);
```

`addState`方法添加节点时，还能指定其父节点添加。

当我们构造完了状态机时，还需要指定其中一个节点为启动点，消息从启动点开始处理，通过`setInitialState`方法来指定启动点，最后通过`start`方法启动状态机。

## 状态机消息处理

当构造完想要的状态机结构时，就是对状态机内部消息流转的处理了。

当`start`状态机时，状态机的第一个动作就是调用节点的`enter`方法，不过，它调用的是指定的启动点的最远的父节点的`enter`方法，然后再是次一级的父节点的`enter`方法，最后才是启动点的`enter`方法，就如同上面的结构所示，先调用 mP1 点，然后才是 mS1 点的方法，此时 mS1 节点就是状态机的当前对外的点。由此可见，当启动点进入 `enter`状态时，它的父节点，直至最顶层的父节点都进入了`enter`状态了。


状态机中的每个节点都 0 个或 1 个父节点，当子节点不能处理当前消息时，它可以通过返回`NOT_HANDLED`将当前消息传递给其父节点来处理。如果一个消息从未被处理过，那么`unhandledMessage`方法将会被调用给最后一次机会来处理该消息。

除此之外，节点还可以通过`transitionTo`方法将当前节点转移至另外一个新的节点。

``` java
          mP0
         /   \
        mP1   mS0
       /   \
      mS2   mS1
     /  \    \
    mS3  mS4  mS5  ---> initial state
```

例如，当 mS5 处理消息，想要将当前节点转移至 mS4 时，那么它会先找到 mS5 和 mS4 最近的公有父节点 mP1。然后，除了这个最近的公有父节点 mP1 以及它的上层节点外，mS5 进入`enter`状态时启动的那些父节点都会退出调用`exit`方法。最后再由 mP1 节点下的节点调用`enter`方法直到新节点 mS4 调用了`enter`方法。

也就是说，从 mS5 到 mS4 状态的转变，先是 mS5、mS1 调用了`exit`方法，再是 mS2、mS4 调用了`enter`方法，这就是状态机中节点发生状态转移时的调用过程。


除此之外，节点还可以调用`deferMessage`和`sendMessageAtFroneOfQueue`方法。`deferMessage`方法使得消息存储在消息队列中，当状态转移到新节点时才会处理，而`sendMessageAtFrontOfQueue`方法则是将消息放置到消息队列的头部。


当状态机的所有消息都完成时，可以调用`transitionToHaltingState`方法来将状态机处理停止状态。此时，状态机将转移到`HaltingState`停止状态，并调用`halting`方法。随后收到的所有消息都只是会调用`haltedProcessMessage`方法来处理了。

若想要完全的停止状态机，则可以使用`quit`或者`quitNow`方法来处理。

以上就是状态机对于消息处理的过程，长篇的文字说明还是不如代码来的直观，这里就不贴完整的代码了，参考代码的连接如下：[GithubGist 链接地址](https://gist.github.com/glumes/e76ea009843eecb2b1e5cf9d8a38a369)



## 状态机实现原理分析

如果我们在初始化状态机时只是传递了一个名字，而没有传递 Looper 或者 Handler 之类的消息循环，那么状态机默认就是启用其内部的一个线程`HandlerThread`。
``` java
  protected StateMachine(String name) {
        mSmThread = new HandlerThread(name); // 创建 HandlerThread 线程
        mSmThread.start();
        Looper looper = mSmThread.getLooper();
        initStateMachine(name, looper);
    }
    // 将消息循环 Looper 与 Handler 进行绑定
  private void initStateMachine(String name, Looper looper) {
        mName = name;
        mSmHandler = new SmHandler(looper, this);
    }
```
构造状态及时，在其内部开启了一个线程，并将其消息循环 Looer 传递给了 `SmHandler` 对象，而`SmHandler`对象就是状态机中最主要的用来派发消息事件和切换状态的了，它派发的消息都是在 HandlerThread 线程进行处理的。


同时，SmHandler 内部有两个数组，用来保存状态机中的链式状态关系，分别是`mStateStack`和`mTempStateStack`变量。当状态机完成启动时，就会通过上面来个变量来保存节点信息。

而状态机的消息处理，内部也是通过 SmHandler 来处理转发的。

具体的实现，建议参考这篇文章：http://blog.csdn.net/yangwen123/article/details/10591451 讲的实在太详细了，拜读了多遍也不敢说写的能比它更清楚。



## 参考
1、http://blog.csdn.net/yangwen123/article/details/10591451