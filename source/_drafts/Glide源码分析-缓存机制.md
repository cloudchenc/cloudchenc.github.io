---
title: Glide源码分析-缓存机制
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

#### 1. 读取内存缓存

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

#### 2. 写入内存缓存

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


参考资料：  

https://blog.csdn.net/guolin_blog/article/details/54895665