# IntentFilter的匹配规则

隐式调用需要Intent能够匹配目标组件的IntentFilter中所设置的过滤信息，如果不普配将无法启动Activity。IntentFilter的过滤信息有action，category，data。

一个activity中可以有很多个intent-filter,**一个intent只能匹配任何一组intent-filter即可成功启动activity**。



## action

隐式启动Intent中的action**存在且必须**和过滤规则中的其中一个action相同。如果不存在，则匹配失败。



## category

category要求intent可以没有category，但是如果一旦有category，不管有几个，每个都要求能够跟过滤规则中的任何一个category相同。为了能接受隐式调用，必须在intent-filter中指定"android.intent.category.DEFAULT"这个category



## data

如果过滤规则中定义了data，那么Intent中必须也要定义可匹配的data。

data分为两部分，mimeType和URI。mimeType指媒体类型，如image/jpeg等，URI统一资源定位符。

通过

> intent.setDataAndType(Uri.parse("file://abc"),"image/png")



## 最后

通过隐式启动activity的时候，可以做一下判断。采用PackageManager的resolveActivity方法或者Intent的resolveActivity方法，如果匹配不到就会返回null。PackageManager还提供queryIntentActivities方法。这个方法返回所有匹配成功的activity

```
public abstract List<ResolveInfo> queryIntentActivities(Intent intent,int flag);
public abstract ResolveInfo resolveActivity(Intent intent,int flag);
```

------

Android 5.0 (Lollipop) 之后规定。Service Intent, 要用显性意图。如果想继续使用隐式意图的话，加上包名信息即可

```
Intent intent = new Intent();
intent.setAction("example.czp.com.aidldemo.MyService");
intent.setPackage(getPackageName());
bindService(intent, conn, Context.BIND_AUTO_CREATE);
```

否则报错**Caused by: java.lang.IllegalArgumentException: Service Intent must be explicit**

