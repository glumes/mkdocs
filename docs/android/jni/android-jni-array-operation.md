---
title: "Android JNI 数组 操作"
date: 2018-05-07T10:21:22+08:00
subtitle: ""
draft: false
categories: ["Android"]
tags: ["NDK"]
toc: true
original: true
addwechat: true
slug: "android-jni-array-operation"
 
---


JNI 中有两种数组操作，基础数据类型数组和对象数组，JNI 对待基础数据类型数组和对象数组是不一样的。

<!--more-->

## 基本数据类型数组
---

对于基本数据类型数组，JNI 都有和 Java 相对应的结构，在使用起来和基本数据类型的使用类似。

在 Android JNI 基础知识篇提到了 Java 数组类型对应的 JNI 数组类型。比如，Java int 数组对应了 jintArray，boolean 数组对应了 jbooleanArray。

如同 String 的操作一样，JNI 提供了对应的转换函数：GetArrayElements、ReleaseArrayElements。

```java
    intArray = env->GetIntArrayElements(intArray_, NULL);
    env->ReleaseIntArrayElements(intArray_, intArray, 0);
```

另外，JNI 还提供了如下的函数：

*	GetTypeArrayRegion / SetTypeArrayRegion

将数组内容复制到 C 缓冲区内，或将缓冲区内的内容复制到数组上。

*	GetArrayLength

得到数组中的元素个数，也就是长度。

*	NewTypeArray

返回一个指定数据类型的数组，并且通过 SetTypeArrayRegion 来给指定类型数组赋值。

*	GetPrimitiveArrayCritical / ReleasePrimitiveArrayCritical

如同 String 中的操作一样，返回一个指定基础数据类型数组的直接指针，在这两个操作之间不能做任何阻塞的操作。

实际操作如下：

```java
    // Java 传递 数组 到 Native 进行数组求和
    private native int intArraySum(int[] intArray, int size);
```

对应的 C++ 代码如下：
```cpp
JNIEXPORT jint JNICALL
Java_com_glumes_cppso_jnioperations_ArrayTypeOps_intArraySum(JNIEnv *env, jobject instance,
                                                             jintArray intArray_, jint num) {
    jint *intArray;
    int sum = 0;
    // 操作方法一：
    // 如同 getUTFString 一样，会申请 native 内存
    intArray = env->GetIntArrayElements(intArray_, NULL);
    if (intArray == NULL) {
        return 0;
    }
    // 得到数组的长度
    int length = env->GetArrayLength(intArray_);
    LOGD("array length is %d", length);
    for (int i = 0; i < length; ++i) {
        sum += intArray[i];
    }
    LOGD("sum is %d", sum);

    // 操作方法二：
    jint buf[num];
    // 通过 GetIntArrayRegion 方法来获取数组内容
    env->GetIntArrayRegion(intArray_, 0, num, buf);
    sum = 0;
    for (int i = 0; i < num; ++i) {
        sum += buf[i];
    }
    LOGD("sum is %d", sum);
    // 使用完了别忘了释放内存
    env->ReleaseIntArrayElements(intArray_, intArray, 0);
    return sum;
}
```


假如需要从 JNI 中返回一个基础数据类型的数组，对应的代码如下：

```java
    // 从 Native 返回基本数据类型数组
    private native int[] getIntArray(int num);
```

对应的 C++ 代码如下：
```cpp
/**
 * 从 Native 返回 int 数组，主要调用 set<Type>ArrayRegion 来填充数据，其他数据类型类似操作
 */
extern "C"
JNIEXPORT jintArray JNICALL
Java_com_glumes_cppso_jnioperations_ArrayTypeOps_getIntArray(JNIEnv *env, jobject instance,
                                                             jint num) {
    jintArray intArray;
    intArray = env->NewIntArray(num);

    jint buf[num];
    for (int i = 0; i < num; ++i) {
        buf[i] = i * 2;
    }

    // 使用 setIntArrayRegion 来赋值
    env->SetIntArrayRegion(intArray, 0, num, buf);
    return intArray;
}
```

以上例子，基本把相关的操作都使用上了，可以发现和 String 的操作大都是相似的。


## 对象数组
---

对于对象数组，也就是引用类型数组，数组中的每个类型都是引用类型，JNI 只提供了如下函数来操作。

*	GetObjectArrayElement / SetObjectArrayElement

和基本数据类型不同的是，不能一次得到数据中的所有对象元素或者一次复制多个对象元素到缓冲区。只能通过上面的函数来访问或者修改指定位置的元素内容。

字符串和数组都是引用类型，因此也只能通过上面的方法来访问。

例如在 JNI 中创建一个二维的整型数组并返回：

```java
    // 从 Native 返回二维整型数组，相当于是一个一维整型数组，数组中的每一项内容又是数组
    private native int[][] getTwoDimensionalArray(int size);
```

二维数组具有特殊性在于，可以将它看成一维数组，其中数组的每项内容又是一维数组。

具体 C++ 代码如下：

```cpp
/**
 * 从 Native 返回一个二维的整型数组
 */
extern "C"
JNIEXPORT jobjectArray JNICALL
Java_com_glumes_cppso_jnioperations_ArrayTypeOps_getTwoDimensionalArray(JNIEnv *env,
                                                                        jobject instance,
                                                                        jint size) {
    // 声明一个对象数组
    jobjectArray result;
    // 找到对象数组中具体的对象类型,[I 指的就是数组类型
    jclass intArrayCls = env->FindClass("[I");

    if (intArrayCls == NULL) {
        return NULL;
    }
    // 相当于初始化一个对象数组，用指定的对象类型
    result = env->NewObjectArray(size, intArrayCls, NULL);

    if (result == NULL) {
        return NULL;
    }
    for (int i = 0; i < size; ++i) {
        // 用来给整型数组填充数据的缓冲区
        jint tmp[256];
        // 声明一个整型数组
        jintArray iarr = env->NewIntArray(size);
        if (iarr == NULL) {
            return NULL;
        }
        for (int j = 0; j < size; ++j) {
            tmp[j] = i + j;
        }
        // 给整型数组填充数据
        env->SetIntArrayRegion(iarr, 0, size, tmp);
        // 给对象数组指定位置填充数据，这个数据就是一个一维整型数组
        env->SetObjectArrayElement(result, i, iarr);
        // 释放局部引用
        env->DeleteLocalRef(iarr);
    }
    return result;
}
```

首先需要使用 NewObjectArray 方法来创建对象数组。

然后使用 SetObjectArrayElement 函数填充数据时，需要构建好每个位置对应的对象。这里就使用了 NewIntArray 来创造了一个对象，并给对象填充数据后，在赋值给对象数组。

通过一个 for 循环就完成给对象数组赋值的操作。

在创建对象数组时，有一个操作是找到对应的对象类型，通过 findClass 方法。findClass 的参数 **[I** 这里就涉及到 Java 与 JNI 对应签名的转换。

## Java 与 JNI 签名的转换
----

在前一篇文章中，用表格列出了 Java 与 JNI 对应的数据类型格式的转换关系，现在要列举的是 Java 与 JNI 对应签名的转换关系。

这里的签名指的是在 JNI 中去查找 Java 中对应的数据类型、对应的方法时，需要将 Java 中的签名转换成 JNI 所能识别的。

#### 对于类的签名转换

对于 Java 中类或者接口的转换，需要用到 Java 中类或者接口的全限定名，把 Java 中描述类或者接口的 **.** 换成 **/** 就好了，比如 String 类型对应的 JNI 描述为：

```cpp
java/lang/String     // . 换成 / 
```

对于数组类型，则是用 **[** 来表示数组，然后跟一个字段的签名转换。

```cpp
[I 		// 代表一维整型数组，I 表示整型
[[I	    // 代表二维整型数组
[Ljava/lang/String;      // 代表一维字符串数组， 
```

#### 对于字段的签名转换

对应基础类型字段的转换：

|Java 类型|JNI 对应的描述转|
|---|---|
|boolean|Z|
|byte|B|
|char|C|
|short|S|
|int|I|
|long|J|
|float|F|
|double|D|


对于引用类型的字段签名转换，是大写字母 **L** 开头，然后是类的签名转换，最后以 **;** 结尾。

|Java 类型| JNI 对应的描述转换|
|---|---|
|String|Ljava/lang/String;|
|Class|Ljava/lang/Class;|
|Throwable|Ljava/lang/Throwable|
|int[]|"[I"|
|Object[]|"[Ljava/lang/Object;"



#### 对于方法的签名转换

对于方法签名描述的转换，首先是将方法内所有参数转换成对应的字段描述，并全部写在小括号内，然后在小括号外再紧跟方法的返回值类型描述。

|Java 类型|JNI 对应的描述转换|
|---|---|
|String f();|()Ljava/lang/String;|
|long f(int i, Class c);|(ILjava/lang/Class;)J|
|String(byte[] bytes);|([B)V|

这里要注意的是在 JNI 对应的描述转换中不要出现空格。

了解并掌握这些转换后，就可以进行更多的操作了，实现 Java 与 C++ 的相互调用。


比如，有一个自定义的 Java 类，然后再 Native 中打印类的对象数组的某一个字段值。

```java
    private native void printAnimalsName(Animal[] animal);
```

具体 C++ 代码如下：

```cpp
/**
 * 打印对象数组中的信息
 */
extern "C"
JNIEXPORT void JNICALL
Java_com_glumes_cppso_jnioperations_ArrayTypeOps_printAnimalsName(JNIEnv *env, jobject instance,
                                                                  jobjectArray animals) {
    jobject animal;
    // 数组长度
    int size = env->GetArrayLength(animals);
    // 数组中对应的类
    jclass cls = env->FindClass("com/glumes/cppso/model/Animal");
    // 类对应的字段描述
    jfieldID fid = env->GetFieldID(cls, "name", "Ljava/lang/String;");
    // 类的字段具体的值
    jstring jstr;
    // 类字段具体值转换成 C/C++ 字符串
    const char *str;

    for (int i = 0; i < size; ++i) {
        // 得到数组中的每一个元素
        animal = env->GetObjectArrayElement(animals, i);
        // 每一个元素具体字段的值
        jstr = (jstring) (env->GetObjectField(animal, fid));
        str = env->GetStringUTFChars(jstr, NULL);
        if (str == NULL) {
            continue;
        }
        LOGD("str is %s", str);
        env->ReleaseStringUTFChars(jstr, str);
    }
}
```

具体示例代码可参考我的 Github 项目 [https://github.com/glumes/AndroidDevWithCpp](https://github.com/glumes/AndroidDevWithCpp)，欢迎 Star。