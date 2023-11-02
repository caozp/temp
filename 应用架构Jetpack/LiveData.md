# LiveData

LiveData 是一个可以感知 Activity 、Fragment生命周期的数据容器。当 LiveData 所持有的数据改变时，它会通知相应的界面代码进行更新。同时，LiveData 持有界面代码 Lifecycle 的引用，这意味着它会在界面代码（LifecycleOwner）的生命周期处于 started 或 resumed 时作出相应更新，而在 LifecycleOwner 被销毁时停止更新。

LiveData通常跟ViewModel配合使用

在Activity或fragment之中，我们先获取ViewModel。然后通过ViewModel观察Livedata，LiveData会在每一次`postValue(...)`或者`value=...`时，`observe()`便会回调，哪怕是null。

它的优点：

- UI和实时数据保持一致
  因为LiveData采用的是观察者模式，这样一来就可以在数据发生改变时获得通知，更新UI。
- 避免内存泄漏
  观察者被绑定到组件的生命周期上，当被绑定的组件销毁（destory）时，观察者会立刻自动清理自身的数据。
- 不会再产生由于Activity处于stop状态而引起的崩溃
  例如：当Activity处于后台状态时，是不会收到LiveData的任何事件的。
- 不需要再解决生命周期带来的问题
  LiveData可以感知被绑定的组件的生命周期，只有在活跃状态才会通知数据变化。
- 实时数据刷新
  当组件处于活跃状态或者从不活跃状态到活跃状态时总是能收到最新的数据
- 解决Configuration Change问题
  在屏幕发生旋转或者被回收再次启动，立刻就能收到最新的数据。
- 数据共享
  如果对应的LiveData是单例的话，就能在app的组件间分享数据。这部分详细的信息可以参考继承LiveData

## MutableLiveData

MutableLiveData是可变的LiveData。LiveData并没有提供可用的方法来更改储存的数据。MutableLiveData可以通过postValue()或者setValue()来编辑数值。区别是`setValue`是在主线程调用，`postValue`在非主线程调用。

### Transformations功能

`LiveData`中数据变换方法有`map()`和`switchMap()`

官方实例

```
val userLiveData: LiveData<User> = UserLiveData()
val userName: LiveData<String> = Transformations.map(userLiveData) {
    user -> "${user.name} ${user.lastName}"
}
```

map()的原理就是基于MediatorLiveData，MediatorLiveData内部会将传递进来的LiveData和Observer封装成内部类，然后放在内部维护的一个Map中。并且自动帮我们完成`observeForever`()和`removeObserver()`

**switchMap LiveData<Y> switchMap (LiveData<X> trigger, Function<X, LiveData<Y>> func)**

用一个`LiveData<X>`的`value`改变来触发另外一个`LiveData<Y>`的获取

```
    private final MutableLiveData<String> addressInput = new MutableLiveData();
    public final LiveData<String> postalCode =
            Transformations.switchMap(addressInput, (address) -> {
                return repository.getPostCode(address);
             });
```



## MediatorLiveData

```
    /**
     * Starts to listen the given {@code source} LiveData, {@code onChanged} observer will be called
     * when {@code source} value was changed.
     * <p>
     * {@code onChanged} callback will be called only when this {@code MediatorLiveData} is active.
     * <p> If the given LiveData is already added as a source but with a different Observer,
     * {@link IllegalArgumentException} will be thrown.
     *
     * @param source    the {@code LiveData} to listen to
     * @param onChanged The observer that will receive the events
     * @param <S>       The type of data hold by {@code source} LiveData
     */
    @MainThread
    public <S> void addSource(LiveData<S> source, Observer<S> onChanged) {
        //新建一个Source并且将该Source的Observer传进去
        Source<S> e = new Source<>(source, onChanged);
        //检查这个Source是否存在
        Source<?> existing = mSources.putIfAbsent(source, e);
        //如果存在且这个Source的Observer不等于新传进来的Observer就会报错
        if (existing != null && existing.mObserver != onChanged) {
            throw new IllegalArgumentException(
                    "This source was already added with the different observer");
        }
        if (existing != null) {//如果存在直接return
            return;
        }
        if (hasActiveObservers()) {//不存在就插入(plug)
            e.plug();
        }
    }
    
    
        void plug() {
            mLiveData.observeForever(mObserver);
            //observeForever()这个方法不会自动移除
            //需要手动停止实际它内部调用的observe(ALWAYS_ON, observer);
        }

        void unplug() {
            mLiveData.removeObserver(mObserver);
        }
```

从注释来看，addSource()是add一个LiveData对象作为一个source,同时add一个Observer对象来监听这个LiveData的值的变化，如果有变化则会在onChange()里回调。
 并且仅当这个MediatorLiveData处于active时Observer的onChange()才会回调。

**如果这个LiveData已经被add作为一个source,但是这个source没有被remove的情况下，再次调用addSource()并且传了同一个LiveData和一个不同的Observer就会报非法数据异常**

同时要注意`observeForever` 需要在主线程`assertMainThread`

```
 @MainThread
    public void observeForever(@NonNull Observer<? super T> observer) {
        assertMainThread("observeForever");
        AlwaysActiveObserver wrapper = new AlwaysActiveObserver(observer);
        ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
        if (existing != null && existing instanceof LiveData.LifecycleBoundObserver) {
            throw new IllegalArgumentException("Cannot add the same observer"
                    + " with different lifecycles");
        }
        if (existing != null) {
            return;
        }
        wrapper.activeStateChanged(true);
    }
```

**addSource讲解**

```
   testModel.getResult().observe(this, result -> { 
    //注册观察者,注意这个必须得注册，否则ViewModel中的MediatorLiveData就不处于onActive()状态。
       Timber.e("result ="+result.toString());
   });
```

然后

```
 public void setQuery(@Nonnull String originalInput){
        testLive.setValue(originalInput);
        result.addSource(testLive, str -> {
            Timber.e("addSource1执行了");
            result.removeSource(testLive);//先移除
            if(str == null){
                Timber.e("str == null");
            } else {
                result.addSource(testLive, newNumber -> result.setValue("成功"));
                  //双层嵌套，前提是前面有removeSource
            }
        });
        testLive.setValue("test");//注意这里和remove就是使用双层嵌套的原因
    }
```

则执行结果

```
addSource1执行了
result =成功
result =成功
```

观察testLive的变化，testLive#setValue执行了两次，所以addSource里会回调两次，因为removeSource，所以

"addSource1执行了",执行了一次

再把代码改成如下

```
public void setQuery(@Nonnull String originalInput){
        testLive.setValue(originalInput);
        result.addSource(testLive, str -> {
            Timber.e("addSource1执行了");
            result.removeSource(testLive);//先移除
            if(str == null){
                Timber.e("str == null");
            } else {
                result.setValue("成功咯");
            }
        });
        testLive.setValue("test");//注意这里和remove就是使用双层嵌套的原因
    }
```

由于removeSource了所以addSource只执行一次,输出结果

```
addSource1执行了
result =成功
```

------

