---
title: Volley源码分析
tags: 
    - Volley
    - 网络框架
    - 源码分析
categories: Android
---

RequestQueue  
------------

###初始化RequestQueue
根据Android版本使用不同的网络请求，版本2.3以上使用httpurlconnection，否则使用httpclient，作为参数创建RequestQueue

调用RequestQueue.start方法，默认后台开启5个线程，1个CacheDispatcher（缓存调度线程），4个NetworkDispatcher（网络调度线程）

###将Request请求添加到RequestQueue请求队列
首先通过执行request.shouldCache()方法判断request是否应该缓存，默认是true，也就是所有的请求默认都要缓存。  
如果request不需要缓存，就把请求放进网络请求队列。  
如果request需要缓存，就把请求放进缓存队列。  


CacheDispatcher
------------

```java
    @Override
    public void run() {
        if (DEBUG) VolleyLog.v("start new dispatcher");
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);

        // Make a blocking call to initialize the cache.
        mCache.initialize();

        while (true) {
            try {
                processRequest();
            } catch (InterruptedException e) {
                // We may have been interrupted because it was time to quit.
                if (mQuit) {
                    Thread.currentThread().interrupt();
                    return;
                }
                VolleyLog.e(
                        "Ignoring spurious interrupt of CacheDispatcher thread; "
                                + "use quit() to terminate it");
            }
        }
    }

    // 不断从缓存队列中取出网络请求，执行processRequest方法

    void processRequest(final Request<?> request) throws InterruptedException {
        request.addMarker("cache-queue-take");

        // If the request has been canceled, don't bother dispatching it.
        if (request.isCanceled()) {
            request.finish("cache-discard-canceled");
            return;
        }

        // Attempt to retrieve this item from cache.
        Cache.Entry entry = mCache.get(request.getCacheKey());
        if (entry == null) {
            request.addMarker("cache-miss");
            // Cache miss; send off to the network dispatcher.
            if (!mWaitingRequestManager.maybeAddToWaitingRequests(request)) {
                mNetworkQueue.put(request);
            }
            return;
        }

        // If it is completely expired, just send it to the network.
        if (entry.isExpired()) {
            request.addMarker("cache-hit-expired");
            request.setCacheEntry(entry);
            if (!mWaitingRequestManager.maybeAddToWaitingRequests(request)) {
                mNetworkQueue.put(request);
            }
            return;
        }

        // We have a cache hit; parse its data for delivery back to the request.
        request.addMarker("cache-hit");
        Response<?> response =
                request.parseNetworkResponse(
                        new NetworkResponse(entry.data, entry.responseHeaders));
        request.addMarker("cache-hit-parsed");

        if (!entry.refreshNeeded()) {
            // Completely unexpired cache hit. Just deliver the response.
            mDelivery.postResponse(request, response);
        } else {
            // Soft-expired cache hit. We can deliver the cached response,
            // but we need to also send the request to the network for
            // refreshing.
            request.addMarker("cache-hit-refresh-needed");
            request.setCacheEntry(entry);
            // Mark the response as intermediate.
            response.intermediate = true;

            if (!mWaitingRequestManager.maybeAddToWaitingRequests(request)) {
                // Post the intermediate response back to the user and have
                // the delivery then forward the request along to the network.
                mDelivery.postResponse(
                        request,
                        response,
                        new Runnable() {
                            @Override
                            public void run() {
                                try {
                                    mNetworkQueue.put(request);
                                } catch (InterruptedException e) {
                                    // Restore the interrupted status
                                    Thread.currentThread().interrupt();
                                }
                            }
                        });
            } else {
                // request has been added to list of waiting requests
                // to receive the network response from the first request once it returns.
                mDelivery.postResponse(request, response);
            }
        }
    }    
```
如果request被取消，就退出该方法  
如果request没被取消，就判断request是否有缓存的response，
如果有且没过期，就判断是否需要刷新，
如果需要，就对缓存的response进行解析并回调给主线程，同时把request放入网络请求队列，等待网络调度线程轮询
如果request没有缓存的response或缓存的response已经过期，就把request放入网络请求队列，等待网络调度线程轮询

NetworkDispatcher
------------

```java
    @Override
    public void run() {
        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
        while (true) {
            try {
                processRequest();
            } catch (InterruptedException e) {
                // We may have been interrupted because it was time to quit.
                if (mQuit) {
                    Thread.currentThread().interrupt();
                    return;
                }
                VolleyLog.e(
                        "Ignoring spurious interrupt of NetworkDispatcher thread; "
                                + "use quit() to terminate it");
            }
        }
    }

    // 不断从网络请求队列中取出网络请求，执行processRequest方法

    void processRequest(Request<?> request) {
        long startTimeMs = SystemClock.elapsedRealtime();
        try {
            request.addMarker("network-queue-take");

            // If the request was cancelled already, do not perform the
            // network request.
            if (request.isCanceled()) {
                request.finish("network-discard-cancelled");
                request.notifyListenerResponseNotUsable();
                return;
            }

            addTrafficStatsTag(request);

            // Perform the network request.
            NetworkResponse networkResponse = mNetwork.performRequest(request);
            request.addMarker("network-http-complete");

            // If the server returned 304 AND we delivered a response already,
            // we're done -- don't deliver a second identical response.
            if (networkResponse.notModified && request.hasHadResponseDelivered()) {
                request.finish("not-modified");
                request.notifyListenerResponseNotUsable();
                return;
            }

            // Parse the response here on the worker thread.
            Response<?> response = request.parseNetworkResponse(networkResponse);
            request.addMarker("network-parse-complete");

            // Write to cache if applicable.
            // TODO: Only update cache metadata instead of entire record for 304s.
            if (request.shouldCache() && response.cacheEntry != null) {
                mCache.put(request.getCacheKey(), response.cacheEntry);
                request.addMarker("network-cache-written");
            }

            // Post the response back.
            request.markDelivered();
            mDelivery.postResponse(request, response);
            request.notifyListenerResponseReceived(response);
        } catch (VolleyError volleyError) {
            volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
            parseAndDeliverNetworkError(request, volleyError);
            request.notifyListenerResponseNotUsable();
        } catch (Exception e) {
            VolleyLog.e(e, "Unhandled exception %s", e.toString());
            VolleyError volleyError = new VolleyError(e);
            volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
            mDelivery.postError(request, volleyError);
            request.notifyListenerResponseNotUsable();
        }
    }
```
如果request被取消，就退出该方法
如果request没被取消，就调用Network(即BasicNetwork)的performRequest方法，把返回的response回调到主线程
```java
    @Override
    public NetworkResponse performRequest(Request<?> request) throws VolleyError {
        long requestStart = SystemClock.elapsedRealtime();
        while (true) {
            HttpResponse httpResponse = null;
            byte[] responseContents = null;
            List<Header> responseHeaders = Collections.emptyList();
            try {
                // Gather headers.
                Map<String, String> additionalRequestHeaders =
                        getCacheHeaders(request.getCacheEntry());
                httpResponse = mBaseHttpStack.executeRequest(request, additionalRequestHeaders);
                int statusCode = httpResponse.getStatusCode();

                responseHeaders = httpResponse.getHeaders();
                // Handle cache validation.
                if (statusCode == HttpURLConnection.HTTP_NOT_MODIFIED) {
                    Entry entry = request.getCacheEntry();
                    if (entry == null) {
                        return new NetworkResponse(
                                HttpURLConnection.HTTP_NOT_MODIFIED,
                                /* data= */ null,
                                /* notModified= */ true,
                                SystemClock.elapsedRealtime() - requestStart,
                                responseHeaders);
                    }
                    // Combine cached and response headers so the response will be complete.
                    List<Header> combinedHeaders = combineHeaders(responseHeaders, entry);
                    return new NetworkResponse(
                            HttpURLConnection.HTTP_NOT_MODIFIED,
                            entry.data,
                            /* notModified= */ true,
                            SystemClock.elapsedRealtime() - requestStart,
                            combinedHeaders);
                }

                // Some responses such as 204s do not have content.  We must check.
                InputStream inputStream = httpResponse.getContent();
                if (inputStream != null) {
                    responseContents =
                            inputStreamToBytes(inputStream, httpResponse.getContentLength());
                } else {
                    // Add 0 byte response as a way of honestly representing a
                    // no-content request.
                    responseContents = new byte[0];
                }

                // if the request is slow, log it.
                long requestLifetime = SystemClock.elapsedRealtime() - requestStart;
                logSlowRequests(requestLifetime, request, responseContents, statusCode);

                if (statusCode < 200 || statusCode > 299) {
                    throw new IOException();
                }
                return new NetworkResponse(
                        statusCode,
                        responseContents,
                        /* notModified= */ false,
                        SystemClock.elapsedRealtime() - requestStart,
                        responseHeaders);
            } catch (SocketTimeoutException e) {
                attemptRetryOnException("socket", request, new TimeoutError());
            } catch (MalformedURLException e) {
                throw new RuntimeException("Bad URL " + request.getUrl(), e);
            } catch (IOException e) {
                int statusCode;
                if (httpResponse != null) {
                    statusCode = httpResponse.getStatusCode();
                } else {
                    throw new NoConnectionError(e);
                }
                VolleyLog.e("Unexpected response code %d for %s", statusCode, request.getUrl());
                NetworkResponse networkResponse;
                if (responseContents != null) {
                    networkResponse =
                            new NetworkResponse(
                                    statusCode,
                                    responseContents,
                                    /* notModified= */ false,
                                    SystemClock.elapsedRealtime() - requestStart,
                                    responseHeaders);
                    if (statusCode == HttpURLConnection.HTTP_UNAUTHORIZED
                            || statusCode == HttpURLConnection.HTTP_FORBIDDEN) {
                        attemptRetryOnException(
                                "auth", request, new AuthFailureError(networkResponse));
                    } else if (statusCode >= 400 && statusCode <= 499) {
                        // Don't retry other client errors.
                        throw new ClientError(networkResponse);
                    } else if (statusCode >= 500 && statusCode <= 599) {
                        if (request.shouldRetryServerErrors()) {
                            attemptRetryOnException(
                                    "server", request, new ServerError(networkResponse));
                        } else {
                            throw new ServerError(networkResponse);
                        }
                    } else {
                        // 3xx? No reason to retry.
                        throw new ServerError(networkResponse);
                    }
                } else {
                    attemptRetryOnException("network", request, new NetworkError());
                }
            }
        }
    }
```
通过mBaseHttpStack的executeRequest方法获取response，
其实mBaseHttpStack就是RequestQueue在初始化时构造的HulStack和HttpClientStack的父类
回到NetworkDispatcher请求网络后，会将响应结果存在缓存中，并调用mDelivery(即ExecutorDelivery)的postResponse方法。

```java
    @Override
    public void postResponse(Request<?> request, Response<?> response, Runnable runnable) {
        request.markDelivered();
        request.addMarker("post-response");
        mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, runnable));
    }

            @Override
            public void run() {
                // NOTE: If cancel() is called off the thread that we're currently running in (by
                // default, the main thread), we cannot guarantee that deliverResponse()/deliverError()
                // won't be called, since it may be canceled after we check isCanceled() but before we
                // deliver the response. Apps concerned about this guarantee must either call cancel()
                // from the same thread or implement their own guarantee about not invoking their
                // listener after cancel() has been called.
    
                // If this request has canceled, finish it and don't deliver.
                if (mRequest.isCanceled()) {
                    mRequest.finish("canceled-at-delivery");
                    return;
                }
    
                // Deliver a normal response or error, depending.
                if (mResponse.isSuccess()) {
                    mRequest.deliverResponse(mResponse.result);
                } else {
                    mRequest.deliverError(mResponse.error);
                }
    
                // If this is an intermediate response, add a marker, otherwise we're done
                // and the request can be finished.
                if (mResponse.intermediate) {
                    mRequest.addMarker("intermediate-response");
                } else {
                    mRequest.finish("done");
                }
    
                // If we have been provided a post-delivery runnable, run it.
                if (mRunnable != null) {
                    mRunnable.run();
                }
            }
```
如果响应成功将会执行Request的deliverResponse方法，并把响应结果传进去，最后传递到Request的onResponse方法；    
如果响应失败，就执行deliverError方法，并把响应失败的对象传进去，最后传递到Request的onErrorResponse方法。

Tips：
1. Volley是如何把请求的数据回调回主线程中的？  
使用Handler.postRunnable（Runnable）方法回调回主线程中进行处理
```java
    public ExecutorDelivery(final Handler handler) {
        // Make an Executor that just wraps the handler.
        mResponsePoster =
                new Executor() {
                    @Override
                    public void execute(Runnable command) {
                        handler.post(command);
                    }
                };
    }

    public RequestQueue(Cache cache, Network network, int threadPoolSize) {
        this(
                cache,
                network,
                threadPoolSize,
                new ExecutorDelivery(new Handler(Looper.getMainLooper())));
    }
```

优点  

自动调度网络请求  
多并发网络请求  
可以缓存http请求  
支持请求的优先级  
支持取消网络请求  
REST API  

缺点  

使用httpclient，httpurlconnection
不适合大文件I/O操作，因为Volley会把所有服务器返回的数据在解析期间缓存到内存
只支持http请求
图片加载性能一般


参考资料：

https://my.oschina.net/tingzi/blog/3019686