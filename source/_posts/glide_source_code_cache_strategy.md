---
title: Glide源码分析-缓存机制
date: 2020-07-11 17:19:30
tags:
    - Glide
    - 网络框架
    - 源码分析
categories: Android
---

Glide缓存分为两个部分，一部分是内存缓存，另一部分是硬盘缓存。
  
内存缓存的作用是防止应用重复将图片数据读取到内存当中，

硬盘缓存的作用是防止应用重复从网络或其他地方重复下载和读取数据。

```java
public class Engine implements EngineJobListener,
        MemoryCache.ResourceRemovedListener,
        EngineResource.ResourceListener {

    public <T, Z, R> LoadStatus load(Key signature, int width, int height, DataFetcher<T> fetcher,
            DataLoadProvider<T, Z> loadProvider, Transformation<Z> transformation, ResourceTranscoder<Z, R> transcoder,
            Priority priority, boolean isMemoryCacheable, DiskCacheStrategy diskCacheStrategy, ResourceCallback cb) {
        Util.assertMainThread();
        long startTime = LogTime.getLogTime();

        final String id = fetcher.getId();
        EngineKey key = keyFactory.buildKey(id, signature, width, height, loadProvider.getCacheDecoder(),
                loadProvider.getSourceDecoder(), transformation, loadProvider.getEncoder(),
                transcoder, loadProvider.getSourceEncoder());

        EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
        if (cached != null) {
            cb.onResourceReady(cached);
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                logWithTimeAndKey("Loaded resource from cache", startTime, key);
            }
            return null;
        }

        EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
        if (active != null) {
            cb.onResourceReady(active);
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                logWithTimeAndKey("Loaded resource from active resources", startTime, key);
            }
            return null;
        }

        EngineJob current = jobs.get(key);
        if (current != null) {
            current.addCallback(cb);
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                logWithTimeAndKey("Added to existing load", startTime, key);
            }
            return new LoadStatus(cb, current);
        }

        EngineJob engineJob = engineJobFactory.build(key, isMemoryCacheable);
        DecodeJob<T, Z, R> decodeJob = new DecodeJob<T, Z, R>(key, width, height, fetcher, loadProvider, transformation,
                transcoder, diskCacheProvider, diskCacheStrategy, priority);
        EngineRunnable runnable = new EngineRunnable(engineJob, decodeJob, priority);
        jobs.put(key, engineJob);
        engineJob.addCallback(cb);
        engineJob.start(runnable);

        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Started new load", startTime, key);
        }
        return new LoadStatus(cb, engineJob);
    }
}
```

### 缓存key

通过调用 EngineKeyFactory 的 buildKey()方法生成 EngineKey 对象，作为缓存的key键。

### 内存缓存

#### 读取内存缓存

`EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);`
获取到缓存图片后，就直接调用`cb.onResourceReady(cached);`进行加载。
```java
    // 注意这里的isMemoryCacheable默认值为true, 通过skipMemoryCache()方法可以关闭内存缓存
    private EngineResource<?> loadFromCache(Key key, boolean isMemoryCacheable) {
        if (!isMemoryCacheable) {
            return null;
        }

        EngineResource<?> cached = getEngineResourceFromCache(key);
        if (cached != null) {
            cached.acquire();
            // 使用activeResources缓存当前正在使用中的图片
            activeResources.put(key, new ResourceWeakReference(key, cached, getReferenceQueue()));
        }
        return cached;
    }

    private EngineResource<?> getEngineResourceFromCache(Key key) {
        // 从当前LruResourceCache中移除，下面会放入activeResources中
        Resource<?> cached = cache.remove(key);

        final EngineResource result;
        if (cached == null) {
            result = null;
        } else if (cached instanceof EngineResource) {
            // Save an object allocation if we've cached an EngineResource (the typical case).
            result = (EngineResource) cached;
        } else {
            result = new EngineResource(cached, true /*isCacheable*/);
        }
        return result;
    }
```
这里的cache对象是 MemoryCache 接口的实现，赋值是在 GlideBuilder 对象的createGlide()方法中，创建的 LruResourceCache 对象  
 `memoryCache = new LruResourceCache(calculator.getMemoryCacheSize());`  
本质上是LruCache对象，内部维护了一个LinkedHashMap来保存强引用(区别于下面的弱引用)下的缓存图片。
```java
public class LruResourceCache extends LruCache<Key, Resource<?>> implements MemoryCache {
    private ResourceRemovedListener listener;

    /**
     * Constructor for LruResourceCache.
     *
     * @param size The maximum size in bytes the in memory cache can use.
     */
    public LruResourceCache(int size) {
        super(size);
    }

    @Override
    public void setResourceRemovedListener(ResourceRemovedListener listener) {
        this.listener = listener;
    }

    @Override
    protected void onItemEvicted(Key key, Resource<?> item) {
        if (listener != null) {
            listener.onResourceRemoved(item);
        }
    }

    @Override
    protected int getSize(Resource<?> item) {
        return item.getSize();
    }

    @SuppressLint("InlinedApi")
    @Override
    public void trimMemory(int level) {
        if (level >= android.content.ComponentCallbacks2.TRIM_MEMORY_MODERATE) {
            // Nearing middle of list of cached background apps
            // Evict our entire bitmap cache
            clearMemory();
        } else if (level >= android.content.ComponentCallbacks2.TRIM_MEMORY_BACKGROUND) {
            // Entering list of cached background apps
            // Evict oldest half of our bitmap cache
            trimToSize(getCurrentSize() / 2);
        }
    }
}
```


`EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);`
获取到缓存图片后，就直接调用`cb.onResourceReady(cached);`进行加载。
```java
    private EngineResource<?> loadFromActiveResources(Key key, boolean isMemoryCacheable) {
        if (!isMemoryCacheable) {
            return null;
        }

        EngineResource<?> active = null;
        WeakReference<EngineResource<?>> activeRef = activeResources.get(key);
        if (activeRef != null) {
            active = activeRef.get();
            if (active != null) {
                active.acquire();
            } else {
                // 当前key键对应的图片资源无效
                activeResources.remove(key);
            }
        }

        return active;
    }
```
这里的activeResources对象是本地维护的一个HashMap来保存弱引用下的缓存图片。  

在执行图片加载请求前，会先调用 loadFromCache()和loadFromActiveResources()两个方法来获取内存缓存，loadFromCache()使用的是LruCache算法，保存了强引用下的缓存图片；
而loadFromActiveResources()保存了弱引用下的缓存图片。只有在获取不到缓存的情况下，才会向下执行，开启线程来加载图片。

#### 写入内存缓存

上面理清了从内存缓存读取的流程，接下来分析一下是如何把图片写入内存缓存的。

这就要追溯到 EngineJob 中，当图片加载完成后，EngineJob 会通过 Handler 发送一条消息 MSG_COMPLETE 将执行逻辑切换到主线程中，执行 handleResultOnMainThread()方法
```java
class EngineJob implements EngineRunnable.EngineRunnableManager {

    private EngineResource<?> engineResource;

    private void handleResultOnMainThread() {
        if (isCancelled) {
            resource.recycle();
            return;
        } else if (cbs.isEmpty()) {
            throw new IllegalStateException("Received a resource without any callbacks to notify");
        }
        engineResource = engineResourceFactory.build(resource, isCacheable);
        hasResource = true;

        // Hold on to resource for duration of request so we don't recycle it in the middle of notifying if it
        // synchronously released by one of the callbacks.
        engineResource.acquire();
        listener.onEngineJobComplete(key, engineResource);

        for (ResourceCallback cb : cbs) {
            if (!isInIgnoredCallbacks(cb)) {
                engineResource.acquire();
                cb.onResourceReady(engineResource);
            }
        }
        // Our request is complete, so we can release the resource.
        engineResource.release();
    }

    static class EngineResourceFactory {
        public <R> EngineResource<R> build(Resource<R> resource, boolean isMemoryCacheable) {
            return new EngineResource<R>(resource, isMemoryCacheable);
        }
    }
}
```
通过EngineResourceFactory构建出了一个包含图片资源的EngineResource对象，然后把这个对象回调到Engine的onEngineJobComplete()方法中
```java
public class Engine implements EngineJobListener,
        MemoryCache.ResourceRemovedListener,
        EngineResource.ResourceListener {
    @Override
    public void onEngineJobComplete(Key key, EngineResource<?> resource) {
        Util.assertMainThread();
        // A null resource indicates that the load failed, usually due to an exception.
        if (resource != null) {
            resource.setResourceListener(key, this);

            if (resource.isCacheable()) {
                activeResources.put(key, new ResourceWeakReference(key, resource, getReferenceQueue()));
            }
        }
        // TODO: should this check that the engine job is still current?
        jobs.remove(key);
    }
}
```
回调过来的 EngineResource 对象会判断是否开启了内存缓存，然后放入到activeResources中，这就是弱引用缓存的写入了。

接着回到上面的 handleResultOnMainThread()方法中，可以看到 EngineResource 对象先调用了acquire()方法，接着调用了release()方法。
根据下面 EngineResource 的源码，可以发现内部使用了一个 acquired 变量用来记录图片被引用的次数，调用acquire()方法会让 acquired 加1，调用release()方法会让 acquired 变量减1。
```java
class EngineResource<Z> implements Resource<Z> {

    private int acquired;
    
    ...

    void acquire() {
        if (isRecycled) {
            throw new IllegalStateException("Cannot acquire a recycled resource");
        }
        if (!Looper.getMainLooper().equals(Looper.myLooper())) {
            throw new IllegalThreadStateException("Must call acquire on the main thread");
        }
        ++acquired;
    }

    void release() {
        if (acquired <= 0) {
            throw new IllegalStateException("Cannot release a recycled or not yet acquired resource");
        }
        if (!Looper.getMainLooper().equals(Looper.myLooper())) {
            throw new IllegalThreadStateException("Must call release on the main thread");
        }
        if (--acquired == 0) {
            listener.onResourceReleased(key, this);
        }
    }
}
```
也就是说，当 acquired 变量大于0时，说明图片正在使用，也就是放在activeResources弱引用缓存中。
而调用release()方法，当acquired变量为0时，说明图片不再使用，会回调 Engine 的 onResourceReleased()方法释放资源。
```java
public class Engine implements EngineJobListener,
        MemoryCache.ResourceRemovedListener,
        EngineResource.ResourceListener {
    @Override
    public void onResourceReleased(Key cacheKey, EngineResource resource) {
        Util.assertMainThread();
        activeResources.remove(cacheKey);
        if (resource.isCacheable()) {
            cache.put(cacheKey, resource);
        } else {
            resourceRecycler.recycle(resource);
        }
    }
}
```
可以看到，首先会从activeResources中移除缓存图片，判断当前资源是否开启缓存，如果开启的话，就放入LruResourceCache中，如果没有开启，就释放资源。
这样就实现了当图片正在使用时，就使用弱引用来进行缓存，当图片不再使用时，就使用LruCache来进行缓存。

以上就是Glide内存缓存的实现原理。


### 硬盘缓存

```java
Glide.with(this)
     .load(url)
     .diskCacheStrategy(DiskCacheStrategy.NONE)
     .into(imageView);
```
diskCacheStrategy()方法基本上就是Glide硬盘缓存的开关，接收四种参数：  
+ DiskCacheStrategy.ALL：表示既缓存原始图片，也缓存转换后的图片  
+ DiskCacheStrategy.NONE：表示不缓存任何内容  
+ DiskCacheStrategy.SOURCE：表示只缓存原始图片
+ DiskCacheStrategy.RESULT：表示只缓存转换后的图片（默认值）

当使用Glide加载一张图片的时候，默认并不会将原始图片展示出来，而是会对图片进行压缩和转换，Glide硬盘缓存默认保存的就是转换后的图片

#### 读取硬盘缓存
Glide开启线程来加载图片后会执行EngineRunnable的run()方法，run()方法中又会调用一个decode()方法
```java
class EngineRunnable implements Runnable, Prioritized {
    private Resource<?> decode() throws Exception {
        if (isDecodingFromCache()) {
            return decodeFromCache();
        } else {
            return decodeFromSource();
        }
    }
}
```
默认情况下Glide会调用decodeFromCache()方法从硬盘缓存中读取图片，
当缓存中不存在要读取的图片时，才会调用onLoadFailed()方法，将`stage`赋值为`Stage.SOURCE`，
重新执行EngineRunnable的run()方法，最终调用到decodeFromSource()方法去读取原始图片。

那么分析下decodeFromCache()方法：
```java
    private Resource<?> decodeFromCache() throws Exception {
        Resource<?> result = null;
        try {
            result = decodeJob.decodeResultFromCache();
        } catch (Exception e) {
            if (Log.isLoggable(TAG, Log.DEBUG)) {
                Log.d(TAG, "Exception decoding result from cache: " + e);
            }
        }

        if (result == null) {
            result = decodeJob.decodeSourceFromCache();
        }
        return result;
    }
```
先调用了decodeResultFromCache()方法获取缓存，如果获取不到，会再调用decodeSourceFromCache()方法获取缓存，
对应DiskCacheStrategy.RESULT和DiskCacheStrategy.SOURCE这两个参数。
```java
    public Resource<Z> decodeResultFromCache() throws Exception {
        if (!diskCacheStrategy.cacheResult()) {
            return null;
        }

        long startTime = LogTime.getLogTime();
        Resource<T> transformed = loadFromCache(resultKey);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Decoded transformed from cache", startTime);
        }
        startTime = LogTime.getLogTime();
        Resource<Z> result = transcode(transformed);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Transcoded transformed from cache", startTime);
        }
        return result;
    }

    public Resource<Z> decodeSourceFromCache() throws Exception {
        if (!diskCacheStrategy.cacheSource()) {
            return null;
        }

        long startTime = LogTime.getLogTime();
        Resource<T> decoded = loadFromCache(resultKey.getOriginalKey());
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Decoded source from cache", startTime);
        }
        return transformEncodeAndTranscode(decoded);
    }
```
这两个方法都是调用了loadFromCache()方法从缓存中读取数据，
只是 decodeResultFromCache()传入的是resultKey对应转换后的图片缓存key，
而decodeSourceFromCache()传入的是resultKey.getOriginalKey()【也就是图片的url】对应原始图片的缓存key。

当decodeResultFromCache()获取到缓存后，直接将数据转码为另一个图片类型并返回
当decodeSourceFromCache()获取到缓存后，需要调用transformEncodeAndTranscode()方法先将数据转换一下宽高，再转码为另一个图片类型返回

而loadFromCache()方法其实就是DiskLruCache方法的调用
```java
    private Resource<T> loadFromCache(Key key) throws IOException {
        File cacheFile = diskCacheProvider.getDiskCache().get(key);
        if (cacheFile == null) {
            return null;
        }

        Resource<T> result = null;
        try {
            result = loadProvider.getCacheDecoder().decode(cacheFile, width, height);
        } finally {
            if (result == null) {
                diskCacheProvider.getDiskCache().delete(key);
            }
        }
        return result;
    }
```
调用getDiskCache()方法获取到DiskLruCache的实例，根据key得到硬盘缓存的图片。
如果是空，或解码失败，就返回null，同时删除key键对应的硬盘缓存资源，
不为空就解码为Resource对象返回。

#### 写入硬盘缓存
上文分析，在没有缓存的情况下，会调用 decodeFromSource()方法来读取原始图片。
```java
    public Resource<Z> decodeFromSource() throws Exception {
        Resource<T> decoded = decodeSource();
        return transformEncodeAndTranscode(decoded);
    }
```
decodeSource()方法是用来解码原始图片的，而transformEncodeAndTranscode()是对图片进行转换和转码的。
所以硬盘缓存中的原始图片缓存是在decodeSource()方法中写入的：
```java
    private Resource<T> decodeSource() throws Exception {
        Resource<T> decoded = null;
        try {
            long startTime = LogTime.getLogTime();
            final A data = fetcher.loadData(priority);
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                logWithTimeAndKey("Fetched data", startTime);
            }
            if (isCancelled) {
                return null;
            }
            decoded = decodeFromSourceData(data);
        } finally {
            fetcher.cleanup();
        }
        return decoded;
    }

    private Resource<T> decodeFromSourceData(A data) throws IOException {
        final Resource<T> decoded;
        if (diskCacheStrategy.cacheSource()) {
            decoded = cacheAndDecodeSourceData(data);
        } else {
            long startTime = LogTime.getLogTime();
            decoded = loadProvider.getSourceDecoder().decode(data, width, height);
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                logWithTimeAndKey("Decoded from source", startTime);
            }
        }
        return decoded;
    }

    private Resource<T> cacheAndDecodeSourceData(A data) throws IOException {
        long startTime = LogTime.getLogTime();
        SourceWriter<A> writer = new SourceWriter<A>(loadProvider.getSourceEncoder(), data);
        diskCacheProvider.getDiskCache().put(resultKey.getOriginalKey(), writer);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Wrote source to cache", startTime);
        }

        startTime = LogTime.getLogTime();
        Resource<T> result = loadFromCache(resultKey.getOriginalKey());
        if (Log.isLoggable(TAG, Log.VERBOSE) && result != null) {
            logWithTimeAndKey("Decoded source from cache", startTime);
        }
        return result;
    }
```
fetcher.loadData()读取原始数据 
--> decodeFromSourceData()解码图片 
--> 判断是否缓存原始图片 
--> cacheAndDecodeSourceData()将原始图片写入硬盘缓存【使用的key是resultKey.getOriginalKey()】

而硬盘缓存中的转换后的图片是在transformEncodeAndTranscode()方法中写入的：
```java
    private Resource<Z> transformEncodeAndTranscode(Resource<T> decoded) {
        long startTime = LogTime.getLogTime();
        Resource<T> transformed = transform(decoded);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Transformed resource from source", startTime);
        }

        writeTransformedToCache(transformed);

        startTime = LogTime.getLogTime();
        Resource<Z> result = transcode(transformed);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Transcoded transformed from source", startTime);
        }
        return result;
    }

    private void writeTransformedToCache(Resource<T> transformed) {
        if (transformed == null || !diskCacheStrategy.cacheResult()) {
            return;
        }
        long startTime = LogTime.getLogTime();
        SourceWriter<Resource<T>> writer = new SourceWriter<Resource<T>>(loadProvider.getEncoder(), transformed);
        diskCacheProvider.getDiskCache().put(resultKey, writer);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Wrote transformed from source to cache", startTime);
        }
    }
```
transform()方法对图片进行转换
--> writeTransformedToCache()将转换后的图片写入硬盘缓存【使用的key是resultKey】

至此，Glide缓存机制分析结束，送上一张流程图：



参考资料：  

https://blog.csdn.net/guolin_blog/article/details/54895665