---
title: OkHttp源码分析
tags:
    - OkHttp
    - 源码分析
    - 网络框架
categories: Android
---

### OkHttp请求流程

```java
// 创建 OkHttpClient 对象
OkHttpClient okHttpClient = new OkHttpClient().newBuilder().build();
// 创建 Request 对象
Request request = new Request.Builder()
        .url(url)
        .build();
// 同步发起请求
Response execute = okHttpClient.newCall(request).execute();
// 异步发起请求
okHttpClient.newCall(request).enqueue(new Callback() {
    @Override
    public void onFailure(Call call, IOException e) {

    }
    @Override
    public void onResponse(Call call, Response response) throws IOException {

    }
});
```

#### 创建 OkHttpClient 对象

```java
OkHttpClient client = new OkHttpClient();

public OkHttpClient() {
    this(new Builder());
}

// 建造者模式
OkHttpClient(Builder builder) {
    // 自定义拦截器
    // 自定义连接池
    ....
}
```

#### 创建 Request 对象

```java
// 建造者模式
Request request = new Request.Builder()
    .url(url)
    .build();
```

#### 同步执行请求

```java
Response execute = okHttpClient.newCall(request).execute();

/**
   * Prepares the {@code request} to be executed at some point in the future.
   */
  @Override public Call newCall(Request request) {
    // 实际是通过 RealCall 来执行请求的
    return new RealCall(this, request, false /* for web socket */);
  }
```

##### RealCall#execute()
```java
@Override public Response execute() throws IOException {
    synchronized (this) {
        // 每个 Call 只能执行一次
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    try {
        // 通知 dispatcher 当前请求进入同步请求队列
      client.dispatcher().executed(this);
        // 通过一系列拦截器请求处理和响应处理得到Response 
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } finally {
        // 通知 dispatcher 当前请求执行结束，移除队列
      client.dispatcher().finished(this);
    }
  }
```

##### Dispatcher#executed()
```java
// 正在运行的同步请求队列
private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();

synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }
```

##### RealCall#getResponseWithInterceptorChain()
```java
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    // 创建 okHttpClient 时设置的 interceptors
    interceptors.addAll(client.interceptors());
    // 负责失败重试以及重定向
    interceptors.add(retryAndFollowUpInterceptor);
    // 请求时负责添加header，响应时负责移除header
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    // 负责写入缓存，读取缓存，更新缓存
    interceptors.add(new CacheInterceptor(client.internalCache()));
    // 负责和服务器建立连接
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
        // 创建 okHttpClient设置的 networkInterceptors
      interceptors.addAll(client.networkInterceptors());
    }
    // 负责向服务器发送请求数据，从服务器读取响应数据
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(
        interceptors, null, null, null, 0, originalRequest);
    // 使用责任链模式开启链式调用
    return chain.proceed(originalRequest);
}
```

##### RealInterceptorChain#proceed()
```java
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      Connection connection) throws IOException {

    ....

    // Call the next interceptor in the chain.
    // 实例化下一个拦截器对应的 RealInterceptorChain 对象
    RealInterceptorChain next = new RealInterceptorChain(
        interceptors, streamAllocation, httpCodec, connection, index + 1, request);
    // 获取当前拦截器
    Interceptor interceptor = interceptors.get(index);
    // 调用当前拦截器的 intercept()方法，并传入下一个拦截器的 RealInterceptorChain 对象，最后得到响应
    Response response = interceptor.intercept(next);

    ....

    return response;
}
```

#### 异步执行请求
```java
okHttpClient.newCall(request).enqueue(new Callback() {
    @Override
    public void onFailure(Call call, IOException e) {

    }
    @Override
    public void onResponse(Call call, Response response) throws IOException {

    }
});
```

##### RealCall#enqueue()
```java
@Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    // 通知 dispatcher 添加请求到异步请求队列
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```

##### Dispatcher#enqueue()
```java
// 准备中的异步请求队列
private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();
// 正在运行的异步请求队列
private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

synchronized void enqueue(AsyncCall call) {
    // 如果正在运行的异步请求队列长度小于最大请求数，且call占用的host小于最大host数
    // 就把call放入runningAsyncCalls中，同时使用线程池执行call
    // 否则将call放入readyAsyncCalls等待执行
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
    } else {
      readyAsyncCalls.add(call);
    }
  }
```
##### AsyncCall#execute()
```java
@Override protected void execute() {
      boolean signalledCallback = false;
      try {
        // 通过一系列拦截器请求处理和响应处理得到Response 
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }
```
最后拦截器的链式调用对okhttp做了以下处理：
+ okHttpClient配置的自定义普通拦截器
+ RetryAndFollowUpInterceptor(重试，重定向)
+ BridgeInterceptor(header处理)
+ CacheInterceptor(缓存处理)
+ ConnectInterceptor(服务器连接处理)
+ okHttpClient配置的自定义网络拦截器
+ CallServerInterceptor(向服务器发起请求，读取响应)
        
### 缓存处理 CacheInterceptor

#### CacheInterceptor#intercept()
```java
// okhttp的缓存接口，具体实现其实是Cache类，缓存是通过DiskLruCache对象实现的
final InternalCache cache;

@Override public Response intercept(Chain chain) throws IOException {
    // 根据request获取cache中缓存的response
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;

    long now = System.currentTimeMillis();
    
    // request 判断缓存策略，是否使用网络请求缓存，响应数据缓存或两者都有
    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;

    if (cache != null) {
      cache.trackResponse(strategy);
    }

    if (cacheCandidate != null && cacheResponse == null) {
      closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
    }

    // If we're forbidden from using the network and the cache is insufficient, fail.
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

    // If we don't need the network, we're done.
    // 不执行网络请求，直接返回缓存的响应数据
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }

    Response networkResponse = null;
    try {
        // 调用下一个拦截器, 传入缓存的网络请求获取response
      networkResponse = chain.proceed(networkRequest);
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }

    // If we have a cache response too, then we're doing a conditional get.
    // 如果本地已经存在 cacheResponse，同网络获取的 networkResponse 做比较，决定是否更新 cacheResponse
    if (cacheResponse != null) {
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

    if (HttpHeaders.hasBody(response)) {
        // 缓存 之前没有缓存的 response
      CacheRequest cacheRequest = maybeCache(response, networkResponse.request(), cache);
      response = cacheWritingResponse(cacheRequest, response);
    }

    return response;
  }
```

#### CacheInterceptor#maybeCache()
```java
private CacheRequest maybeCache(Response userResponse, Request networkRequest,
      InternalCache responseCache) throws IOException {
    if (responseCache == null) return null;

    // Should we cache this response for this request?
    if (!CacheStrategy.isCacheable(userResponse, networkRequest)) {
      if (HttpMethod.invalidatesCache(networkRequest.method())) {
        try {
          responseCache.remove(networkRequest);
        } catch (IOException ignored) {
          // The cache cannot be written.
        }
      }
      return null;
    }

    // Offer this request to the cache.
    return responseCache.put(userResponse);
  }
```
缓存拦截器会根据网络请求的信息和缓存响应的信息来判断是否存在缓存可用，如果有可以使用的缓存，那么就返回该缓存给用户，否则就继续使用责任链模式来从服务器中获取响应。当获取到响应的时候，又会把响应缓存到磁盘上。

### 连接处理 ConnectInterceptor

#### 连接复用

##### ConnectInterceptor#intercept()
```java
@Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    StreamAllocation streamAllocation = realChain.streamAllocation();

    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    // HttpCodec 负责 HTTP 协议的封装和解析，有两个实现：Http1Codec 和 Http2Codec 分别对应 HTTP/1.1 和 HTTP/2
    HttpCodec httpCodec = streamAllocation.newStream(client, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
```

##### StreamAllocation#newStream()
```java
// 实现连接池的复用处理
public HttpCodec newStream(OkHttpClient client, boolean doExtensiveHealthChecks) {
    int connectTimeout = client.connectTimeoutMillis();
    int readTimeout = client.readTimeoutMillis();
    int writeTimeout = client.writeTimeoutMillis();
    boolean connectionRetryEnabled = client.retryOnConnectionFailure();

    try {
      RealConnection resultConnection = findHealthyConnection(connectTimeout, readTimeout,
          writeTimeout, connectionRetryEnabled, doExtensiveHealthChecks);
      HttpCodec resultCodec = resultConnection.newCodec(client, this);

      synchronized (connectionPool) {
        codec = resultCodec;
        return resultCodec;
      }
    } catch (IOException e) {
      throw new RouteException(e);
    }
  }
```

##### StreamAllocation#findHealthyConnection()
```java
/**
   * Finds a connection and returns it if it is healthy. If it is unhealthy the process is repeated
   * until a healthy connection is found.
   */
  private RealConnection findHealthyConnection(int connectTimeout, int readTimeout,
      int writeTimeout, boolean connectionRetryEnabled, boolean doExtensiveHealthChecks)
      throws IOException {
    while (true) {
        // 循环查找可用的连接
      RealConnection candidate = findConnection(connectTimeout, readTimeout, writeTimeout,
          connectionRetryEnabled);

      // If this is a brand new connection, we can skip the extensive health checks.
      synchronized (connectionPool) {
        if (candidate.successCount == 0) {
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

##### StreamAllocation#findConnection()
```java
/**
   * Returns a connection to host a new stream. This prefers the existing connection if it exists,
   * then the pool, finally building a new connection.
   */
private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      boolean connectionRetryEnabled) throws IOException {
    Route selectedRoute;
    synchronized (connectionPool) {
      if (released) throw new IllegalStateException("released");
      if (codec != null) throw new IllegalStateException("codec != null");
      if (canceled) throw new IOException("Canceled");

      // Attempt to use an already-allocated connection.
        // 尝试使用已分配的连接，如果连接可以创建新的流，就直接返回
      RealConnection allocatedConnection = this.connection;
      if (allocatedConnection != null && !allocatedConnection.noNewStreams) {
        return allocatedConnection;
      }

      // Attempt to get a connection from the pool.
        // 尝试从连接池中获取一个连接，如果连接不为空，就直接返回
      Internal.instance.get(connectionPool, address, this);
      if (connection != null) {
        return connection;
      }

      selectedRoute = route;
    }

    // If we need a route, make one. This is a blocking operation.
    if (selectedRoute == null) {
      selectedRoute = routeSelector.next();
    }

    // Create a connection and assign it to this allocation immediately. This makes it possible for
    // an asynchronous cancel() to interrupt the handshake we're about to do.
    RealConnection result;
    synchronized (connectionPool) {
      route = selectedRoute;
      refusedStreamCount = 0;
        // 创建一个新的连接，并立即分配
      result = new RealConnection(connectionPool, selectedRoute);
      acquire(result);
      if (canceled) throw new IOException("Canceled");
    }

    // Do TCP + TLS handshakes. This is a blocking operation.
    // 进行 TCP 和 TLS 握手
    result.connect(connectTimeout, readTimeout, writeTimeout, connectionRetryEnabled);
    routeDatabase().connected(result.route());

    Socket socket = null;
    synchronized (connectionPool) {
      // Pool the connection.
        // 将该连接放入连接池中
      Internal.instance.put(connectionPool, result);

      // If another multiplexed connection to the same address was created concurrently, then
      // release this connection and acquire that one.
        // 如果同时创建了另一个到同一地址的多路复用连接，则释放这个连接并获取那个连接。
      if (result.isMultiplexed()) {
        socket = Internal.instance.deduplicate(connectionPool, address, this);
        result = connection;
      }
    }
    closeQuietly(socket);

    return result;
  }
```
以上源码分析可知：

StreamAllocation 对象，它相当于一个管理类，维护了服务器连接、并发流和请求之间的关系，该类还会初始化一个 Socket 连接对象，获取输入/输出流对象。

+ 判断当前的连接是否可以使用：流是否关闭，且已经被限制创建新的流
+ 如果当前de连接无法使用，就从连接池中获取一个连接
+ 如果连接池中没有可用的连接，就创建一个新的连接，进行握手，然后放入连接池中

连接复用的好处：省去了进行 TCP 和 TLS 握手的一个过程（因为建立连接需要消耗一些时间，连接复用后可以提升网络访问的效率）

那么，ConnectionPool是如何实现连接管理的？

#### 连接管理

OkHttp 的缓存管理分成两个步骤，一边当我们创建了一个新的连接的时候，我们要把它放进连接池里面；另一边，我们还要来对连接池进行清理。
在 ConnectionPool 中，当我们向连接池中缓存一个连接的时候，只要调用双端队列的 add() 方法，将其加入到双端队列即可，而清理连接缓存的操作则交给线程池来定时执行。
```java
// 双端队列的设计意义是什么？
private final Deque<RealConnection> connections = new ArrayDeque<>();

void put(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (!cleanupRunning) {
      cleanupRunning = true;
        // 使用线程池执行清理任务
      executor.execute(cleanupRunnable);
    }
    // 将新建的连接插入到双端队列中
    connections.add(connection);
  }

private final Runnable cleanupRunnable = new Runnable() {
    @Override public void run() {
      while (true) {
        // 调用 cleanup()方法清理无效的连接
        long waitNanos = cleanup(System.nanoTime());
        if (waitNanos == -1) return;
        if (waitNanos > 0) {
          long waitMillis = waitNanos / 1000000L;
          waitNanos -= (waitMillis * 1000000L);
          synchronized (ConnectionPool.this) {
            try {
              ConnectionPool.this.wait(waitMillis, (int) waitNanos);
            } catch (InterruptedException ignored) {
            }
          }
        }
      }
    }
  };

private final int maxIdleConnections; // 最大闲置连接数，默认是5个，可以通过传参配置
private final long keepAliveDurationNs; // 最长连接存活时间，默认是5分钟，可以通过传参配置

long cleanup(long now) {
    int inUseConnectionCount = 0;
    int idleConnectionCount = 0;
    RealConnection longestIdleConnection = null;
    long longestIdleDurationNs = Long.MIN_VALUE;

    // Find either a connection to evict, or the time that the next eviction is due.
    synchronized (this) {
        // 遍历连接池
      for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
        RealConnection connection = i.next();

        // If the connection is in use, keep searching.
        // 如果当前连接正在使用，继续遍历
        if (pruneAndGetAllocationCount(connection, now) > 0) {
          inUseConnectionCount++;
          continue;
        }

        idleConnectionCount++;

        // If the connection is ready to be evicted, we're done.
        // 如果找到了一个可以被清理的连接，对比最长闲置时间，寻找闲置最久的连接进行释放
        long idleDurationNs = now - connection.idleAtNanos;
        if (idleDurationNs > longestIdleDurationNs) {
          longestIdleDurationNs = idleDurationNs;
          longestIdleConnection = connection;
        }
      }

        // 
      if (longestIdleDurationNs >= this.keepAliveDurationNs
          || idleConnectionCount > this.maxIdleConnections) {
        // We've found a connection to evict. Remove it from the list, then close it below (outside
        // of the synchronized block).
        // 该连接的时长超出了最长闲置时间或闲置的连接数量超出了最大闲置数量，直接移除
        connections.remove(longestIdleConnection);
      } else if (idleConnectionCount > 0) {
        // A connection will be ready to evict soon.
        // 闲置的连接数量大于0，等待一段时间
        return keepAliveDurationNs - longestIdleDurationNs;
      } else if (inUseConnectionCount > 0) {
        // All connections are in use. It'll be at least the keep alive duration 'til we run again.
        // 所有连接都在使用中，等待最长闲置时间
        return keepAliveDurationNs;
      } else {
        // No connections, idle or in use.
        // 没有连接
        cleanupRunning = false;
        return -1;
      }
    }

    closeQuietly(longestIdleConnection.socket());

    // Cleanup again immediately.
    return 0;
  }
```
首先会对缓存中的连接进行遍历，以寻找一个闲置时间最长的连接，然后根据该连接的闲置时长和最大允许的连接数量等参数来决定是否应该清理该连接。
同时注意上面的方法的返回值是一个时间，如果闲置时间最长的连接仍然需要一段时间才能被清理的时候，会返回这段时间的时间差，然后会在这段时间之后再次对连接池进行清理。

### 总结


参考：  
https://jsonchao.github.io/2018/12/01/Android主流三方库源码分析（一、深入理解OKHttp源码）/