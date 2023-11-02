# JNIEnv

JNIEnv是Native世界中Java环境的代表

通过JNIEnv *指针就可以在native世界中访问Java世界的代码进行操作。它只在创建它的线程中有效，不能跨线程传递。因此不同的JNIEnv是彼此独立的。



## 定义

jni.h对JNIEnv是如下的定义

```
#if defined(__cplusplus)
typedef _JNIEnv JNIEnv;
typedef _JavaVM JavaVM;
#else
typedef const struct JNINativeInterface* JNIEnv;
typedef const struct JNIInvokeInterface* JavaVM;
#endif
```

这里使用预定义宏_cplusplus来区分C和C++两种代码。

如果定义了_cplusplus，就是C++代码中的定义，否则就是C的代码。

在C++中，JNIEnv是结构体。

在C语言中是`JNINativeInterface`的指针类型

```
#include <jni.h>

#ifndef _Included_com_example_sample_jni_HelloFun
#define _Included_com_example_sample_jni_HelloFun
#ifdef __cplusplus
extern "C" {
#endif

...

#ifdef __cplusplus
}
#endif
#endif
```

那么在C++中`_JNIEnv`又是如何定义的呢？

```
struct _JNIEnv {
    /* do not rename this; it does not seem to be entirely opaque */
    const struct JNINativeInterface* functions;
    #if defined(__cplusplus)

    jint GetVersion()
    { return functions->GetVersion(this); }

    ...

#endif /*__cplusplus*/
};
```

可以看到内部也是包含了JNINativeInterface的指针，无论是C还是C++,都和JNINativeInterface结构有关



## 引用类型

JNI中也有引用类型，分别是本地引用，全局引用和弱引用。

### 本地引用

JNIEnv提供的函数所返回的引用基本都是本地引用。

* 在Native函数返回时，本地引用就会自动释放。
* 本地引用也只在创建它的线程中有效，不能够跨线程使用。
* 本地引用是JVM负责的引用类型。受JVM管理

我们也可以是用`DeleteLocalRef`来手动删除本地引用

### 全局引用

* 不受JVM管理
* 可以跨进程访问。
* 需要手动来释放，不会被GC回收

通过NewGlobalRef函数来创建全局引用，DeleteGlobalRef函数来释放全局引用

