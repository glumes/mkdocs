---
title: "Android JNI 调用时的异常处理"
date: 2018-05-16T17:55:52+08:00
subtitle: ""
draft: false
categories: ["Android"]
tags: ["NDK"]
toc: true
original: true
addwechat: true
slug: "android-jni-exception-handle"
 
---


Android JNI 调用时的异常主要有如下两种：

*	Native 代码调用 Java 层代码时发生了异常要处理
*	Native 代码自己抛出了一个异常让 Java 层去处理


<!--more-->

可以看到异常的发生和处理基本都需要 Native 和 Java 交互，而对于 Native 自身出了异常，也就是 C/C++ 代码有问题，导致应用崩溃的又是另一回事了。


## Native 调用 Java 方法时的异常

之前的博客中就讲述了如何从 [Native 调用 Java 的方法](https://glumes.com/post/android/android-jni-access-field-and-method/)，先准备一个有异常的方法供 Native 去调用。

```java
    private int operation() {
        return 2 / 0;
    }
```

然后在 Native 中调用该方法。

```cpp
    jclass cls = env->FindClass("com/glumes/cppso/jnioperations/ExceptionOps");
    jmethodID mid = env->GetMethodID(cls, "<init>", "()V");
    jobject obj = env->NewObject(cls, mid);
    mid = env->GetMethodID(cls, "operation", "()I");
    // 先初始化一个类，然后调用类方法，就如博客中描述的那样
    env->CallIntMethod(obj, mid);
```

显然，除数为 0 ，一调用应用直接崩溃了。

```java
 java.lang.ArithmeticException: divide by zero
```

接下来就是要进行异常处理的部分了。

```cpp
    jclass cls = env->FindClass("com/glumes/cppso/jnioperations/ExceptionOps");
    jmethodID mid = env->GetMethodID(cls, "<init>", "()V");
    jobject obj = env->NewObject(cls, mid);
    mid = env->GetMethodID(cls, "operation", "()I");
// 先初始化一个类，然后调用类方法，就如博客中描述的那样
    env->CallIntMethod(obj, mid);
    //检查是否发生了异常
    jthrowable exc = env->ExceptionOccurred();
//    jboolean result = env->ExceptionCheck();

    if (exc) {
        // 打印异常日志
        env->ExceptionDescribe();
        // 这行代码才是关键不让应用崩溃的代码，
        env->ExceptionClear();
        // 发生异常了要记得释放资源
        env->DeleteLocalRef(cls);
        env->DeleteLocalRef(obj);
    }
```

添加如上代码进行处理后，应用并不会直接崩溃了，并且在 LogCat 中会看到对应的异常日志，这里面到了做了哪些操作呢？

Native 提供了 `ExceptionOccurred` 和 `ExceptionCheck` 方法来检测是否有异常发生，前者返回的是 `jthrowable` 类型，后者返回的是 `jboolean` 类型。

如果有异常，会通过 `ExceptionDescribe` 方法来打印异常信息，方便我们在 LogCat 中看到对应的信息。

而 `ExceptionClear` 方法则是关键的不会让应用直接崩溃的方法，类似于 Java 的 `catch` 捕获异常处理，它会消除这次异常。

这样就把由 Native 调用 Java 时的一个异常进行了处理，当处理完异常之后，别忘了释放对应的资源。

不过，我们这样仅仅是消除了这次异常，还应该让调用者有异常的发生，那么就需要通过 Native 来抛出一个异常告诉 Java 调用者了。

## Native 抛出 Java 中的异常

有时在 Native 代码中进行一些操作，需要抛出异常到 Java ，交由上层去处理。

比如 Java 调用 Native 方法传递了某个参数，而这个参数有问题，那么 Native 就可以抛出异常让 Java 去处理这个参数异常的问题。

Native 抛出异常的代码大致都是相同的，可以抽出一个通用函数来：

```cpp
void throwByName(JNIEnv *env, const char *name, const char *msg) {
    jclass cls = env->FindClass(name);
    if (cls != NULL) {
        env->ThrowNew(cls, msg);
    }
    env->DeleteLocalRef(cls);
}
// 调用抛出异常
extern "C"
JNIEXPORT void JNICALL
Java_com_glumes_cppso_jnioperations_ExceptionOps_nativeThrowException(JNIEnv *env, jobject instance) {
    throwByName(env, "java/lang/IllegalArgumentException", "native throw exception");
}
```

根据异常类型和异常信息来抛出异常。

而在 Java 中，就必须要用 try ... catch... 来准备好捕获这次异常了。

```java
        try {
            nativeThrowException();
        } catch (IllegalArgumentException e) {
            Log.e("NativeMethod", e.getMessage());
        }
```

## 异常处理小结

除了以上两种异常情况之外，还有一个特别常见的异常，就是当判断某个变量为 NULL 之后，执行立即返回的操作。

当发生异常时，一定要先处理异常，然后才能继续执行后面的步骤。如果不是需要立即返回的，那么就通过 `ExceptionClear`清除这次异常，然后在进行其他的处理。

对于在 Native 中发生了异常，需要让 Java 层去处理了，则在 Native 中抛出对应的异常，由 Java 层去捕获，比如在使用 `ExceptionClear` 清除了异常之后，就可以通过 `throwNew` 来抛出异常信息。

具体的异常处理方法和时机还是要看具体的使用场景，选择最合适的处理方法。

具体示例代码可参考我的 Github 项目 [https://github.com/glumes/AndroidDevWithCpp](https://github.com/glumes/AndroidDevWithCpp)，欢迎 Star。

