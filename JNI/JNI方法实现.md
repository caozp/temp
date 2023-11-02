## 字符串交互

**目标**

java层传递一个文件的全路径给jni层。jni层拿到路径后，然后向文件中写入数据

```
#include <android/log.h>
#include <stdio.h>
#include <string.h>

JNIEXPORT void JNICALL Java_com_example_ndktest_HelloFun_writeFile
(JNIEnv *env, jclass jcls, jstring path){

    //生成native层的char类型指针
    const char* filepath = (*env)->GetStringUTFChars(env,path,NULL);
    if (filepath !=NULL){
        __android_log_print(ANDROID_LOG_DEBUG,"FILE","filepath = %s",filepath);
    }

    //打开文件
    FILE* file = fopen(filepath,"a+");
    if (file!=NULL){
        __android_log_print(ANDROID_LOG_DEBUG,"FILE","open file success");
    }

    //写入文件
    char data[] = "Hello world ,write File";
    int count = fwrite(data,strlen(data),1,file);
    if (count>0){
        __android_log_print(ANDROID_LOG_DEBUG,"FILE","write file success");
    }
    
    //关闭文件
    if(file!=NULL){
        fclose(file);
    }
    
    //释放char类型指针
    (*env)->ReleaseStringUTFChars(env,path,filepath)
}
```

另外还配置CMakeLists.txt

添加log库的依赖

```
target_link_libraries(hello
        log)
```

## 数组交互

JNI关于数组的处理,两种处理方式

- 生成native层的数组拷贝
- 直接调用数组指针进行操作





目标，java层传递一个数组给jni，jni中对数组中的每个值进行平方，然后返回给java层

###### 方法1

```
JNIEXPORT jintArray JNICALL Java_com_example_cmakestudy_HelloFun_updateIntArray
        (JNIEnv *env, jclass jcls, jintArray array){

    jint nativeArray[5];
    (*env)->GetIntArrayRegion(env,array,0,5,nativeArray);

    int j;
    for(j = 0;j<5;j++){
        nativeArray[j] = nativeArray[j]*nativeArray[j];
    }

    (*env)->SetIntArrayRegion(env,array,0,5,nativeArray);
    return array;
}
```

数组的引用类型是一般是jarray或者或者jarray的子类型jintArray。就像jstring不是一个C字符串类型一样，jarray也不是一个C数组类型。所以，不要直接访问jarray。你必须使用合适的JNI函数来访问基本数组元素

1. 使用GetIntArrayRegion函数来把一个int数组中的所有元素复制到一个C缓冲区中，然后我们在本地代码中通过C缓冲区来访问这些元素。
2. 将基本类型数组的某一区域从缓冲区中复制回来的一组函数



###### 方法2

```
JNIEXPORT jintArray JNICALL Java_com_example_cmakestudy_HelloFun_updateIntArray
        (JNIEnv *env, jclass jcls, jintArray array){
    jint* data = (*env)->GetIntArrayElements(env,array,NULL);
    jsize len = (*env)->GetArrayLength(env,array);

    for(int j =0;j<len;j++){
        data[j] = data[j]+1;
    }

    (*env)->ReleaseIntArrayElements(env,array,data,0);
    return array;
}
```

我们在此调用了`GetIntArrayElements(...)`来获取一个指向intArray[]数组第一个元素的指针。
用`getArrayLength(..)`函数来得到数组的长度，以方便数组遍历时使用。最后应用`ReleaseArrayElements(...)`函数来释放该数组指针

## C调用Java

###### 调用静态方法

首先 我们在java的层的HelloFun中定义如下，java层调用callStaticMethod，然后native层调用java层的静态方法staticMethod，同时修改静态变量tag的值

```
    public static String tag = "helloFun";
    public static void staticMethod(String data){
        Log.d("tag-hellofun",data);
        Log.d("tag-hellofun",tag);
    }

    public static native void callStaticMethod(int i);
```

方法实现如下

```
JNIEXPORT void JNICALL Java_com_example_cmakestudy_HelloFun_callStaticMethod
        (JNIEnv *env, jclass jcls, jint i){
        //1.找到HelloFun类，需要将包名的.全部替换成斜杠/
    jclass  cls_hello = (*env)->FindClass(env,"com/example/cmakestudy/HelloFun");
    if (cls_hello == NULL){
        __android_log_print(ANDROID_LOG_INFO,"jni","cls_hello");
        return;
    }

     //2.根据方法名，方法签名，通过GetStaticMethodID发现静态方法staticMethod，
    jmethodID mth_static_method = (*env)->GetStaticMethodID(env,cls_hello,"staticMethod","(Ljava/lang/String;)V");
    if (mth_static_method==NULL){
        __android_log_print(ANDROID_LOG_INFO,"jni","mth_static_method");
        return;
    }

    //3.找到静态字段
    jfieldID  fld_static = (*env)->GetStaticFieldID(env,cls_hello,"tag","Ljava/lang/String;");

    //4.对静态字段重新赋值为Hello World,Jni
    jstring name = (*env)->NewStringUTF(env,"Hello World,Jni");
    (*env)->SetStaticObjectField(env,cls_hello,fld_static,name);

    //5.调用静态方法staticMethod，传入参数
    jstring data = (*env)->NewStringUTF(env,"native call java static method");
    (*env)->CallStaticVoidMethod(env,jcls,mth_static_method,data);

    //6.删除引用
    (*env)->DeleteLocalRef(env,cls_hello);
    (*env)->DeleteLocalRef(env,data);
    (*env)->DeleteLocalRef(env,name);
}
```

###### 调用实例方法

调用实例方法 需要构造实例，java中通过new创建对象，native层中通过调用对象的构造方法

```
jmethod mthconstruct= (*env)->GetMethodID(env,cls_hello,"<init>","()V");
jobject hello = (*env)->NewObject(env,cls_hello,mthconstruct,NULL);

(*env)->DeleteLocalRef(env,hello);
```

