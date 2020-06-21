---
title: Handler机制原理
tags: 
    - 消息机制
    - Handler  
categories: Android
---

说到Android消息机制，就不得不谈到Handler机制，那么本文以android-29的源码分析Handler机制的原理。

首先Handler是在 android.os 包下，与它在同一个包下的 Looper，Message，MessageQueue等就是本文的重点。

```$xslt
    //handler example
    class LooperThread extends Thread {
         public Handler mHandler;
         public void run() {
            Looper.prepare();
            mHandler = new Handler() {
                public void handleMessage(Message msg) {
                    // process incoming messages here
                }
            };
            Looper.loop();
         }
    }
```

1.Looper.prepare()调用后，创建了MessageQueue和Looper，并把looper的实例放入ThreadLocal中保存

2.创建handler

Handler提供了两种使用方式，
第一种方式是直接实例化，调用handler的构造函数。

```$xslt
    //Handler.java#构造函数一
    public Handler(@Nullable Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            // 注意这里在判断handler的类型，满足条件时会提示可能发生内存泄漏
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }
        // 注意这里如果是子线程要提前调用Looper.prepare()方法，不然就会走入下面的报错
        mLooper = Looper.myLooper(); 
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

```$xslt
    //Handler.java#构造函数二
    public Handler(@NonNull Looper looper, @Nullable Callback callback, boolean async) {
        mLooper = looper;
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }
```

构造函数里有三个参数，分别是

looper：创建了MessageQueue，在某个线程中遍历对应的消息队列，相当于发动机的角色。
callback：定义了handleMessage()方法，对接收到的消息进行处理，相当于快递分拣员的角色。
async：设置消息是否异步，默认是同步的，如果是异步将不受同步障碍的约束。


第二种方式是继承handler，重写handleMessage()方法。

创建handler后，handler就持有了looper和messageQueue的引用，
通过调用enqueueMessage()方法往messageQueue中发送消息，其实最终调用的是MessageQueue的enqueueMessage()方法
```
    boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }

/frameworks/base/core/jni/android_os_MessageQueue.cpp#
static void android_os_MessageQueue_nativeWake(JNIEnv* env, jclass clazz, jlong ptr) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->wake();
}

/system/core/libutils/Looper.cpp#
void Looper::wake() {

    uint64_t inc = 1;
    ssize_t nWrite = TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
    if (nWrite != sizeof(uint64_t)) {
        if (errno != EAGAIN) {
            LOG_ALWAYS_FATAL("Could not write wake signal to fd %d: %s",
                    mWakeEventFd, strerror(errno));
        }
    }
}
```
TEMP_FAILURE_RETRY(write(mWakeEventFd, &inc, sizeof(uint64_t)));
这行代码会不断地调用write()方法，直到管道中的数据全部读完


3.Looper.loop()
```$xslt
        //Looper.java#裁剪代码
        public static void loop() {
            final Looper me = myLooper(); // 获取当前线程的looper实例
            if (me == null) {
                throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
            }
            final MessageQueue queue = me.mQueue; // 获取当前线程的messageQueue实例
    
            // 轮询消息队列
            for (; ; ) {
                Message msg = queue.next(); // 划重点！！！
                if (msg == null) {
                    return;
                }
    
                try {
                    msg.target.dispatchMessage(msg); // 调用目标handler处理消息
                } catch (Exception exception) {
                    throw exception;
                }
    
                msg.recycleUnchecked(); // 回收消息，放入对象池，通过Message.obtain()复用
            }
        }

    Handler.java#
    public void dispatchMessage(@NonNull Message msg) {
        if (msg.callback != null) {
            //当消息回调不为空时，执行回调的run方法
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                //当handler设置了mCallback时，执行重写的handleMessage方法
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            //执行默认的handleMessage方法
            handleMessage(msg);
        }
    }
```

这里重点分析 Message msg = queue.next();
```$xslt
    Message next() {
        final long ptr = mPtr; // NativeMessageQueue对象id
        if (ptr == 0) {
            return null;
        }

        int pendingIdleHandlerCount = -1; // 第一次循环默认为-1
        int nextPollTimeoutMillis = 0;
        for (;;) {
            if (nextPollTimeoutMillis != 0) {
                Binder.flushPendingCommands();
            }
            //阻塞方法，直到消息队列被唤醒或等待超时
            nativePollOnce(ptr, nextPollTimeoutMillis);

            synchronized (this) {
                final long now = SystemClock.uptimeMillis();
                Message prevMsg = null;
                Message msg = mMessages;
                // 当消息的目标handler为空时，尝试获取下一个异步消息
                if (msg != null && msg.target == null) {
                    // 受同步障碍影响，在消息队列中查找异步消息
                    do {
                        prevMsg = msg;
                        msg = msg.next;
                    } while (msg != null && !msg.isAsynchronous());
                }
                if (msg != null) {
                    if (now < msg.when) {
                        // 延时消息时间尚未触发，则设置下一轮超时唤醒时间
                        nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                    } else {
                        mBlocked = false;
                        if (prevMsg != null) {
                            prevMsg.next = msg.next;
                        } else {
                            mMessages = msg.next;
                        }
                        msg.next = null;
                        msg.markInUse();
                        return msg;
                    }
                } else {
                    // 没有更多消息
                    nextPollTimeoutMillis = -1;
                }

                // 所有待处理的消息均已处理，现在执行退出消息
                if (mQuitting) {
                    dispose();
                    return null;
                }

                // 如果是第一次空闲，则获取当前idle handler的数量，对应上面初始化时设置为-1。
                // idle handler仅在消息队列为空或当前消息时间尚未触发时运行。
                if (pendingIdleHandlerCount < 0
                        && (mMessages == null || now < mMessages.when)) {
                    pendingIdleHandlerCount = mIdleHandlers.size();
                }
                if (pendingIdleHandlerCount <= 0) {
                    // 没有idle handler运行, 进入等待状态
                    mBlocked = true;
                    continue;
                }

                if (mPendingIdleHandlers == null) {
                    mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
                }
                mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
            }

            // 调用所有idle handler
            for (int i = 0; i < pendingIdleHandlerCount; i++) {
                final IdleHandler idler = mPendingIdleHandlers[i];
                mPendingIdleHandlers[i] = null; // release the reference to the handler

                boolean keep = false;
                try {
                    keep = idler.queueIdle();
                } catch (Throwable t) {
                    Log.wtf(TAG, "IdleHandler threw exception", t);
                }

                if (!keep) {
                    synchronized (this) {
                        mIdleHandlers.remove(idler);
                    }
                }
            }

            // 重置idle handler数量，防止重复调用
            pendingIdleHandlerCount = 0;
            // 重置下次轮询的超时等待时间
            nextPollTimeoutMillis = 0;
        }
    }
```
mPtr的赋值是nativeInit(),网上暂时没找到Android Q的在线查看，暂时以Android P来分析native层的逻辑。
native层创建NativeMessageQueue对象后将对象id返回给java层，其中NativeMessageQueue创建时创建了Looper对象，
与java层Looper对象在创建时创建了MessageQueue对象恰恰相反
```
static jlong android_os_MessageQueue_nativeInit(JNIEnv* env, jclass clazz) {
    NativeMessageQueue* nativeMessageQueue = new NativeMessageQueue();
    if (!nativeMessageQueue) {
        jniThrowRuntimeException(env, "Unable to allocate native queue");
        return 0;
    }

    nativeMessageQueue->incStrong(env); // 强引用指针计数
    return reinterpret_cast<jlong>(nativeMessageQueue);
}
```

```$xslt
NativeMessageQueue::NativeMessageQueue() :
        mPollEnv(NULL), mPollObj(NULL), mExceptionObj(NULL) {
    mLooper = Looper::getForThread();
    // 
    if (mLooper == NULL) {
        mLooper = new Looper(false);
        Looper::setForThread(mLooper);
    }
}
```

nativePollOnce()调用到native层，
```$xslt
static void android_os_MessageQueue_nativePollOnce(JNIEnv* env, jobject obj,
        jlong ptr, jint timeoutMillis) {
    NativeMessageQueue* nativeMessageQueue = reinterpret_cast<NativeMessageQueue*>(ptr);
    nativeMessageQueue->pollOnce(env, obj, timeoutMillis);
}

frameworks/base/core/jni/android_os_MessageQueue.cpp#
void NativeMessageQueue::pollOnce(JNIEnv* env, jobject pollObj, int timeoutMillis) {
    mPollEnv = env;
    mPollObj = pollObj;
    mLooper->pollOnce(timeoutMillis); 调用到了looper的pollOnce()方法
    mPollObj = NULL;
    mPollEnv = NULL;

    if (mExceptionObj) {
        env->Throw(mExceptionObj);
        env->DeleteLocalRef(mExceptionObj);
        mExceptionObj = NULL;
    }
}

system/core/libutils/Looper.cpp#
int Looper::pollOnce(int timeoutMillis, int* outFd, int* outEvents, void** outData) {
    int result = 0;
    for (;;) {
        while (mResponseIndex < mResponses.size()) {
            const Response& response = mResponses.itemAt(mResponseIndex++);
            int ident = response.request.ident;
            if (ident >= 0) {
                int fd = response.request.fd;
                int events = response.events;
                void* data = response.request.data;

                if (outFd != NULL) *outFd = fd;
                if (outEvents != NULL) *outEvents = events;
                if (outData != NULL) *outData = data;
                return ident;
            }
        }

        if (result != 0) {
            if (outFd != NULL) *outFd = 0;
            if (outEvents != NULL) *outEvents = 0;
            if (outData != NULL) *outData = NULL;
            return result;
        }

        result = pollInner(timeoutMillis);
    }
}

第一次会跳过对response的处理，直接调用pollInner()方法
int Looper::pollInner(int timeoutMillis) {
    if (timeoutMillis != 0 && mNextMessageUptime != LLONG_MAX) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        int messageTimeoutMillis = toMillisecondTimeoutDelay(now, mNextMessageUptime);
        if (messageTimeoutMillis >= 0
                && (timeoutMillis < 0 || messageTimeoutMillis < timeoutMillis)) {
            timeoutMillis = messageTimeoutMillis;
        }
    }

    int result = POLL_WAKE;
    mResponses.clear();
    mResponseIndex = 0;

    mPolling = true; // 即将进入idle状态

    struct epoll_event eventItems[EPOLL_MAX_EVENTS];
    //阻塞当前方法，用于等待事件发生或者超时
    int eventCount = epoll_wait(mEpollFd, eventItems, EPOLL_MAX_EVENTS, timeoutMillis);

    mPolling = false; // 即将退出idle状态

    mLock.lock();

    // 如果需要的话重建epoll，跳转Done
    if (mEpollRebuildRequired) {
        mEpollRebuildRequired = false;
        rebuildEpollLocked();
        goto Done;
    }

    // 检查事件数量是否小于0，返回异常，跳转Done
    if (eventCount < 0) {
        if (errno == EINTR) {
            goto Done;
        }
        result = POLL_ERROR;
        goto Done;
    }

    // 检查事件数量是否等于0，返回超时，跳转Done
    if (eventCount == 0) {
        result = POLL_TIMEOUT;
        goto Done;
    }

    // 处理所有事件消息
    for (int i = 0; i < eventCount; i++) {
        int fd = eventItems[i].data.fd;
        uint32_t epollEvents = eventItems[i].events;
        if (fd == mWakeEventFd) {
            if (epollEvents & EPOLLIN) {
                awoken();
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on wake event fd.", epollEvents);
            }
        } else {
            ssize_t requestIndex = mRequests.indexOfKey(fd);
            if (requestIndex >= 0) {
                int events = 0;
                if (epollEvents & EPOLLIN) events |= EVENT_INPUT;
                if (epollEvents & EPOLLOUT) events |= EVENT_OUTPUT;
                if (epollEvents & EPOLLERR) events |= EVENT_ERROR;
                if (epollEvents & EPOLLHUP) events |= EVENT_HANGUP;
                pushResponse(events, mRequests.valueAt(requestIndex));
            } else {
                ALOGW("Ignoring unexpected epoll events 0x%x on fd %d that is "
                        "no longer registered.", epollEvents, fd);
            }
        }
    }
Done: ;

    // 调用待处理的消息回调
    mNextMessageUptime = LLONG_MAX;
    while (mMessageEnvelopes.size() != 0) {
        nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
        const MessageEnvelope& messageEnvelope = mMessageEnvelopes.itemAt(0);
        if (messageEnvelope.uptime <= now) {
            { // obtain handler
                sp<MessageHandler> handler = messageEnvelope.handler;
                Message message = messageEnvelope.message;
                mMessageEnvelopes.removeAt(0);
                mSendingMessage = true;
                mLock.unlock();

                handler->handleMessage(message);
            } // release handler

            mLock.lock();
            mSendingMessage = false;
            result = POLL_CALLBACK;
        } else {
            // The last message left at the head of the queue determines the next wakeup time.
            mNextMessageUptime = messageEnvelope.uptime;
            break;
        }
    }

    mLock.unlock(); // 释放锁

    // 调用所有响应回调
    for (size_t i = 0; i < mResponses.size(); i++) {
        Response& response = mResponses.editItemAt(i);
        if (response.request.ident == POLL_CALLBACK) {
            int fd = response.request.fd;
            int events = response.events;
            void* data = response.request.data;

            int callbackResult = response.request.callback->handleEvent(fd, events, data);
            if (callbackResult == 0) {
                removeFd(fd, response.request.seq);
            }

            response.request.callback.clear();
            result = POLL_CALLBACK;
        }
    }
    return result;
}
```

总结

    Looper: 每个线程只有一个Looper，负责管理MessageQueue,会不断地从MessageQueue中取出消息，并将消息分给对应的Handler处理。

    MessageQueue:由Looper负责管理，采用先进先出的方式管理Message(消息队列)。

    Handler：把消息发送给Looper管理的MessageQueue并负责处理Looper分给它的消息。

    消息只能在某个具体的Looper上消耗，因此每个Handler都会绑定一个Looper。但是多个Handler可以绑定同一个Looper（这也是在主线程中能够创建新的Handler的原因）。

![handler](https://tvax4.sinaimg.cn/mw690/d7f9b0f4gy1gfz57es2lxj20iv0ec0vh.jpg)


Tips:

1.为什么在主线程中使用handler不用调用Looper.prepareMainLooper()和Looper.loop()？
这是因为 ActivityThread 在初始化时已经调用过了，具体见如下源码。

```$xslt
    //ActivityThread.java#
    public static void main(String[] args) {
        ~~~~~~
        Looper.prepareMainLooper(); // 创建MessageQueue, Looper
        ......
        Looper.loop(); // 在线程中遍历消息队列
        ~~~~~~
    }
```

2.一个线程可以有几个Looper？为什么？
这是因为Looper.prepare()在每个线程只能调用一次，在创建Looper时，是以线程作为key值保存的，
所以第二次创建时，会从sThreadLocal中获取用当获取looper时会判断当前线程是否已经创建了looper，如果没有才会创建新的实例
```$xslt
static final ThreadLocal<Looper> sThreadLocal = new ThreadLocal<Looper>();
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```

3.主线程中的Looper.loop()一直无限循环为什么不会造成ANR?
首先要明白ANR的定义，activity超时5s，service超时10s，broadcast超时20s，才会导致Application Not Response。

4.HandlerThread 和 AsyncTask

参考资料：
http://gityuan.com/2015/12/26/handler-message-framework/
http://gityuan.com/2015/12/27/handler-message-native/
