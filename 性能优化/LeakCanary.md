# LeakCanary

大名鼎鼎的 square 公司开源的内存泄漏检测工具.

我们知道内存泄漏问题的排查有很多种方法， 比如说，Android Studio 自带的 Profile 工具、MAT(Memory Analyzer Tool)、以及LeakCanary。 选择 LeakCanary 作为首选的内存泄漏检测工具主要是因为它能实时检测泄漏并以非常直观的调用链方式展示内存泄漏的原因。



主要的原理如下

Application类提供了`registerActivityLifecycleCallbacks`和`unregisterActivityLifecycleCallbacks`方法用来注册和反注册Activity的生命周期监听。

同时`fragmentManager`类也提供了`registerFragmentLifecycleCallbacks`和`unregisterFragmentLifecycleCallbacks`方法来注册和反注册 Fragment 的生命周期监听。

再销毁的时候 调用

**refWatcher.watch(）方法**

该方法主要将监视对象key包装成一个弱引用对象

```
public void watch(Object watchedReference, String referenceName) {
  if (this == DISABLED) {
    return;
  }
  checkNotNull(watchedReference, "watchedReference");
  checkNotNull(referenceName, "referenceName");
  final long watchStartNanoTime = System.nanoTime();
  // 对当前监视对象设置一个唯一 id
  String key = UUID.randomUUID().toString();
  // 添加到 Set<String> 中
  retainedKeys.add(key);
  // 构造一个带有id 的 WeakReference 对象
  final KeyedWeakReference reference =
      new KeyedWeakReference(watchedReference, key, referenceName, queue);
  // 检测对象是否被回收了
  ensureGoneAsync(watchStartNanoTime, reference);
}
```

接着，引用检查的任务通过`Looper.myQueue().addIdleHandler`添加到MessageQueue中，在主线程空闲的时候进行引用检查操作

最后，如何检查Activity/fragment是否发生内存泄露呢？

```
Retryable.Result ensureGone(final KeyedWeakReference reference, final long watchStartNanoTime) {
  long gcStartNanoTime = System.nanoTime();
  // 前面不是有一个重试的机制么，这里会计下这次重试距离第一次执行花了多长时间
  long watchDurationMs = NANOSECONDS.toMillis(gcStartNanoTime - watchStartNanoTime);
  // 移除所有弱引用可达对象，后面细讲
  removeWeaklyReachableReferences();
  // 判断当前是否正在开启USB调试，LeakCanary 的解释是调试时可能会触发不正确的内存泄漏
  if (debuggerControl.isDebuggerAttached()) {
    // The debugger can create false leaks.
    return RETRY;
  }
  // 上面执行 removeWeaklyReachableReferences 方法，判断是不是监视对象已经被回收了，如果被回收了，那么说明没有发生内存泄漏，直接结束
  if (gone(reference)) {
    return DONE;
  }
  // 手动触发一次 GC 垃圾回收
  gcTrigger.runGc();
  // 再次移除所有弱引用可达对象
  removeWeaklyReachableReferences();
  // 如果对象没有被回收
  if (!gone(reference)) {
    long startDumpHeap = System.nanoTime();
    long gcDurationMs = NANOSECONDS.toMillis(startDumpHeap - gcStartNanoTime);
    // 使用 Debug 类 dump 当前堆内存中对象使用情况
    File heapDumpFile = heapDumper.dumpHeap();
    // dumpHeap 失败的话，会走重试机制
    if (heapDumpFile == RETRY_LATER) {
      // Could not dump the heap.
      return RETRY;
    }
    long heapDumpDurationMs = NANOSECONDS.toMillis(System.nanoTime() - startDumpHeap);
    // 将hprof文件、key等属性构造一个 HeapDump 对象
    HeapDump heapDump = heapDumpBuilder.heapDumpFile(heapDumpFile).referenceKey(reference.key)
        .referenceName(reference.name)
        .watchDurationMs(watchDurationMs)
        .gcDurationMs(gcDurationMs)
        .heapDumpDurationMs(heapDumpDurationMs)
        .build();
    // heapdumpListener 分析 heapDump 对象
    heapdumpListener.analyze(heapDump);
  }
  return DONE;
}
```

这里先调用`removeWeaklyReachableReferences()`来移除弱引用对象，判断监视对象是否被回收，如果回收，说明没有泄露，结束。如果还存在，手动触发gc，再次调用`removeWeaklyReachableReferences()`。再去检查是否回收，如果还是没有回收掉，生成hprof文件。

`removeWeaklyReachableReferences()`如下。

```
private void removeWeaklyReachableReferences() {
  KeyedWeakReference ref;
  while ((ref = (KeyedWeakReference) queue.poll()) != null) {
    retainedKeys.remove(ref.key);
  }
}
```

为什么是可以这样移除弱引用对象？



弱引用在定义的时候可以指定引用对象和一个 ReferenceQueue，弱引用对象在垃圾回收器执行回收方法时，**如果原对象只有这个弱引用对象引用着，那么会回收原对象，并将弱引用对象加入到 ReferenceQueue**，通过 ReferenceQueue 的 poll 方法，可以取出这个弱引用对象，获取弱引用对象本身的一些信息。

看下面这个例子

```
mReferenceQueue = new ReferenceQueue<>();
// 定义一个对象
o = new Object();
// 定义一个弱引用对象引用 o,并指定引用队列为 mReferenceQueue
weakReference = new WeakReference<Object>(o, mReferenceQueue);
// 去掉强引用
o = null;
// 触发应用进行垃圾回收
Runtime.getRuntime().gc();
// hack: 延时100ms,等待gc完成
try {
    Thread.sleep(100);
} catch (InterruptedException e) {
    e.printStackTrace();
}
Reference ref = null;
// 遍历 mReferenceQueue，取出所有弱引用
while ((ref = mReferenceQueue.poll()) != null) {
    System.out.println("ref in queue");
}
```

可以看到输出 ref in queue。





[LeakCanary 内存泄漏原理完全解析](https://www.jianshu.com/p/59106802b62c)