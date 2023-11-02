# Bitmap

位图，是使用像素阵列表示的图像。Bitmap位图包括像素以及长、宽、颜色等描述信息。长宽和像素位数是用来描述图片的，可以通过这些信息计算出图片的像素占用内存的大小。



## PNG与JPG

我们平时见到的后缀名为PNG或者JPG的文件，就是位图。他们之间的区别：

- JPEG——有损压缩,会牺牲图片质量
- PNG——无损压缩,背景支持透明



## 颜色通道

位图将图像定义为由点（像素）组成，每个点可以由多种色彩表示。

在安卓系统中bitmap图片一般是以**ARGB_8888**（ARGB分别代表的是透明度,红色,绿色,蓝色,每个值分别用8bit来记录,也就是一个像素会占用4byte,共32bit。）来进行存储的。

> 一张位图的内存大小 = 图片的长度px *  宽度px * 每个像素点所占的内存

而每个像素点所占内存的大小跟色彩模式有关，常见的Config有：

| **Bitmap.Config** | 字节                                                         |
| ----------------- | ------------------------------------------------------------ |
| ARGB_4444         | 即A=4，R=4，G=4，B=4，那么一个像素点占4+4+4+4=16位 ,占2字节  |
| ARGB_8888         | 即A=8，R=8，G=8，B=8，那么一个像素点占8+8+8+8=32位，占4字节  |
| RGB_565           | 即R=5，G=6，B=5，没有透明度，那么一个像素点占5+6+5=16位，占2字节 |
| ALPHA_8           | 每个像素占四位，只有透明度，没有颜色 A = 8。占1字节          |

## 相关API

**静态方法createBitmap**——返回bitmap示例

```
    public static Bitmap createBitmap(@NonNull Bitmap source, int x, int y, int width, int height) {
        return createBitmap(source, x, y, width, height, null, false);
    }
```

通过createBitmap+Matrix可以实现bitmap的旋转

```
Matrix matrix = new Matrix();
matrix.setRotate(angle);
        // 围绕原地进行旋转
Bitmap newBM = Bitmap.createBitmap(origin, 0, 0, width, height, matrix, false);
```



**压缩方法**——按指定的图片格式以及画质，将图片转换为输出流。 

- quality 压缩质量0-100，数值越高，质量越高
- format 格式Bitmap.CompressFormat.PNG或Bitmap.CompressFormat.JPEG

```
public boolean compress(CompressFormat format, int quality, OutputStream stream) {}
```

**内存大小**——getByteCount()

**回收**——isRecycled()，recycle()回收内存，把位图标记为Dead



## BitmapDrawable

bitmapDrawable是drawable的子类，并不是所有的drawable都有内部的宽高概念，但是BitmapDrawable就是一张图片，它的内部宽高就是图片的宽高，可以通过`getIntrinsicWidth`,`getIntrinsicHeight`获取他们。

```
bitmap转drawable
BitmapDrawable drawable = new BitmapDrawable(getResources(),bitmap);
==================================================================================
bitmapdrawable转bitmap:
private void drawableToBitamp(Drawable drawable)   {
    BitmapDrawable bd = (BitmapDrawable) drawable;
    bitmap = bd.getBitmap();
}
```



## 加载

如何加载一个图片呢？BitmapFactory类提供了四类方法：

- `decodeFile(String pathName, Options opts)`
- `decodeResource(Resources res, int id, Options opts)`
- `decodeSteam(@Nullable InputStream is)`
- `decodeByteArray(byte[] data, int offset, int length, Options opts)`

分别用于支持从文件系统，资源，输入流和字节数组中加载出一个bitmap对象。其中`decodeFile`和`decodeResource`又间接调用了`decodeSteam`方法，这四类方法最终都是在安卓底层实现的。

### decodeResource

图片的内存大小计算公式是：分辨率*像素点大小，但是然后如果图片的来源是在 res 的话，就需要注意，图片是放于哪个资源目录下的，以及设备本身的 dpi 值，因为`decodeResource()` 方法内部会根据 dpi 进行分辨率的转换，这样的话，图片的内存大小就会不一样。

> 而转换的规则是：
>
> 新图的高度 = 原图高度 * (设备的 dpi / 目录对应的 dpi ) 
>
> 新图的宽度 = 原图宽度 * (设备的 dpi / 目录对应的 dpi )

| 密度   | 值   |
| ------ | ---- |
| ldpi   | 120  |
| mdpi   | 160  |
| hdpi   | 240  |
| xhdpi  | 320  |
| xxhdpi | 480  |

结论：**位于 res 内的不同资源目录中的图片，当加载进内存时，会先经过一次分辨率的转换，然后再计算大小，转换的影响因素是设备的 dpi 和不同的资源目录。**

## 压缩

### 质量压缩

质量压缩主要借助Bitmap中的compress方法实现

```
public static void compressQuality(Bitmap bitmap, int quality, File file) {
    ByteArrayOutputStream baos = new ByteArrayOutputStream();
    bitmap.compress(Bitmap.CompressFormat.JPEG, quality, baos);
    try {
        FileOutputStream fos = new FileOutputStream(file);
        fos.write(baos.toByteArray());
        fos.flush();
        fos.close();
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

但是,**质量压缩只是改变磁盘中的文件大小，并不能改变加载时内存中的图片大小**。因为，它在内存中的图片大小还是图片的宽 * 高 * 每个像素的所占内存大小



### 尺寸压缩

等比例缩放图片大小。**改变了加载时的宽高，等于改变了加载时内存中的图片大小**

```
/**
 * 尺寸压缩
 *
 * @param bitmap
 * @param file
 */
public static void compressSize(Bitmap bitmap, File file) {
    int ratio = 8;//尺寸压缩比例
    Bitmap result = Bitmap.createBitmap(bitmap.getWidth() / ratio, bitmap.getHeight() / ratio, Bitmap.Config.ARGB_8888);
    Canvas canvas = new Canvas(result);
    Rect rect = new Rect(0, 0, bitmap.getWidth() / ratio, bitmap.getHeight() / ratio);
    canvas.drawBitmap(bitmap, null, rect, null);

    compressQuality(result, 100, file);
}
```



### 采样率压缩

BitmapFactory.Options用来缩放图片，主要是用到它的inSampleSize参数，即采样率。官方文档中指出，inSampleSize的取值应该总是为2的指数，比如1，2，4，8，16等。如果小于1，作用等于1，没有缩放效果。如果不为2的指数，那么系统会向下取整，并选择一个接近2的指数代替。

如何获取采样率呢？

1. 将Options的inJustDecodeBounds参数设为true加载图片
2. 从Options中取出图片的原始宽高信息，对应与outWidth和outHeight参数
3. 根据需要计算出采样率
4. 将Options的inJustDecodeBounds参数设为False，重新加载

需要说明的是**Options的inJustDecodeBounds参数为true时，BitmapFactory只会解析图片的原始信息，并不会真正的去加载图片，这个操作时轻量级的**

