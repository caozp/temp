为了解决TCP握手和挥手的效率问题，http有keepalive机制。也就是可以在一次TCP连接中可以持续发送多份数据而不会断开连接。所以复用显得比较重要。而复用连接就需要对连接进行管理，于是有了连接池的概念。

Okhttp中使用ConnectionPool实现连接池，默认支持5个并发keepalive，默认链路生命为5分钟。



## 连接池的创建

ConnectionPool默认是在OkhttpClient.Builder中创建

```
    public Builder() {
      ...
      connectionPool = new ConnectionPool();
      ...
    }
```



## 引用计数

使用了类似于引用计数的方式跟踪Socket流的调用，这里的计数对象是`StreamAllocation`，它被反复执行`aquire`与`release`操作，这两个函数其实是在改变`RealConnection`中的`List<Reference<StreamAllocation>>` 的大小。

`RealConnection`是`socket`物理连接的包装，它里面维护了`List<Reference<StreamAllocation>>`的引用。List中`StreamAllocation`的数量也就是`socket`被引用的计数，如果计数为0的话，说明此连接没有被使用就是空闲的，需要通过下文的算法实现回收；如果计数不为0，则表示上层代码仍然引用，就不需要关闭连接。

```
  public void acquire(RealConnection connection, boolean reportedAcquired) {
    assert (Thread.holdsLock(connectionPool));
    if (this.connection != null) throw new IllegalStateException();

    this.connection = connection;
    this.reportedAcquired = reportedAcquired;
    connection.allocations.add(new StreamAllocationReference(this, callStackTrace));
  }

  /** Remove this allocation from the connection's list of allocations. */
  private void release(RealConnection connection) {
    for (int i = 0, size = connection.allocations.size(); i < size; i++) {
      Reference<StreamAllocation> reference = connection.allocations.get(i);
      if (reference.get() == this) {
        connection.allocations.remove(i);
        return;
      }
    }
    throw new IllegalStateException();
  }
```



## 寻找连接

在连接拦截器中，会调用streamAllocation.newStream方法

```
HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
```

看下它的具体方法,它会调用`findHealthyConnection`方法`RealConnection`

```
public HttpCodec newStream(
      OkHttpClient client, Interceptor.Chain chain, boolean doExtensiveHealthChecks) {
    int connectTimeout = chain.connectTimeoutMillis();
    int readTimeout = chain.readTimeoutMillis();
    int writeTimeout = chain.writeTimeoutMillis();
    int pingIntervalMillis = client.pingIntervalMillis();
    boolean connectionRetryEnabled = client.retryOnConnectionFailure();

    try {
      RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
          writeTimeout, pingIntervalMillis, connectionRetryEnabled, doExtensiveHealthChecks);
      HttpCodec resultCodec = resultConnection.newCodec(client, chain, this);

      synchronized (connectionPool) {
        codec = resultCodec;
        return resultCodec;
      }
    } catch (IOException e) {
      throw new RouteException(e);
    }
  }
```



而`findHealthyConnection` 通过`findConnection`去找一个连接。

```
  private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
      int writeTimeout, int pingIntervalMillis, boolean connectionRetryEnabled,
      boolean doExtensiveHealthChecks) throws IOException {
    while (true) {
      RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
          pingIntervalMillis, connectionRetryEnabled);

      // If this is a brand new connection, we can skip the extensive health checks.
      synchronized (connectionPool) {
        if (candidate.successCount == 0) { //表明这是个新的连接，不用检查了 直接返回
          return candidate;
        }
      }

      // Do a (potentially slow) check to confirm that the pooled connection is still good. If it
      // isn't, take it out of the pool and start again.
      if (!candidate.isHealthy(doExtensiveHealthChecks)) {
        noNewStreams();
        continue;
      }

      return candidate; 
    }
  }
```

如果这个连接时新建的，就直接返回。

如果这个连接是从连接池中取出来的，检查该连接是否可用，如果不可用，移除，并继续寻找。

最后的寻找逻辑来到`findConnection`中





```
private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      int pingIntervalMillis, boolean connectionRetryEnabled) throws IOException {
    boolean foundPooledConnection = false;
    RealConnection result = null;
    Route selectedRoute = null;
    Connection releasedConnection;
    Socket toClose;
    synchronized (connectionPool) {
      if (released) throw new IllegalStateException("released");
      if (codec != null) throw new IllegalStateException("codec != null");
      if (canceled) throw new IOException("Canceled");

      // Attempt to use an already-allocated connection. We need to be careful here because our
      // already-allocated connection may have been restricted from creating new streams.
      releasedConnection = this.connection;
      toClose = releaseIfNoNewStreams();
      if (this.connection != null) {
        // We had an already-allocated connection and it's good.
        result = this.connection;
        releasedConnection = null;
      }
      if (!reportedAcquired) {
        // If the connection was never reported acquired, don't report it as released!
        releasedConnection = null;
      }

      if (result == null) {
        // Attempt to get a connection from the pool.
        // 试图从连接池中获取一个连接
        Internal.instance.acquire(connectionPool, address, this, null);
        if (connection != null) {
          foundPooledConnection = true;
          result = connection;
        } else {
          selectedRoute = route;
        }
      }
    }
    closeQuietly(toClose);

    if (releasedConnection != null) {
      eventListener.connectionReleased(call, releasedConnection);
    }
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
    }
    if (result != null) {
      // If we found an already-allocated or pooled connection, we're done.
      return result;
    }

    // If we need a route selection, make one. This is a blocking operation.
    // 获取路由配置，所谓路由其实就是代理，ip地址等参数的一个组合
    boolean newRouteSelection = false;
    if (selectedRoute == null && (routeSelection == null || !routeSelection.hasNext())) {
      newRouteSelection = true;
      routeSelection = routeSelector.next();
    }

    synchronized (connectionPool) {
      if (canceled) throw new IOException("Canceled");

      if (newRouteSelection) {
        // Now that we have a set of IP addresses, make another attempt at getting a connection from
        // the pool. This could match due to connection coalescing.
        List<Route> routes = routeSelection.getAll();
        for (int i = 0, size = routes.size(); i < size; i++) {
          Route route = routes.get(i);
          Internal.instance.acquire(connectionPool, address, this, route);
          if (connection != null) {
            foundPooledConnection = true;
            result = connection;
            this.route = route;
            break;
          }
        }
      }

      if (!foundPooledConnection) {
        if (selectedRoute == null) {
          selectedRoute = routeSelection.next();
        }

        // Create a connection and assign it to this allocation immediately. This makes it possible
        // for an asynchronous cancel() to interrupt the handshake we're about to do.
        route = selectedRoute;
        refusedStreamCount = 0;
        result = new RealConnection(connectionPool, selectedRoute);
        acquire(result, false);
      }
    }

    // If we found a pooled connection on the 2nd time around, we're done.
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
      return result;
    }

    // Do TCP + TLS handshakes. This is a blocking operation.
    result.connect(connectTimeout, readTimeout, writeTimeout, pingIntervalMillis,
        connectionRetryEnabled, call, eventListener);
    routeDatabase().connected(result.route());

    Socket socket = null;
    synchronized (connectionPool) {
      reportedAcquired = true;
      // 将新建的连接放入到连接池中
      // Pool the connection.
      Internal.instance.put(connectionPool, result);

      // If another multiplexed connection to the same address was created concurrently, then
      // release this connection and acquire that one.
       // 如果同时存在多个连向同一个地址的多路复用连接，则关闭多余连接，只保留一个
      if (result.isMultiplexed()) {
        socket = Internal.instance.deduplicate(connectionPool, address, this);
        result = connection;
      }
    }
    closeQuietly(socket);

    eventListener.connectionAcquired(call, result);
    return result;
  }
```

其主要逻辑大致分为以下几个步骤：

- 查看当前streamAllocation是否有之前已经分配过的连接，有则直接使用
- 从连接池中查找可复用的连接，有则返回该连接
- 配置路由，配置后再次从连接池中查找是否有可复用连接，有则直接返回
- 新建一个连接，并修改其StreamAllocation标记计数，将其放入连接池中
- 查看连接池是否有重复的多路复用连接，有则清除



## 回收

连接池维护中双端队列Deque,线程池executor。

连接池提供对Deque进行put，get，connectionBecameIdle和evictAll的操作，分别是放入，获取，移除和移除所有连接的操作

```
void put(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (!cleanupRunning) {
      cleanupRunning = true;
      executor.execute(cleanupRunnable);
    }
    connections.add(connection);
  }
  
   RealConnection get(Address address, StreamAllocation streamAllocation) {
    assert (Thread.holdsLock(this));
    for (RealConnection connection : connections) {
      if (connection.allocations.size() < connection.allocationLimit
          && address.equals(connection.route().address)
          && !connection.noNewStreams) {
        streamAllocation.acquire(connection);
        return connection;
      }
    }
    return null;
  }
```



其中当put时，会去触发清理任务

```
while (true) {
  //执行清理并返回下场需要清理的时间
  long waitNanos = cleanup(System.nanoTime());
  if (waitNanos == -1) return;
  if (waitNanos > 0) {
    synchronized (ConnectionPool.this) {
      try {
        //在timeout内释放锁与时间片
        ConnectionPool.this.wait(TimeUnit.NANOSECONDS.toMillis(waitNanos));
      } catch (InterruptedException ignored) {
      }
    }
  }
}
```



这段死循环实际上是一个阻塞的清理任务，首先进行清理(clean)，并返回下次需要清理的间隔时间，然后调用`wait(timeout)`进行等待以释放锁与时间片，当等待时间到了后，再次进行清理，并返回下次要清理的间隔时间...

OkHttp的连接池通过计数+标记清理的机制来管理连接池，使得无用连接可以被会回收，并保持多个健康的keep-alive连接。这也是OkHttp的连接池能保持高效的关键原因。

其基本逻辑如下：

- 遍历连接池中所有连接，标记泄露连接
- 如果被标记的连接满足(`空闲socket连接超过5个`&&`keepalive时间大于5分钟`)，就将此连接从`Deque`中移除，并关闭连接，返回`0`，也就是将要执行`wait(0)`，提醒立刻再次扫描
- 如果(`目前还可以塞得下5个连接，但是有可能泄漏的连接(即空闲时间即将达到5分钟)`)，就返回此连接即将到期的剩余时间，供下次清理
- 如果(`全部都是活跃的连接`)，就返回默认的`keep-alive`时间，也就是5分钟后再执行清理