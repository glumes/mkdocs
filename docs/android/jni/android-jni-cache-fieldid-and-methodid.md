---
title: "Android  JNI 调用时缓存字段和方法 ID"
date: 2018-05-07T10:39:50+08:00
subtitle: ""
draft: false
categories: ["Android"]
tags: ["NDK"]
toc: true
original: true
addwechat: true
slug: "android-jni-cache-fieldid-and-methodid"
 
---


在 JNI 去调用 Java 的方法和访问字段时，最先要做的操作就是获得对应的类以及对应的方法 id。

事实上，通过 FindClass 、GetFieldID、GetMethodID 去找到对应的信息是很耗时的，如果方法被频繁调用，那么肯定不能每次都去查找对应的信息，有必要将它们缓存起来，在下一次调用时，直接使用缓存内容就好了。

缓存有两种方式，分别是使用时缓存和初始化时缓存。

<!--more-->


## 使用时缓存
---

使用时缓存，就是在调用时查找一次，然后将它缓存成 `static` 变量，这样下次调用时就已经被初始化过了。

直到内存释放了，才会缓存失效。

```cpp
extern "C"
JNIEXPORT void JNICALL
Java_com_glumes_cppso_jnioperations_CacheFieldAndMethodOps_staticCacheField(JNIEnv *env, jobject instance, jobject animal) {
    static jfieldID fid = NULL; // 声明为 static 变量进行缓存
    // 两种方法都行
//    jclass cls = env->GetObjectClass(animal);
    jclass cls = env->FindClass("com/glumes/cppso/model/Animal");
    jstring jstr;
    const char *c_str;
    // 从缓存中查找
    if (fid == NULL) {
        fid = env->GetFieldID(cls, "name", "Ljava/lang/String;");
        if (fid == NULL) {
            return;
        }
    } else {
        LOGD("field id is cached");
    }
    jstr = (jstring) env->GetObjectField(animal, fid);
    c_str = env->GetStringUTFChars(jstr, NULL);
    if (c_str == NULL) {
        return;
    }
    env->ReleaseStringUTFChars(jstr, c_str);
    jstr = env->NewStringUTF("new name");
    if (jstr == NULL) {
        return;
    }
    env->SetObjectField(animal, fid, jstr);
}
```

通过声明为 static 变量进行缓存。但这种缓存方式显然有弊端，当多个调用者同时调用时，就会出现缓存多次的情况，并且每次调用时都要检查是否缓存过了。

## 初始化时缓存

在初始化时缓存，就是在类加载时，进行缓存。当类被加载进内存时，会先调用类的静态代码块，所以可以在类的静态代码块中进行缓存。

比如：
```java
public class CacheFieldAndMethodOps extends BaseOperation {
    
    static {
        initCacheMethodId(); // 静态代码块中进行缓存
    }
    private static native void initCacheMethodId();
}
```

在静态代码块中，可以将所需要的字段 id 或者方法 id 缓存成全局变量。

具体代码如下：

```cpp
// 全局变量，作为缓存方法 id
jmethodID InstanceMethodCache;

// 初始化加载时缓存方法 id
extern "C"
JNIEXPORT void JNICALL
Java_com_glumes_cppso_jnioperations_CacheFieldAndMethodOps_initCacheMethodId(JNIEnv *env, jclass type) {
    jclass cls = env->FindClass("com/glumes/cppso/model/Animal");
    InstanceMethodCache = env->GetMethodID(cls, "getName", "()Ljava/lang/String;");
}
```
在 JNI 中直接将方法 id 缓存成全局变量了，这样再调用时，就不要再进行一次查找了，并且避免了多个线程同时调用会多次查找的情况。

```cpp
extern "C"
JNIEXPORT void JNICALL
Java_com_glumes_cppso_jnioperations_CacheFieldAndMethodOps_callCacheMethod(JNIEnv *env, jobject instance, jobject animal) {
    jstring name = (jstring) env->CallObjectMethod(animal, InstanceMethodCache);
    const char *c_name = env->GetStringUTFChars(name, NULL);
    LOGD("call cache method and value is %s", c_name);
}
```


## 小结
----

可以看出，如果不能预先知道方法和字段所在类的源码，那么在使用时缓存比较合理。但如果知道的话，在初始化时缓存优点较多，既避免了每次使用时检查，还避免了在多线程被调用的情况。

具体示例代码可参考我的 Github 项目 [https://github.com/glumes/AndroidDevWithCpp](https://github.com/glumes/AndroidDevWithCpp)，欢迎 Star。
