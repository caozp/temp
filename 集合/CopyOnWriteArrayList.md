#  CopyOnWriteArrayList

并发包中List只有`CopyOnWriteArrayList`，它是线程安全的。

`CopyOnWrite`容器即写时复制的容器。通俗的理解是当我们往一个容器添加元素的时候，不直接往当前容器添加，而是先将当前容器进行Copy，复制出一个新的容器，然后新的容器里添加元素，添加完元素之后，再将原容器的引用指向新的容器。这样做的好处是我们可以对CopyOnWrite容器进行并发的读，而不需要加锁，因为当前容器不会添加任何元素。所以CopyOnWrite容器也是一种读写分离的思想，读和写不同的容器。



在类图中，每个`CopyOnWriteArrayList`对象里有一个array数组对象用来存放具体元素，ReentrantLock独占锁对象来保证只有一个线程对array进行修改

## 写操作

可以发现在添加的时候是需要加锁的，否则多线程写的时候会Copy出N个副本出来

```
    public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
    }
```

## 读操作

读的时候不需要加锁，如果读的时候有多个线程正在向CopyOnWriteArrayList添加数据，读还是会读到旧的数据

```
public E get(int index) {
    return get(getArray(), index);
}
```

例如，当线程 x 调用 get 方法获取指定位置的元素时。

1. 步骤A:首先获取 array数组
2. 步骤B:获取index位置的元素，假设array数组为1，2，3。

由于执行步骤 A 和步骤 B 没有加锁。可能导致在线程 x执行完步骤 A后执行步 骤 B 前，另外一给线程 y 进行了 remove 操作，假设要删除元素 1 。而这个时候线程x还在使用array，步骤B操作的还是原来的数组。访问index为0的元素还是1，而不是2。

所以不能用于**实时读**的场景。



## 内存问题

由于写操作的时候，需要拷贝数组，会消耗内存，如果原数组的内容比较多的情况下，可能导致GC

