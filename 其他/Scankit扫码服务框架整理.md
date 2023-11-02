# Scankit

主要有sdk跟apk，apk与sdk是同一套代码。

解码能力分为lite和plus版本。plus是算法工程师通过custom cv实现的。

## aidl接口

scankit 会比较sdk版本号与hms core中apk版本的大小，来决定加载本地或者远端。

**旧版本中:**

sdk版本号大于apk版本号时，使用的是本地的页面，本地的算法能力(取决于是依赖scan还是scanplus)；

sdk版本号小于apk版本号时，使用的是远端的页面，远端apk的算法plus能力。



**新版本中**:

将页面与算法接口分离。apk提供解码能力接口

也就是

sdk 只用本地的页面，再判断sdk版本与apk版本号，决定是用本地算法能力或者apk算法能力。



**实现逻辑:**

这里解码能力接口为示例

1. 首先是定义`IDecodeDelegate`的aidl接口，内部有`decode`方法

   ```
   interface IDecodeDelegate {
       Result decode(byte[] bytes);
   }
   ```

2. 然后我们需要去实现`decode`具体方法，通过继承`IDecodeDelegate.Stub`的方式

   ```
   class DecodeDelegateImp extends IDecodeDelegate.Stub{
   
       Result decode(byte[] bytes) {
            ...//这里是算法的实现
       }
   }
   ```

3. 接着我们去定义一个`IDecodeCreator`的aidl接口，它主要是提供给我们获取`IDecodeDelegate`接口

   ```
   interface IDecodeCreator {
       IDecodeDelegate newDecodeDelegate();
   }
   
   ```

4. 然后同样我们需要定义`DecodeCreator`来继承`IDecodeCreator.Stub`。实现`newDecodeDelegate`方法，返回`IDecodeDelegate`接口

5. 我们通过判断版本号后，首先需要获取Context，它可能是local的context，可能是apk的context。.然后通过context进行类加载来创建`DecodeCreator`类

   ```
   
   Object o = DecodeCreator.class.newInstance();
       //远端的类是通过context.loadClass("...")
   if(o instanceof IBinder) {
      IBinder b = IDecodeCreator.Stub.asInterface(o);
      //在获取解码的接口
      IDecodeDelegate gate = b.newDecodeDelegate()
   }
   ```

   最后通过**gate**来调用decode方法完成解码部分。





