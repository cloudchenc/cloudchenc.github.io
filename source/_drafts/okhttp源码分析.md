---
title: Okhttp源码分析
tags:
categories:
---

### 1. okhttp请求流程

1. 建造者模式创建 OkHttpClient 对象
2. 创建 Request 对象
3. 将 okhttpclient 和 request 作为参数创建 recall 对象，开始执行请求
4. 调用getResponseWithInterceptorChain()方法，使用责任链模式调用一系列拦截器请求处理和响应得到Response 
5. 拦截器分析：
        自定义普通拦截器
        RetryAndFollowUpInterceptor(重试请求)
        BridgeInterceptor(添加header)，
        CacheInterceptor(缓存处理)，
        ConnectInterceptor(和服务器建立连接)，
        自定义网络拦截器
        CallServerInterceptor(向服务器发起请求，读取响应)
        
        

### 2. 网络请求缓存处理

缓存拦截器会根据请求的信息和缓存的响应的信息来判断是否存在缓存可用，
如果有可以使用的缓存，那么就返回该缓存给用户，
否则就继续使用责任链模式来从服务器中获取响应。当获取到响应的时候，又会把响应缓存到磁盘上面。

### 3. 连接池复用

HttpCodec是对 HTTP 协议操作的抽象，有两个实现：Http1Codec和Http2Codec，
顾名思义，它们分别对应 HTTP/1.1 和 HTTP/2 版本的实现。

连接复用的好处：省去了进行 TCP 和 TLS 握手的一个过程

首先会对缓存中的连接进行遍历，以寻找一个闲置时间最长的连接，
然后根据该连接的闲置时长和最大允许的连接数量等参数来决定是否应该清理该连接。
同时注意上面的方法的返回值是一个时间，如果闲置时间最长的连接仍然需要一段时间才能被清理的时候，
会返回这段时间的时间差，然后会在这段时间之后再次对连接池进行清理。


