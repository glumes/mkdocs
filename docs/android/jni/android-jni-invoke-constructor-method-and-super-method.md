---
title: "Android 通过 JNI 调用 Java 类的构造方法和父类的方法"
date: 2018-05-07T10:32:01+08:00
subtitle: ""
draft: false
categories: ["Android"]
tags: ["NDK"]
toc: true
original: true
addwechat: true
slug: "android-jni-invoke-constructor-method-and-super-method"
 
---

Android 还可以通过 JNI 来调用 Java 一个类的构造方法，从而创建一个 Java 类。

<!--more-->

## 调用构造方法

调用构造方法的步骤和之前调用类的实例方法步骤类似，也需要获得对应的类和方法 id。

对于类，通过 FindClass 可以找到对应的 Java 类型。

对于构造方法，它的方法 id 还是通过 `GetMethodID` 方法来获得，但是构造方法对应的名称为 **<init>**，返回值类型是 void 类型的。

完成了以上准备条件后，就可以通过 `NewObject` 来调用构造方法，从而创建具体的类。

下面以 String 的某个构造方法为例

```java
public String(char value[]) // Java String 类的其中一个构造方法
```

对应的 C++ 代码：
```
extern "C"
JNIEXPORT jstring JNICALL
Java_com_glumes_cppso_jnioperations_InvokeConstructorOps_invokeStringConstructors(JNIEnv *env, jobject instance) {

    jclass stringClass;
    jmethodID cid;
    jcharArray elemArr;
    jstring result;

    // 由 C++ 字符串创建一个 Java 字符串
    jstring temp = env->NewStringUTF("this is char array");
    // 再从 Java 字符串获得一个字符数组指针，作为 String 构造函数的参数
    const jchar *chars = env->GetStringChars(temp, NULL);
    int len = 10;

    stringClass = env->FindClass("java/lang/String"); // 找到具体的 String 类
    if (stringClass == NULL) {
        return NULL;
    }
    // 找到具体的方法，([C)V 表示选择 String 的 String(char value[]) 构造方法
    cid = env->GetMethodID(stringClass, "<init>", "([C)V");
    if (cid == NULL) {
        return NULL;
    }
    // 字符串数组作为参数
    elemArr = env->NewCharArray(len);
    if (elemArr == NULL) {
        return NULL;
    }
    // 给字符串数组赋值
    env->SetCharArrayRegion(elemArr, 0, len, chars);
    // 创建类
    result = (jstring) env->NewObject(stringClass, cid, elemArr);
    env->DeleteLocalRef(elemArr);
    env->DeleteLocalRef(stringClass);
    return result;
}
```

由于 String 的构造函数需要传递一个字符数组，就先构造好了字符数组并赋值，得到对应的类和方法 id 之后，直接通过 `NewObject` 方法调用即可。

再来看一个调用自定义类的构造方法的示例，还是之前的 Animal 类，它的构造方法有一个 String 类型的参数。

```cpp
/**
 * 创建一个 Java 的 Animal 类并返回
 */
extern "C"
JNIEXPORT jobject JNICALL
Java_com_glumes_cppso_jnioperations_InvokeConstructorOps_invokeAnimalConstructors(JNIEnv *env, jobject instance) {
    jclass animalClass;
    jmethodID mid;
    jobject result;
    animalClass = env->FindClass("com/glumes/cppso/model/Animal");
    if (animalClass == NULL) {
        return NULL;
    }
    mid = env->GetMethodID(animalClass, "<init>", "(Ljava/lang/String;)V");
    if (mid == NULL) {
        return NULL;
    }
    jstring args = env->NewStringUTF("this animal name");
    result = env->NewObject(animalClass, mid, args);
    env->DeleteLocalRef(animalClass);
    return result;
}
```
可以看到，整个调用流程只要按照步骤来，就可以了。


除了 NewObject 方法之外，JNI 还提供了 AllocObject 方法来创建对象，以同样调用 Animal 类构造方法为例：

```cpp
/**
 * 通过 AllocObject 方法来创建一个类
 */
extern "C"
JNIEXPORT jobject JNICALL
Java_com_glumes_cppso_jnioperations_InvokeConstructorOps_allocObjectConstructor(JNIEnv *env, jobject instance) {
    jclass animalClass;
    jobject result;
    jmethodID mid;
    // 获得对应的 类
    animalClass = env->FindClass("com/glumes/cppso/model/Animal");
    if (animalClass == NULL) {
        return NULL;
    }
    // 获得构造方法 id
    mid = env->GetMethodID(animalClass, "<init>", "(Ljava/lang/String;)V");
    if (mid == NULL) {
        return NULL;
    }
    // 构造方法的参数
    jstring args = env->NewStringUTF("use AllocObject");
    // 创建对象，此时创建的对象未初始化的对象
    result = env->AllocObject(animalClass);
    if (result == NULL) {
        return NULL;
    }
    // 调用 CallNonvirtualVoidMethod 方法去调用类的构造方法
    env->CallNonvirtualVoidMethod(result, animalClass, mid, args);
    if (env->ExceptionCheck()) {
        env->DeleteLocalRef(result);
        return NULL;
    }
    return result;
}
```

同样的，要先准备必要的东西。获得对应类的类型、方法 id、构造方法的参数。

然后通过 `AllocObject` 方法创建对象，但要注意的是，此时创建的对象是未被初始化的，不同于 `NewObject` 方法创建的对象直接就是初始化了，在一定程度上，可以说 `AllocObject` 方法是延迟初始化的。

接下来是要通过 `CallNonvirtualVoidMethod` 来调用对应的构造方法。此处传入的一个参数不再是 jclass 类型，而是创建的未被初始化的类 jobject 。

通过这种方法，同样可以创建一个 Java 中的类。


## 调用父类的方法
---

可以通过 JNI 来调用父类的实例方法。

在子类中通过调用 `CallNonvirtual<Type>Method` 方法来调用父类的方法。

首先，构造一个相应的子类，然后获得父类的 类型和方法 id，以及准备对应的参数，根据父类方法的返回值选择调用不同的 `CallNonvirtual<Type>Method`  函数。

对于引用类型的，调用 CallNonvirtualObjectMethod 方法；对于基础类型的，调用 CallNonvirtualBooleanMethod、CallNonvirtualIntMethod 等等；对于无返回值类型的，调用 CallNonvirtualVoidMethod 方法。

具体看代码：

```cpp
/**
 * 调用父类的方法
 * 创建一个子类，由子类去调用父类的方法
 */
extern "C"
JNIEXPORT void JNICALL
Java_com_glumes_cppso_jnioperations_InvokeConstructorOps_callSuperMethod(JNIEnv *env, jobject instance) {
    jclass cat_cls; // Cat 类的类型
    jmethodID cat_cid; // Cat 类的构造方法 id
    jstring cat_name; // Cat 类的构造方法参数
    jobject cat;
    // 获得对应的 类
    cat_cls = env->FindClass("com/glumes/cppso/model/Cat");
    if (cat_cls == NULL) {
        return;
    }
    // 获得构造方法 id
    cat_cid = env->GetMethodID(cat_cls, "<init>", "(Ljava/lang/String;)V");
    if (cat_cid == NULL) {
        return;
    }
    // 准备构造方法的参数
    cat_name = env->NewStringUTF("this is cat name");
    // 创建 Cat 类
    cat = env->NewObject(cat_cls, cat_cid, cat_name);
    if (cat == NULL) {
        return;
    }
    //调用父类的 getName 参数
    jclass animal_cls; // 父类的类型
    jmethodID animal_mid; // 被调用的父类的方法 id
    // 获得父类对应的类
    animal_cls = env->FindClass("com/glumes/cppso/model/Animal");
    if (animal_cls == NULL) {
        return;
    }
    // 获得父类被调用的方法 id
    animal_mid = env->GetMethodID(animal_cls, "getName", "()Ljava/lang/String;");
    if (animal_mid == NULL) {
        return;
    }
    jstring name = (jstring) env->CallNonvirtualObjectMethod(cat, animal_cls, animal_mid);
    if (name == NULL) {
        return;
    }
    LOGD("getName method value is %s", env->GetStringUTFChars(name, NULL));

    // 调用父类的其他方法
    animal_mid = env->GetMethodID(animal_cls, "callInstanceMethod", "(I)V");
    if (animal_mid == NULL) {
        return;
    }
    env->CallNonvirtualVoidMethod(cat, animal_cls, animal_mid);
}
```

Cat 类作为 Animal 类的子类，首先由 NewObject 方法创建 Cat 类，然后调用它的父类的方法。

由此，通过 JNI 来调用 Java 算是基本完成了。

具体示例代码可参考我的 Github 项目 [https://github.com/glumes/AndroidDevWithCpp](https://github.com/glumes/AndroidDevWithCpp)，欢迎 Star。