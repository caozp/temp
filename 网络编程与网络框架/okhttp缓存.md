# Http缓存

[参考](https://mp.weixin.qq.com/s/Fazx13maQfPJItfkOqk9FQ)



## Okhttp缓存

我们知道OkHttp的请求在发送到服务器之前会经过一系列的Interceptor，其中有一个CacheInterceptor即是我们需要分析的代码。

```
@Override public Response intercept(Chain chain) throws IOException {
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;

    long now = System.currentTimeMillis();

    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;

    ......

 }
```

方法首先通过InternalCache 获取到对应请求的缓存。这里我们不展开讨论这个类的具体实现，只需要知道，如果之前缓存了该请求url的资源，那么通过request对象可以查找到这个缓存响应。

将获取到的缓存响应，当前时间戳和请求传入CacheStrategy，然后通过执行get方法执行一些逻辑最终可以获取到strategy.networkRequest,strategy.cacheResponse。如果通过CacheStrategy的判断之后，我们发现这次请求无法直接使用缓存数据，需要向服务器发起请求，那么我们就通过CacheStrategy为我们构造的networkRequest来发起这次请求。我们先来看看CacheStrategy做了哪些事情

```
CacheStrategy.Factory.java

public Factory(long nowMillis, Request request, Response cacheResponse) {
      this.nowMillis = nowMillis;
      this.request = request;
      this.cacheResponse = cacheResponse;

      if (cacheResponse != null) {
        this.sentRequestMillis = cacheResponse.sentRequestAtMillis();
        this.receivedResponseMillis = cacheResponse.receivedResponseAtMillis();
        Headers headers = cacheResponse.headers();
        for (int i = 0, size = headers.size(); i < size; i++) {
          String fieldName = headers.name(i);
          String value = headers.value(i);
          if ("Date".equalsIgnoreCase(fieldName)) {
            servedDate = HttpDate.parse(value);
            servedDateString = value;
          } else if ("Expires".equalsIgnoreCase(fieldName)) {
            expires = HttpDate.parse(value);
          } else if ("Last-Modified".equalsIgnoreCase(fieldName)) {
            lastModified = HttpDate.parse(value);
            lastModifiedString = value;
          } else if ("ETag".equalsIgnoreCase(fieldName)) {
            etag = value;
          } else if ("Age".equalsIgnoreCase(fieldName)) {
            ageSeconds = HttpHeaders.parseSeconds(value, -1);
          }
        }
      }
    }
```

**CacheStrategy.Factory的构造方法首先保存了传入的参数，并将缓存响应的相关首部解析保存下来。之后调用的get方法如下**

```
    public CacheStrategy get() {
      CacheStrategy candidate = getCandidate();

      if (candidate.networkRequest != null && request.cacheControl().onlyIfCached()) {
        // We're forbidden from using the network and the cache is insufficient.
        return new CacheStrategy(null, null);
      }

      return candidate;
    }
```

get方法很简单，主要逻辑在getCandidate中，这里的逻辑是**如果返回的candidate所持有的networkRequest不为空，表示我们这次请求需要发到服务器**，此时如果请求的cacheControl要求本次请求只使用缓存数据。那么这次请求恐怕只能以失败告终了，这点我们等会儿回到CacheInterceptor中可以看到。接着我们看看主要getCandidate的主要逻辑。

```
    private CacheStrategy getCandidate() {
      // No cached response.
      if (cacheResponse == null) {
        return new CacheStrategy(request, null);
      }

      // Drop the cached response if it's missing a required handshake.
      if (request.isHttps() && cacheResponse.handshake() == null) {
        return new CacheStrategy(request, null);
      }

      // If this response shouldn't have been stored, it should never be used
      // as a response source. This check should be redundant as long as the
      // persistence store is well-behaved and the rules are constant.
      if (!isCacheable(cacheResponse, request)) {
        return new CacheStrategy(request, null);
      }

      CacheControl requestCaching = request.cacheControl();
      if (requestCaching.noCache() || hasConditions(request)) {
        return new CacheStrategy(request, null);
      }
        ......
    }
```

上面这段代码主要列出四种情况下需要忽略缓存，直接想服务器发起请求的情况：

1. 缓存本身不存在
2. 请求是采用https 并且缓存没有进行握手的数据。
3. 缓存本身不应该不保存下来。可能是缓存本身实现有问题，把一些不应该缓存的数据保留了下来。
4. 如果请求本身添加了 Cache-Control: No-Cache，或是一些条件请求首部，说明请求不希望使用缓存数据。

这些情况下直接构造一个包含networkRequest，但是cacheResponse为空的CacheStrategy对象返回。

```
 private CacheStrategy getCandidate() {
      ......

      CacheControl responseCaching = cacheResponse.cacheControl();
      if (responseCaching.immutable()) {
        return new CacheStrategy(null, cacheResponse);
      }

      long ageMillis = cacheResponseAge();
      long freshMillis = computeFreshnessLifetime();

      if (requestCaching.maxAgeSeconds() != -1) {
        freshMillis = Math.min(freshMillis, SECONDS.toMillis(requestCaching.maxAgeSeconds()));
      }

      long minFreshMillis = 0;
      if (requestCaching.minFreshSeconds() != -1) {
        minFreshMillis = SECONDS.toMillis(requestCaching.minFreshSeconds());
      }

      long maxStaleMillis = 0;
      if (!responseCaching.mustRevalidate() && requestCaching.maxStaleSeconds() != -1) {
        maxStaleMillis = SECONDS.toMillis(requestCaching.maxStaleSeconds());
      }




      if (!responseCaching.noCache() && ageMillis + minFreshMillis < freshMillis + maxStaleMillis) {
        Response.Builder builder = cacheResponse.newBuilder();
        if (ageMillis + minFreshMillis >= freshMillis) {
          builder.addHeader("Warning", "110 HttpURLConnection \"Response is stale\"");
        }
        long oneDayMillis = 24 * 60 * 60 * 1000L;
        if (ageMillis > oneDayMillis && isFreshnessLifetimeHeuristic()) {
          builder.addHeader("Warning", "113 HttpURLConnection \"Heuristic expiration\"");
        }
        return new CacheStrategy(null, builder.build());
      }

        ......     
    }
```

如果缓存响应的Cache-Control首部包含immutable,那么说明该资源不会改变。客户端可以直接使用缓存结果。值得注意的是immutable并不属于http协议的一部分，而是由facebook提出的扩展属性。

之后分别计算ageMills、freshMills、minFreshMills、maxStaleMills这四个值。 
如果响应缓存没有通过Cache-Control:No-Cache 来禁止客户端使用缓存，并且

```
ageMillis + minFreshMillis < freshMillis + maxStaleMillis
```

这个不等式成立，那么我们进入条件代码块之后最终会返回networkRequest为空，并且使用当前缓存值构造的CacheStrtegy。

这个不等式究竟是什么含义呢？我们看看这四个值分别代表什么。  

**ageMills** 指这个缓存资源自响应报文在源服务器中产生或者过期验证的那一刻起，到现在为止所经过的时间。用食品的保质期来比喻的话，好比当前时间距离生产日期已经过去了多久了。  

**freshMills** 表示这个资源在多少时间内是新鲜的。也就是假设保质期18个月，那么这个18个月就是freshMills。 

**minFreshMills** 表示我希望这个缓存至少在多久之后依然是新鲜的。好比我是一个比较讲究的人，如果某个食品只有一个月就过期了，虽然并没有真的过期，但我依然觉得食品不新鲜从而不想再吃了。  

**maxStaleMills** 好比我是一个不那么讲究的人，即使食品已经过期了，只要不是过期很久了，比如2个月，那我觉得问题不大，还可以吃。

minFreshMills 和maxStatleMills都是由请求首部取出的，请求可以根据自己的需要，通过设置

```
Cache-Control:min-fresh=xxx、Cache-Control:max-statle=xxx
```

来控制缓存，以达到对缓存使用严格性的收紧与放松

```
    private CacheStrategy getCandidate() {
        ......

      // Find a condition to add to the request. If the condition is satisfied, the response body
      // will not be transmitted.
      String conditionName;
      String conditionValue;
      if (etag != null) {
        conditionName = "If-None-Match";
        conditionValue = etag;
      } else if (lastModified != null) {
        conditionName = "If-Modified-Since";
        conditionValue = lastModifiedString;
      } else if (servedDate != null) {
        conditionName = "If-Modified-Since";
        conditionValue = servedDateString;
      } else {
        return new CacheStrategy(request, null); // No condition! Make a regular request.
      }

      Headers.Builder conditionalRequestHeaders = request.headers().newBuilder();
      Internal.instance.addLenient(conditionalRequestHeaders, conditionName, conditionValue);

      Request conditionalRequest = request.newBuilder()
          .headers(conditionalRequestHeaders.build())
          .build();
      return new CacheStrategy(conditionalRequest, cacheResponse);
    }
```

如果之前的条件不满足，说明我们的缓存响应已经过期了，这时我们需要通过一个条件请求对服务器进行再验证操作。接下来的代码比较清晰来，就是通过从缓存响应中取出的`Last-Modified`,`Etag`,`Date`首部构造一个条件请求并返回。

接下来我们返回CacheInterceptor

```
// If we're forbidden from using the network and the cache is insufficient, fail.
// 如果不能使用网络，同时又没有符合条件的缓存，直接抛504错误
    if (networkRequest == null && cacheResponse == null) {
      return new Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(Util.EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();
    }
```

可以看到，如果我们返回的`networkRequest`和`cacheResponse`都为空，说明我们即没有可用的缓存，同时请求通过`Cache-Control:only-if-cached`只允许我们使用当前的缓存数据。这个时候我们只能返回一个504的响应。接着往下看，

```
    // If we don't need the network, we're done.
    // 如果有缓存同时又不使用网络，则直接返回缓存结果
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }
```

如果networkRequest为空，说明我们不需要进行再验证了，直接将cacheResponse作为请求结果返回。

```
Response networkResponse = null;
    try {
      //尝试通过网络获取回复
      networkResponse = chain.proceed(networkRequest);
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }

    // If we have a cache response too, then we're doing a conditional get.
    if (cacheResponse != null) {
    // 如果服务端返回的是NOT_MODIFIED,缓存有效，将本地缓存和网络响应做合并
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();

        // Update the cache after combining headers but before stripping the
        // Content-Encoding header (as performed by initContentStream()).
        cache.trackConditionalCacheHit();
        cache.update(cacheResponse, response);
        return response;
      } else {
        closeQuietly(cacheResponse.body());
      }
    }


     Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    if (cache != null) {
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
        // Offer this request to the cache.
        CacheRequest cacheRequest = cache.put(response);
        return cacheWritingResponse(cacheRequest, response);
      }

      if (HttpMethod.invalidatesCache(networkRequest.method())) {
        try {
          cache.remove(networkRequest);
        } catch (IOException ignored) {
          // The cache cannot be written.
        }
      }
    }

    return response;
```

如果networkRequest存在不为空，说明这次请求是需要发到服务器的。此时有两种情况，一种cacheResponse不存在，说明我们没有一个可用的缓存，这次请求只是一个普通的请求。如果cacheResponse存在，说明我们有一个可能过期了的缓存，此时networkRequest是一个用来进行再验证的条件请求。

不管哪种情况，我们都需要通过networkResponse=chain.proceed(networkRequest)获取到服务器的一个响应。不同的只是如果有缓存数据，那么在获取到再验证的响应之后，需要cache.update(cacheResponse, response)去更新当前缓存中的数据。如果没有缓存数据，那么判断此次请求是否可以被缓存。在满足缓存的条件下，将响应缓存下来，并返回。

OkHttp缓存大致的流程就是这样，我们从中看出，整个流程是遵循了Http的缓存流程的。最后我们总结一下缓存的流程：

1. 从接收到的请求中，解析出Url和各个首部。
2. 查询本地是否有缓存副本可以使用。
3. 如果有缓存，则进行新鲜度检测，如果缓存足够新鲜，则使用缓存作为响应返回，如果不够新鲜了，则构造条件请求，发往服务器再验证。如果没有缓存，就直接将请求发往服务器。
4. 把从服务器返回的响应，更新或是新增到缓存中。