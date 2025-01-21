---
title: "Android JNI 中的线程操作"
date: 2018-07-15T14:12:25+08:00
subtitle: ""
draft: false
categories: ["Android"]
tags: ["NDK"]
toc: true
original: true
addwechat: true
 
slug: "android-jni-thread-operation"
---


学习一下如何在 Native 代码中使用线程。

<!--more-->

Native 中支持的线程标准是 POSIX 线程，它定义了一套创建和操作线程的 API 。

我们可以在 Native 代码中使用 POSIX 线程，就相当于使用一个库一样，首先需要包含这个库的头文件：

```cpp
#include <pthread.h>
```

这个头文件中定义了很多和线程相关的函数，这里就暂时使用到了其中部分内容。

## 创建线程

POSIX 创建线程的函数如下：

```cpp
int pthread_create(
	pthread_t* __pthread_ptr, 
	pthread_attr_t const* __attr, 
	void* (*__start_routine)(void*), 
	void* arg);
```

它的参数对应如下：

*	__pthread_ptr 为指向 pthread_t 类型变量的指针，用它代表返回线程的句柄。

*	__attr 为指向 pthread_attr_t 结构的指针，可以通过该结构来指定新线程的一些属性，比如栈大小、调度优先级等，具体看 pthread_attr_t 结构的内容。如果没有特殊要求，可使用默认值，把该变量取值为 NULL 。

*	第三个参数为该线程启动程序的函数指针，也就是线程启动时要执行的那个方法，类似于 Java Runnable 中的 run 方法，它的函数签名格式如下：
```cpp
void* start_routine(void* args)
```

启动程序将线程参数看成 void 指针，返回 void 指针类型结果。

*	第四个参数为线程启动程序的参数，也就是函数的参数，如果不需要传递参数，它可以为 NULL 。

`pthread_create` 函数如果执行成功了则返回 0 ，如果返回其他错误代码。

接下来，我们可以体验一下 pthread_create 方法创建线程。


首先，创建线程启动时运行的函数：
```cpp
void *printThreadHello(void *);
void *printThreadHello(void *) {
    LOGD("hello thread");
    // 切记要有返回值
    return NULL;
}
```

要注意线程启动函数是要有返回值的，没有返回值就直接崩溃了。

另外这个函数一旦 pthread_create 调用了，它就会立即执行。

接下来创建线程：
```cpp
JNIEXPORT void JNICALL
Java_com_glumes_cppso_jnioperations_ThreadOps_createNativeThread(JNIEnv *, jobject) {
    pthread_t handles; // 线程句柄
    int result = pthread_create(&handles, NULL, printThreadHello, NULL);
    if (result != 0) {
        LOGD("create thread failed");
    } else {
        LOGD("create thread success");
    }
}
```

由于没有给该线程设置属性，并且线程运行函数也不需要参数，就都直接设置为了 NULL，那么上面那段程序就可以执行了，并且 printThreadHello 函数是运行在新的线程中的。

## 将线程附着在 Java 虚拟机上 

在上面的线程启动函数中，只是简单的执行了打印 log 的操作，如果想要执行和 Java 相关的操作，比如从 JNI 调用 Java 的函数等等，那就需要用到 Java 虚拟机环境了，也就是用到 JNIEnv 指针，毕竟所有的调用函数都是以它开头的。


pthread_create 创建的线程是一个 C++ 中的线程，虚拟机并不能识别它们，为了和 Java 空间交互，需要先把 POSIX 线程附着到 Java 虚拟机上，然后就可以获得当前线程的 JNIEnv 指针，因为 JNIEnv 指针只是在当前线程中有效。

通过 `AttachCurrentThread` 方法可以将当前线程附着到 Java 虚拟机上，并且可以获得 JNIEnv 指针。

`AttachCurrentThread` 方法是由 JavaVM 指针调用的，它代表的是 Java 虚拟机接口指针，可以在 JNI_OnLoad 加载时来获得，通过全局变量保存起来

```cpp
static JavaVM *gVm = NULL;
JNIEXPORT int JNICALL JNI_OnLoad(JavaVM *vm, void *reserved) {
    JNIEnv *env;
    if (vm->GetEnv((void **) &env, JNI_VERSION_1_6) != JNI_OK) {
        return JNI_ERR;
    }
    gVm = vm;
    return JNI_VERSION_1_6;
}
```

当通过 `AttachCurrentThread` 方法将线程附着当 Java 虚拟机上后，还需要将该线程从 Java 虚拟机上分离，通过 `DetachCurrentThread` 方法，这两个方法是要同时使用的，否则会带来 BUG 。


具体使用如下：

首先在 Java 中定义在 C++ 线程中回调的方法，主要就是打印线程名字：

```java
    private void printThreadName() {
        LogUtil.Companion.d("print thread name current thread name is " + Thread.currentThread().getName());
            Thread.sleep(5000);
    }
```

然后定义 Native 线程中运行的方法：
```cpp
void *run(void *);
void *run(void *args) {
    JNIEnv *env = NULL;
    // 将当前线程添加到 Java 虚拟机上
    // 假设有参数传递
    ThreadRunArgs *threadRunArgs = (ThreadRunArgs *) args;
    if (gVm->AttachCurrentThread(&env, NULL) == 0) {
    // 回调 Java 的方法，打印当前线程 id ，发现不是主线程就对了
        env->CallVoidMethod(gObj, printThreadName);
        // 从 Java 虚拟机上分离当前线程
        gVm->DetachCurrentThread();  
    }
    return (void *) threadRunArgs->result;
}
```

最后创建线程并运行方法：
```cpp
		// 创建传递的参数
	    ThreadRunArgs *threadRunArgs = new ThreadRunArgs();
        threadRunArgs->id = i;
        threadRunArgs->result = i * i;
        // 运行线程
        int result = pthread_create(&handles, NULL, run, (void *) threadRunArgs);
```


通过这样的调用，就可以在 Native 线程中调用 Java 相关的函数了。

## 等待线程返回结果

前面提到在线程运行函数中必须要有返回值，最开始只是返回了一个空指针 NULL ，并且在某个方法里面开启了新线程，新线程运行后，该方法也就立即返回退出，执行完了。

现在，还可以在该方法里等待线程执行完毕后，拿到线程执行完的结果之后再推出。

通过 `pthread_join` 方法可以等待线程终止。

```cpp
int pthread_join(pthread_t __pthread, void** __return_value_ptr);
```

其中：

*	__pthread 代表创建线程的句柄
*	__return_value_ptr 代表线程运行函数返回的结果


使用如下：

```cpp
	for (int i = 0; i < num; ++i) {
        pthread_t pthread;
        // 创建线程，
        int result = pthread_create(&handles[i], NULL, run, (void *) threadRunArgs);
        }
    }
    for (int i = 0; i < num; ++i) {
        void *result = NULL; // 线程执行返回结果
        // 等待线程执行结束
        if (pthread_join(handles[i], &result) != 0) {
            throwByName(env, runtimeException, "Unable to join thread");
        } else {
	        LOGD("return value is %d",result);
        }
    }
```

如果 pthread_join 返回为 0 代表执行成功，非 0 则执行失败。


具体的代码示例可以参考我的 Github 项目，欢迎 Star 。

> [https://github.com/glumes/AndroidDevWithCpp](https://github.com/glumes/AndroidDevWithCpp)

参考

1. 《Android C++ 高级编程--使用 NDK》

