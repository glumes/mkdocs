---
title: "Android JNI 中的引用管理"
date: 2018-05-16T17:50:06+08:00
subtitle: ""
draft: false
categories: ["Android"]
tags: ["NDK"]
toc: true
original: true
 
addwechat: true
slug: "android-jni-reference-manage-rules"
---

在 Native 代码中有时候会接收 Java 传入的引用类型参数，有时候也会通过 NewObject 方法来创建一个 Java 的引用类型变量。

在编写 Native 代码时，要注意这个代表 Java 数据结构类型的引用在使用时会被 GC 回收的可能性。

<!--more-->


Native 代码并不能直接通过引用来访问其内部的数据接口，必须要通过调用 JNI 接口来间接操作这些引用对象，就如在之前的系列文章中写的那样。并且 JNI 还提供了和 Java 相对应的引用类型，因此，我们就需要通过管理好这些引用来管理 Java 对象，避免在使用时被 GC 回收了。



## 引用简介

JNI 提供了三种引用类型：

*	局部引用
*	全局引用
*	弱全局引用

## 局部引用

局部引用是最常见的一种引用。绝大多数 JNI 函数创建的都是局部引用，比如：NewObject、FindClass、NewObjectArray 函数等等。

> 局部引用会阻止 GC 回收所引用的对象，同时，它不能在本地函数中跨函数传递，不能跨线程使用。

局部引用在 Native 函数返回后，所引用的对象会被 GC 自动回收，也可以通过 DeleteLocalRef 函数来手动回收。

在之前文章 JNI 调用时缓存字段和方法 ID，第一种方法采用的是使用时缓存，把字段 ID 通过 static 变量缓存起来。

如果把 FindClass 函数创建的局部引用也通过 static 变量缓存起来，那么在函数退出后，局部引用被自动释放了，static 静态变量中存储的就是一个被释放后的内存地址，成为了一个野指针，再次调用时就会引起程序崩溃了。

```cpp
extern "C"
JNIEXPORT jstring JNICALL
Java_com_glumes_cppso_jnioperations_LocalAndGlobalReferenceOps_errorCacheUseLocalReference(
        JNIEnv *env, jobject instance) {
    static jmethodID mid = NULL;
    static jclass cls;
    // 局部引用不能使用 static 来缓存，否则函数退出后，自动释放，成为野指针，程序 Crash
    if (cls == NULL) {
        cls = env->FindClass("java/lang/String");
        if (cls == NULL) {
            return NULL;
        }
    } else {
        LOGD("cls is not null but program will crash");
    }
    if (mid == NULL) {
        mid = env->GetMethodID(cls, "<init>", "([C)V");
        if (mid == NULL) {
            return NULL;
        }
    }
    jcharArray charEleArr = env->NewCharArray(10);
    const jchar *j_char = env->GetStringChars(env->NewStringUTF("LocalReference"), NULL);
    env->SetCharArrayRegion(charEleArr, 0, 10, j_char);
    jstring result = (jstring) env->NewObject(cls, mid, charEleArr);
    env->DeleteLocalRef(charEleArr);
    return result;
}
```

#### 局部引用手动释放时机

局部引用除了自动释放外，还可以通过 DeleteLocalRef 函数手动释放，它一般存在于以下场景中：

*	当要创建大量的局部引用对象时，会造成 JNI 局部引用表的溢出。

假如有以下代码，则要特别注意，及时释放局部引用，防止溢出。

```cpp
    for (int i = 0; i < len; ++i) {
        jstring jstr = (*env)->GetObjectArrayElement(env, arr, i);
        ... /* process jstr */
        (*env)->DeleteLocalRef(env, jstr);
    }
```

*	编写工具类时，即时释放局部引用。

在编写工具类时，很难知道被调用者具体会是谁，考虑到通用性，完成工具类的任务之后，就要及时释放相应的局部引用，防止被占着内存空间。

*	不需要返回的 Native 方法，即时释放局部引用。

如果 Native 方法不会返回，那么自动释放局部引用就失效了，这时候就必须要手动释放。比如，在某个一直等待的循环中，如果不及时释放局部引用，很快就会溢出了。

*	局部引用使用完了就删除，不必要等到函数结尾。

比如，通过局部引用创建了一个大对象，然后这个对象在函数中间就完成了任务，那么就可以早早地通过手动释放了，而不是等到函数的结尾才释放。


#### 管理局部引用

Java 还提供了一些函数来管理局部引用的生命周期：

*	EnsureLocalCapacity
*	NewLocalRef
*	PushLocalFrame
*	PopLocalFrame

#### EnsureLocalCapacity 函数

JNI 的规范指出，JVM 要确保每个 Native 方法至少可以创建 16 个局部引用，经验表明，16 个局部引用已经足够平常的使用了。

但是，如果要与 JVM 的中对象进行复杂的交互计算，就需要创建更多的局部引用了，这时就需要使用 `EnsureLocalCapacity` 来确保可以创建指定数量的局部引用，如果创建成功返回 0 ，返回返回小于 0 ，如下代码示例：

```cpp
    // Use EnsureLocalCapacity
    int len = 20;
    if (env->EnsureLocalCapacity(len) < 0) {
        // 创建失败，out of memory
    }
    for (int i = 0; i < len; ++i) {
        jstring  jstr = env->GetObjectArrayElement(arr,i);
        // 处理 字符串
        // 创建了足够多的局部引用，这里就不用删除了，显然占用更多的内存
    }
```

引用确保可以创建了足够的局部引用数量，所以在循环处理局部引用时可以不进行删除了，但是显然会消耗更多的内存空间了。

#### PushLocalFrame 与 PopLocalFrame 函数对

PushLocalFrame 与 PopLocalFrame 是两个配套使用的函数对。

它们可以为局部引用创建一个指定数量内嵌的空间，在这个函数对之间的局部引用都会在这个空间内，直到释放后，所有的局部引用都会被释放掉，不用再担心每一个局部引用的释放问题了。

常见的使用场景就是在循环中：

```cpp
 // Use PushLocalFrame & PopLocalFrame
    for (int i = 0; i < len; ++i) {
        if (env->PushLocalFrame(len)) { // 创建指定数据的局部引用空间
            //out ot memory
        }
        jstring jstr = env->GetObjectArrayElement(arr, i);
        // 处理字符串
        // 期间创建的局部引用，都会在 PushLocalFrame 创建的局部引用空间中
        // 调用 PopLocalFrame 直接释放这个空间内的所有局部引用
        env->PopLocalFrame(NULL); 
    }
```


使用 PushLocalFrame & PopLocalFrame 函数对，就可以在期间放心地处理局部引用，最后统一释放掉。

## 全局引用
---

> 全局引用和局部引用一样，也会阻止它所引用的对象被回收。但是它不会在方法返回时被自动释放，必须要通过手动释放才行，而且，全局引用可以跨方法、跨线程使用。

全局引用只能通过 `NewGlobalRef`函数来创建，然后通过 `DeleteGlobalRef` 函数来手动释放。

还是上面提到的缓存字段的例子，现在就可以使用全局引用来缓存了。

```cpp

extern "C"
JNIEXPORT jstring JNICALL
Java_com_glumes_cppso_jnioperations_LocalAndGlobalReferenceOps_cacheWithGlobalReference(JNIEnv *env, jobject instance) {
    static jclass stringClass = NULL;
    if (stringClass == NULL) {
        jclass localRefs = env->FindClass("java/lang/String");
        if (localRefs == NULL) {
            return NULL;
        }
        stringClass = (jclass) env->NewGlobalRef(localRefs);
        env->DeleteLocalRef(localRefs);
        if (stringClass == NULL) {
            return NULL;
        }
    } else {
        LOGD("use stringClass cached");
    }
    static jmethodID stringMid = NULL;
    if (stringMid == NULL) {
        stringMid = env->GetMethodID(stringClass, "<init>", "(Ljava/lang/String;)V");
        if (stringMid == NULL) {
            return NULL;
        }
    } else {
        LOGD("use method cached");
    }
    jstring str = env->NewStringUTF("string");
    return (jstring) env->NewObject(stringClass, stringMid, str);
}
```

### 弱全局引用
----

> 弱全局引用有点类似于 Java 中的弱引用，它所引用的对象可以被 GC 回收，并且它也可以跨方法、跨线程使用。

弱引用使用 `NewWeakGlobalRef` 方法创建，使用 `DeleteWeakGlobalRef` 方法释放。

```cpp
extern "C"
JNIEXPORT void JNICALL
Java_com_glumes_cppso_jnioperations_LocalAndGlobalReferenceOps_useWeakGlobalReference(JNIEnv *env, jobject instance) {
    static jclass stringClass = NULL;
    if (stringClass == NULL) {
        jclass localRefs = env->FindClass("java/lang/String");
        if (localRefs == NULL) {
            return;
        }
        stringClass = (jclass) env->NewWeakGlobalRef(localRefs);
        if (stringClass == NULL) {
            return;
        }
    }
    static jmethodID stringMid = NULL;
    if (stringMid == NULL) {
        stringMid = env->GetMethodID(stringClass, "<init>", "(Ljava/lang/String;)V");
        if (stringMid == NULL) {
            return;
        }
    }
    jboolean isGC = env->IsSameObject(stringClass, NULL);
    if (isGC) {
        LOGD("weak reference has been gc");
    } else {
        jstring str = (jstring) env->NewObject(stringClass, stringMid,
                                               env->NewStringUTF("jstring"));
        LOGD("str is %s", env->GetStringUTFChars(str, NULL));
    }
}
```

在使用弱引用时，要先检查弱引用所指的类对象是否被 GC 给回收了。通过 `isSameObject` 方法进行检查。

`isSameObject` 方法可以用来比较两个引用类型是否相同，也可以用来比较引用是否为 NULL。同时，还可以用 `isSameObject` 来比较弱全局引用所引用的对象是否被 GC 了，返回 JNI_TRUE 则表示回收了，JNI_FALSE 则表示未被回收。

```cpp
env->IsSameObject(obj1, obj2) // 比较局部引用 和 全局引用是否相同
env->IsSameObject(obj, NULL)  // 比较局部引用或者全局引用是否为 NULL
env->IsSameObject(wobj, NULL) // 比较弱全局引用所引用对象是否被 GC 回收
``` 

## 合理管理引用
---

总结一些关于引用管理方面的知识点，可以减少内存的使用和避免因为对象被引用不能释放而造成的内存浪费。

通常来说，Native 代码大体有两种情况：

*	直接实现 Java 层声明的 Native 函数的代码
*	用在任何场景下的工具函数

对于直接实现 Java 层声明的 Native 函数，不要造成全局引用和弱全局引用的累加，因为它们在函数返回后并不会自动释放。

对于工具类的 Native 函数，由于它的调用场合和次数是不确定的，所以要格外小心各种引用类型，避免造成累积而导致内存溢出，比如如下规则：

*	返回基础类型的 Native 工具函数，不能造成全局引用、弱全局引用、局部引用的积累。
*	返回引用类型的 Native 工具函数，除了要返回的引用之外，也不能造成任何的全局引用、弱全局引用、局部引用的积累。

同时，对于工具类的 Native 函数，使用缓存技术来保存一些全局引用也是能够提高效率的，正如 [Android JNI 调用时缓存字段和方法 ID](https://glumes.com/post/android/android-jni-cache-fieldid-and-methodid/) 文章中写到的一样。同时，在工具类中，如果返回的是引用类型，最好说明返回的引用是哪一种类型，如下代码所示：

```cpp
while (JNI_TRUE) {
       jstring infoString = GetInfoString(info);
       ... /* process infoString */
       ??? /* we need to call DeleteLocalRef, DeleteGlobalRef,
              or DeleteWeakGlobalRef depending on the type of
              reference returned by GetInfoString. */
}
```

因为不清楚返回的引用是什么类型，导致不知道调用哪种方式来删除这个引用类型。

JNI 中提供了 `NewLocalRef` 函数来保证工具类函数返回的总是一个局部引用类型，如下代码：

```cpp
 static jstring cachedString = NULL;
 if (cachedString == NULL) {
       /* create cachedString for the first time */
       jstring cachedStringLocal = ... ;
       /* cache the result in a global reference */
       cachedString =(*env)->NewGlobalRef(env, cachedStringLocal);
 }
return (*env)->NewLocalRef(env, cachedString);
```

正如之前提到的，static 缓存是不能缓存一个局部引用的，而 cachedString 正是一个缓存的全局引用，但是通过 NewLocalRef 函数确保将返回的是一个局部引用，这样再使用后局部引用就会被自动释放掉了。

对于引用的管理，最好的方式还是使用 `PushLocalFrame` 与 `PopLocalFrame` 函数对，在这个函数对之间的局部引用就可以自动被 PushLocalFrame 和 PopLocalFrame 接管了。


## 参考
---

1. https://blog.csdn.net/xyang81/article/details/44657385


具体示例代码可参考我的 Github 项目 [https://github.com/glumes/AndroidDevWithCpp](https://github.com/glumes/AndroidDevWithCpp)，欢迎 Star。

