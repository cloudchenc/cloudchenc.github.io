---
title: ThreadLocal原理分析
tags:
categories:
---

```$xslt
/**
 * This class provides thread-local variables.  These variables differ from
 * their normal counterparts in that each thread that accesses one (via its
 * {@code get} or {@code set} method) has its own, independently initialized
 * copy of the variable.  {@code ThreadLocal} instances are typically private
 * static fields in classes that wish to associate state with a thread (e.g.,
 * a user ID or Transaction ID).
 */
```
ThreadLocal类负责维护线程局部变量，这些变量是当前线程私有的，独立的，别的线程无法访问。

```
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```

```$xslt
    static class ThreadLocalMap {
        static class Entry extends WeakReference<ThreadLocal<?>> {
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
    }
```
ThreadLocalMap中存储的Entry拓展了弱引用，以当前ThreadLocal对象作为key值保存局部变量

https://juejin.im/post/5ac2eb52518825555e5e06ee#heading-6