WorkManager 是一个 Jetpack库, 它在工作的触发器 (如适当的网络状态和电池条件) 满足时, 优雅地运行可推迟的后台工作。WorkManager 尽可能使用框架 JobScheduler , 以帮助优化电池寿命和批处理作业。在 Android 6.0 (API 级 23) 下面的设备上, 如果 WorkManager 已经包含了应用程序的依赖项, 则尝试使用Firebase JobDispatcher 。否则, WorkManager 返回到自定义 AlarmManager 实现, 以优雅地处理您的后台工作。

[workmanager的官方介绍](https://developer.android.google.cn/topic/libraries/architecture/workmanager/)

[Android新技术之从Service到WorkManager](https://blog.csdn.net/c6E5UlI1N/article/details/80504526)

WorkManager会根据移动设备的API级别和应用程序状态等因素选择适当的方式来运行我们的任务，如果WorkManager在应用程序运行时执行其中一项任务，则WorkManager可以在应用进程中开启一个的新线程中运行这个任务。如果你的应用程序未运行，WorkManager将选择合适的方式来安排后台任务 - 这取决于设备API级别和应用包含的依赖关系，WorkManager可能使用JobScheduler，Firebase JobDispatcher或AlarmManager。我们完全不需要编写设备逻辑来确定设备具有哪些功能并选择适当的API，相反，我们可以将任务交给WorkManager并让它选择最佳选项。



### 定义worker

新建一个jave类：继承自抽象类Worker，必须实现doWork()方法，要在这个方法里，操作后台任务,非主线程

```
public class MyWork extends Worker {

    @NonNull
    @Override
    public Result doWork() {
        //不需要开线程了 直接写自己的任务
        return null;//3个枚举值SUCCESS,FAILURE,RETRY
    }
}
```

### 定义WorkRequest

在主Activity中，定义WorkRequest，具体可以选择两个子类，**OneTimeWorkRequest**(任务只执行一遍)、**PeriodicWorkRequest**(任务周期性的执行)

```
PeriodicWorkRequest request = new PeriodicWorkRequest.Builder(MyWorker.class, 15, TimeUnit.SECONDS).build()
```

### 加入队列

将WorkRequest加入队列

```
WorkManager.getInstance().enqueue(request);
```

### 传入数据

有时候需要传入数据，在Activity定义Data，将需要传入的数据包装一下，然后通过WorkRequest的setInputData()传入

```
Data data = new Data.Builder().putInt("params1", 1).putString("params2", "hello").build();
PeriodicWorkRequest request = new PeriodicWorkRequest.Builder(MyWorker.class, 15, TimeUnit.SECONDS)
                .setInputData(data)
                .build();

```

可以在Worker中获取

```
public class MyWorker extends Worker
{
    String tag = MyWorker.class.getSimpleName();
    @NonNull
    @Override
    public Result doWork()
    {
        int params1 = getInputData().getInt("params1",0);
        String params2 = getInputData().getString("params2");
        Log.d(tag,"获得参数:"+params1+","+params2);
        return Result.SUCCESS;
    }

    @Override
    public void onStopped(boolean cancelled)
    {
        super.onStopped(cancelled);
        Log.e(tag,"Worker Stopped");
    }

}
```



### 返回数据

doWork()方法是返回void的。你要是有结果想传出去, 就可以用Worker.setOutputData()

```
Data output = new Data.Builder()
                .putInt("a", 1)
                .build();
        setOutputData(output);
```

### WorkStatus

当 WorkRequest入列后，WorkManager 会给它分配一个 work ID，WorkManager可以通过WorkRequest的id，获取到WorkRequest的WorkStatus，返回的是LiveData 形式

```
WorkManager.getInstance().getStatusById(compressionWork.getId())
    .observe(lifecycleOwner, workStatus -> {
        // Do something with the status
        if (workStatus != null && workStatus.getState().isFinished()) {
            // ...
        }
    });
```

### 标签

```
 PeriodicWorkRequest request = new PeriodicWorkRequest.Builder(MyWorker.class, 15, TimeUnit.SECONDS)
                .setInputData(data)
                .addTag("A")
                .build();
PeriodicWorkRequest request2 = new PeriodicWorkRequest.Builder(MyWorker.class, 15, TimeUnit.SECONDS)
                .setInputData(data)
                .addTag("A")
                .build();

```

通过addTag()，将WorkRequest成为了一个组：A组。以后可以直接控制整个组就行了，组内的每个成员都会受到影响。比如通过WorkManager的cancelAllWorkByTag(String tag):取消一组带有相同标签的任务。

### 取消任务

WorkManager可以通过WorkRequest的id取消或者停止任务,

```
WorkManager.getInstance().cancelWorkById(request.id)
~~.cancelAllWork():取消所有任务。 
~~.cancelAllWorkByTag(String tag):取消一组带有相同标签的任务。 
~~.cancelUniqueWork( String uniqueWorkName):取消唯一任务。
```

### 约束

WorkManager 允许指定任务执行的环境，比如网络已连接、电量充足时等，在满足条件的情况下任务才会执行。 

```
Constraints constraints = new Constraints.Builder()
                .setRequiresDeviceIdle(true)//指定{@link WorkRequest}运行时设备是否为空闲
                .setRequiresCharging(true)//在设备充电时才能执行任务
                .setRequiredNetworkType(NetworkType.UNMETERED)
                .setRequiresBatteryNotLow(true)//指定设备电池是否不应低于临界阈值
                .setRequiresDeviceIdle(true)//指定{@link WorkRequest}运行时设备是否为空闲
                .setRequiresStorageNotLow(true)//指定设备可用存储是否不应低于临界阈值
                .addContentUriTrigger(myUri,false)//指定内容{@link android.net.Uri}时是否应该运行{@link WorkRequest}更新
                .build();
                
                
 /**
     * 指定网络状态执行任务
     * NetworkType.NOT_REQUIRED：对网络没有要求
     * NetworkType.CONNECTED：网络连接的时候执行
     * NetworkType.UNMETERED：不计费的网络比如WIFI下执行
     * NetworkType.NOT_ROAMING：非漫游网络状态
     * NetworkType.METERED：计费网络比如3G，4G下执行。
     */
```

在WorkRequest中，通过setConstraints设置约束



### 链式任务

如果处理的不是一个任务，而是一组任务，可以按照一定顺序来执行，也可以按照组合来执行，如果任务链中的任何一个任务，返回WorkerResult.FAILURE，任务链终止 
按照一定顺序执行，需要使用WorkManager的then()方法，将需要执行的任务依次加入 

```
WorkManager.getInstance().beginWith(requestA).then(requestB).enqueue();
```

### 任务唯一性

```
WorkManager.getInstance()
        .beginUniqueWork("unique", ExistingWorkPolicy.REPLACE, request)
        .enqueue()
```

REPLACE：删除已有的 添加新的;  
KEEP:什么都不做，不添加新任务，让已有的继续执行;  
APPEND:加入已有任务的任务链最末端





看下官方的[workManagerdemo](https://github.com/googlesamples/android-architecture-components/tree/master/WorkManagerSample),主要是实现图片的水波纹,模糊,上传功能。选择了一个上传图片的work学习下

```
class UploadWorker(appContext: Context, workerParams: WorkerParameters)
    : Worker(appContext, workerParams) {

    companion object {
        private const val TAG = "UploadWorker"
    }

    override fun doWork(): Result {
        var imageUriInput: String? = null
        try {
            val args = inputData
            imageUriInput = args.getString(Constants.KEY_IMAGE_URI)
            val imageUri = Uri.parse(imageUriInput)
            val imgurApi = ImgurApi.instance.value
            // Upload the image to Imgur.
            val response = imgurApi.uploadImage(imageUri).execute()
            // Check to see if the upload succeeded.
            if (!response.isSuccessful) {
                val errorBody = response.errorBody()
                val error = errorBody?.string()
                val message = String.format("Request failed %s (%s)", imageUriInput, error)
                Log.e(TAG, message)
                Toast.makeText(applicationContext, message, Toast.LENGTH_SHORT).show()
                return Result.failure()
            } else {
                val imageResponse = response.body()
                var outputData = workDataOf()
                if (imageResponse != null) {
                    val imgurLink = imageResponse.data!!.link
                    // Set the result of the worker by calling setOutputData().
                    outputData = Data.Builder()
                            .putString(Constants.KEY_IMAGE_URI, imgurLink)
                            .build()
                }
                return Result.success(outputData)
            }
        } catch (e: Exception) {
            val message = String.format("Failed to upload image with URI %s", imageUriInput)
            Toast.makeText(applicationContext, message, Toast.LENGTH_SHORT).show()
            Log.e(TAG, message)
            return Result.failure()
        }
    }
}
```

从val response = imgurApi.uploadImage(imageUri).execute();这里用了网络请求的同步方法，可见在doWork()方法里是可以执行耗时操作的，不需要去开线程