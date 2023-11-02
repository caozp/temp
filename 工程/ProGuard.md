## Keep规则

Proguard通过keep关键字配合通配符来决定哪些内容要被混淆，哪些会被保留

​	

------

proguard提供了6种不同的配置:

| **保留**                       | *防止被移除或重命名**   | **防止被重命名（未使用的会被移除）** |
| ------------------------------ | ----------------------- | ------------------------------------ |
| 类和类成员                     | -keep                   | -keepnames                           |
| 仅类成员                       | -keepclassmembers       | -keepclassmembernames                |
| 如类含有某成员，保留类及其成员 | -keepclasseswithmembers | -keepclasseswithmembernames          |

基本的格式如下

> **-keep [,modifier,...] class_specification**



## 基本指令

```
# 代码混淆压缩比，在0和7之间，默认为5，一般不需要改
-optimizationpasses 5
 
# 混淆时不使用大小写混合，混淆后的类名为小写
-dontusemixedcaseclassnames
 
# 指定不去忽略非公共的库的类
-dontskipnonpubliclibraryclasses
 
# 指定不去忽略非公共的库的类的成员
-dontskipnonpubliclibraryclassmembers
 
# 不做预校验，preverify是proguard的4个步骤之一
# Android不需要preverify，去掉这一步可加快混淆速度
-dontpreverify
 
# 有了verbose这句话，混淆后就会生成映射文件
# 包含有类名->混淆后类名的映射关系
# 然后使用printmapping指定映射文件的名称
-verbose
-printmapping proguardMapping.txt
 
# 指定混淆时采用的算法，后面的参数是一个过滤器
# 这个过滤器是谷歌推荐的算法，一般不改变
-optimizations !code/simplification/arithmetic,!field/*,!class/merging/*
 
# 保护代码中的Annotation不被混淆，这在JSON实体映射时非常重要，比如fastJson
-keepattributes *Annotation*
 
# 避免混淆泛型，这在JSON实体映射时非常重要，比如fastJson
-keepattributes Signature
 
//抛出异常时保留代码行号，在异常分析中可以方便定位
-keepattributes SourceFile,LineNumberTable

-dontskipnonpubliclibraryclasses用于告诉ProGuard，不要跳过对非公开类的处理。默认情况下是跳过的，因为程序中不会引用它们，有些情况下人们编写的代码与类库中的类在同一个包下，并且对包中内容加以引用，此时需要加入此条声明。

-dontusemixedcaseclassnames，这个是给Microsoft Windows用户的，因为ProGuard假定使用的操作系统是能区分两个只是大小写不同的文件名，但是Microsoft Windows不是这样的操作系统，所以必须为ProGuard指定-dontusemixedcaseclassnames选项
```

这里需要说的是**-keepattributes [attribute_filter]**,指定哪个属性不要混淆，可以指定多个

## 混淆规则

默认的混淆规则是：除了声明不被混淆的代码，其余的都会被混淆！比如我们在WebView中和JS交互的时候，需要提供一个接口给JS调用，如果混淆了方法名，JS就会找不到这个方法。Android项目中，哪些需要被保留呢，所以需要一个个来分析。

jni方法不可混淆，因为native方法是要完整的包名类名方法名来定义的，不能修改，否则找不到

```
-keepclasseswithmembernames class * {
    native <methods>;
}
```

保持Android底层组件和类不要混淆，四大组件会在Manifest中注册，假如混淆后类名会发生改变，不符合四大组件的注册机制

```
-keep public class * extends android.app.Activity                               
-keep public class * extends android.app.Application                           
-keep public class * extends android.app.Service                                
-keep public class * extends android.content.BroadcastReceiver                  
-keep public class * extends android.content.ContentProvider                    
-keep public class * extends android.app.backup.BackupAgentHelper               
-keep public class * extends android.preference.Preference                      
-keep public class com.android.vending.licensing.ILicensingService    
```

保持枚举 enum 类不被混淆，这两个方法是静态添加到代码中运行

```
-keepclassmembers enum * {                                                      
    public static **[] values();
    public static ** valueOf(java.lang.String);
}
```

保持 Parcelable 不被混淆和Creator静态成员变量不混淆，否则会报错BadParcelableExeception

```
-keep class * implements Android.os.Parcelable { 
      # 保持Parcelable不被混淆            
      public static final Android.os.Parcelable$Creator *;
  }
```

WebView的JS调用也需要保证写的接口方法不混淆

```
-keepclassmembers class * extends android.webkit.webViewClient {
    public void *(android.webkit.WebView, java.lang.String, android.graphics.Bitmap);
    public boolean *(android.webkit.WebView, java.lang.String)
}
-keepclassmembers class * extends android.webkit.webViewClient {
    public void *(android.webkit.webView, java.lang.String)
}

# 保留JS调用Java方法不被混淆
-keepattributes *JavascriptInterface* {
    <methods>;
}
```

保留Keep注解的类名和方法

```
-keep class android.support.annotation.Keep
# 不混淆使用了注解的类及类成员
-keep @android.support.annotation.Keep class * {*;}
# 如果类中有使用了注解的方法，则不混淆类和类成员
-keepclasseswithmembers class * {
    @android.support.annotation.Keep <methods>;
}
# 如果类中有使用了注解的字段，则不混淆类和类成员
-keepclasseswithmembers class * {
    @android.support.annotation.Keep <fields>;
}
# 如果类中有使用了注解的构造函数，则不混淆类和类成员
-keepclasseswithmembers class * {
    @android.support.annotation.Keep <init>(...);
}
```

keep继续自Activity中所有包含public void *(android.view.View)签名的方法，如onClick

```
-keepclassmembers class * extends android.app.Activity {
   public void *(android.view.View);
}
```

 R文件的静态字段

```
-keepclassmembers class **.R$* {
    public static <fields>;
}
```

避免混淆Annotation、内部类、泛型、匿名类

```
-keepattributes *Annotation*,InnerClasses,Signature,EnclosingMethod
```

自定义View的set，get方法。但凡在Layout目录下的XML布局文件配置的自定义View，都不能进行混淆。

```
-keepclassmembers public class * extends android.view.View {
   void set*(***);
   *** get*();
}
```

与服务端交互时，使用GSON、fastjson等框架解析服务端数据时，所写的**JSON对象类**不混淆，否则无法将JSON解析成对应的对象

如果有引用android-support-v4.jar包，可以添加下面这行

```
-libraryjars libs/android-support-v4.jar
-dontwarn android.support.v4.**
-keep class android.support.v4.**  { *; }
-keep interface android.support.v4.app.** { *; }
-keep public class * extends android.support.v4.**
-keep public class * extends android.app.Fragment
```

`-dontwarn android.support.v4.**`添加-dontwarn 标签，就不会提示这些警告信息





