---
title: RxJava源码分析
tags:
    - RxJava
    - 源码分析
    - 响应式编程
    - 观察者模式
categories: Android
---

说到RxJava，就不得不提观察者模式。

Observable，Observer，订阅订阅，背压

RxJava是响应式编程的框架，使用了观察者模式来支持数据和事件的序列，同时添加了操作符操作符，来组合序列。

订阅流程

线程切换

#### 订阅流程

```java
Observable.create(new ObservableOnSubscribe<String>() {
    @Override
    public void subscribe(ObservableEmitter<String>emitter) throws Exception {
        emitter.onNext("1");
        emitter.onNext("2");
        emitter.onNext("3");
        emitter.onComplete();
    }
}).subscribe(new Observer<String>() {
    @Override
    public void onSubscribe(Disposable d) {
        Log.d(TAG, "onSubscribe");
    }
    @Override
    public void onNext(String s) {
        Log.d(TAG, "onNext : " + s);
    }
    @Override
    public void onError(Throwable e) {
        Log.d(TAG, "onError : " + e.toString());
    }
    @Override
    public void onComplete() {
        Log.d(TAG, "onComplete");
    }
});
```
首先创建了被观察者 Observable，然后创建了一个观察者 Observer 订阅了这个被观察者，所以订阅流程被分为了：
1. 创建被观察者过程
2. 订阅过程

##### 1.1 创建被观察者过程

###### 1.1 Observable#create()
```java
// 省略一些检测性的注解
public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
    ObjectHelper.requireNonNull(source, "source is null");
    return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
}
```
创建了一个 ObservableCreate 对象，同时把定义好的ObservableOnSubscribe对象作为参数传入 ObservableCreate 对象中，最后调用了RxJavaPlugins的onAssembly()方法

###### 1.2 ObservableCreate
```java
public final class ObservableCreate<T> extends Observable<T> {

    final ObservableOnSubscribe<T> source;

    public ObservableCreate(ObservableOnSubscribe<T> source) {
        this.source = source;
    }

    ...
}
```

###### 1.3 RxJavaPlugins#onAssembly()

###### 1.4 创建被观察者过程小结

仅仅是将自定义的ObservableOnSubscribe对象封装成了ObservableCreate对象

##### 1.2 订阅过程

###### 2.1 Observable#subscribe()

```java
public final void subscribe(Observer<? super T> observer) {
    ...

    // 1
    observer = RxJavaPlugins.onSubscribe(this,observer);

    ...

    // 2
    subscribeActual(observer);

    ...
}
```
在Observable的subscribe()方法内部首先调用了RxJavaPlugins的onSubscribe()方法。

###### 2.2 RxJavaPlugins#onSubscribe()

```java
public static <T> Observer<? super T> onSubscribe(@NonNull Observable<T> source, @NonNull Observer<? super T> observer) {

    // 应用hook函数的一些处理，一般用到不到
    ...

    return observer;
}
```
除去hook应用的逻辑，这里仅仅是将observer返回了。接着来分析下subscribeActual()方法，

###### 2.3 Observable#subscribeActual()
```java
protected abstract void subscribeActual(Observer<? super T> observer);
```
这是一个抽象的方法，很明显，它对应的具体实现类就是我们在第一步创建的ObservableCreate类，接下来看到ObservableCreate的subscribeActual()方法。

###### 2.4 ObservableCreate#subscribeActual()
```java
@Override
protected void subscribeActual(Observer<? super T> observer) {
    // 1
    CreateEmitter<T> parent = new CreateEmitter<T>(observer);
    // 2
    observer.onSubscribe(parent);

    try {
        // 3
        source.subscribe(parent);
    } catch (Throwable ex) {
        Exceptions.throwIfFatal(ex);
        parent.onError(ex);
    }
}
```
在注释1处，首先新创建了一个CreateEmitter对象，同时传入了我们自定义的observer对象进去。

####### 2.4.1 CreateEmitter#
```java
static final class CreateEmitter<T>
extends AtomicReference<Disposable>
implements ObservableEmitter<T>, Disposable {

    ...

    final Observer<? super T> observer;

    CreateEmitter(Observer<? super T> observer) {
        this.observer = observer;
    }

    ...
}
```
从上面可以看出，CreateEmitter通过继承了Java并发包中的原子引用类AtomicReference保证了事件流切断状态Dispose的一致性（这里不理解的话，看到后面讲解Dispose的时候就明白了），并实现了ObservableEmitter接口和Disposable接口，接着我们分析下注释2处的observer.onSubscribe(parent)，这个onSubscribe回调的含义其实就是告诉观察者已经成功订阅了被观察者。再看到注释3处的source.subscribe(parent)这行代码，这里的source其实是ObservableOnSubscribe对象，我们看到ObservableOnSubscribe的subscribe()方法。

####### 2.4.2 ObservableOnSubscribe#subscribe()
```java
Observable observable = Observable.create(new ObservableOnSubscribe<String>() {
    @Override
    public void subscribe(ObservableEmitter<String> emitter) throws Exception {
        emitter.onNext("1");
        emitter.onNext("2");
        emitter.onNext("3");
        emitter.onComplete();
    }
});
```
这里面使用到了ObservableEmitter的onNext()方法将事件流发送出去，最后调用了onComplete()方法完成了订阅过程。ObservableEmitter是一个抽象类，实现类就是我们传入的CreateEmitter对象，接下来我们看看CreateEmitter的onNext()方法和onComplete()方法的处理。

####### 2.4.3 CreateEmitter#onNext() && CreateEmitter#onComplete()
```java
static final class CreateEmitter<T>
extends AtomicReference<Disposable>
implements ObservableEmitter<T>, Disposable {

...

@Override
public void onNext(T t) {
    ...

    if (!isDisposed()) {
        //调用观察者的onNext()
        observer.onNext(t);
    }
}

@Override
public void onComplete() {
    if (!isDisposed()) {
        try {
            observer.onComplete();
        } finally {
            dispose();
        }
    }
}

...

}
```
在CreateEmitter的onNext和onComplete方法中首先都要经过一个isDisposed的判断，作用就是看当前的事件流是否被切断（废弃）掉了，默认是不切断的，如果想要切断，可以调用Disposable的dispose()方法将此状态设置为切断（废弃）状态。我们继续看看这个isDisposed内部的处理。

####### 2.4.4 ObservableEmitter#isDisposed()
```java
@Override
public boolean isDisposed() {
    return DisposableHelper.isDisposed(get());
}
```

####### 2.4.5 DisposableHelper#isDisposed() && DisposableHelper#set()
```java
public enum DisposableHelper implements Disposable {

    DISPOSED;

    public static boolean isDisposed(Disposable d) {
        // 1
        return d == DISPOSED;
    }

    public static boolean set(AtomicReference<Disposable> field, Disposable d) {
        for (;;) {
            Disposable current = field.get();
            if (current == DISPOSED) {
                if (d != null) {
                    d.dispose();
                }
                return false;
            }
            // 2
            if (field.compareAndSet(current, d)) {
                if (current != null) {
                    current.dispose();
                }
                return true;
            }
        }
    }

    ...

    public static boolean dispose(AtomicReference<Disposable> field) {
        Disposable current = field.get();
        Disposable d = DISPOSED;
        if (current != d) {
            // ...
            current = field.getAndSet(d);
            if (current != d) {
                if (current != null) {
                    current.dispose();
                }
                return true;
            }
        }
        return false;
    }

    ...
}
```
DisposableHelper是一个枚举类，内部只有一个值即DISPOSED, 从上面的分析可知它就是用来标记事件流被切断（废弃）状态的。先看到注释2和注释3处的代码field.compareAndSet(current, d)和field.getAndSet(d)，这里使用了原子引用AtomicReference内部包装的CAS方法处理了标志Disposable的并发读写问题，最后看到注释3处，将我们传入的CreateEmitter这个原子引用类保存的Dispable状态和DisposableHelper内部的DISPOSED进行比较，如果相等，就证明数据流被切断了。为了更进一步理解Disposed的作用，再来看看CreateEmitter中剩余的关键方法。

####### 2.4.6 CreateEmitter#
```java
@Override
public void onNext(T t) {
    ...
    // 1
    if (!isDisposed()) {
        observer.onNext(t);
    }
}

@Override
public void onError(Throwable t) {
    if (!tryOnError(t)) {
        // 2
        RxJavaPlugins.onError(t);
    }
}

@Override
public boolean tryOnError(Throwable t) {
    ...
    // 3
    if (!isDisposed()) {
        try {
            observer.onError(t);
        } finally {
            // 4
            dispose();
        }
        return true;
    }
    return false;
}

@Override
public void onComplete() {
    // 5
    if (!isDisposed()) {
        try {
            observer.onComplete();
        } finally {
            // 6
            dispose();
        }
    }
}
```
在注释1、3、5处，onNext()和onError()、onComplete()方法首先都会判断事件流是否被切断的处理，如果事件流此时被切断了，那么onNext()和onComplete()则会退出方法体，不做处理，onError()则会执行到RxJavaPlugins.onError(t)这句代码，内部会直接抛出异常，导致崩溃。如果事件流没有被切断，那么在onError()和onComplete()内部最终会调用到注释4、6处的这句dispose()代码，将事件流进行切断，由此可知，onError()和onComplete()只能调用一个，如果先执行的是onComplete()，再调用onError()的话就会导致异常崩溃。

#### 2. 线程切换
```java
Observable.create(new ObservableOnSubscribe<String>() {
    @Override
    public voidsubscribe(ObservableEmitter<String>emitter) throws Exception {
        emitter.onNext("1");
        emitter.onNext("2");
        emitter.onNext("3");
        emitter.onComplete();
    }
}) 
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe(new Observer<String>() {
        @Override
        public void onSubscribe(Disposable d) {
            Log.d(TAG, "onSubscribe");
        }
        @Override
        public void onNext(String s) {
            Log.d(TAG, "onNext : " + s);
        }
        @Override
        public void onError(Throwable e) {
            Log.d(TAG, "onError : " +e.toString());
        }
        @Override
        public void onComplete() {
            Log.d(TAG, "onComplete");
        }
});
```
可以看到，RxJava的线程切换主要分为subscribeOn()和observeOn()方法，首先，来分析下subscribeOn()方法。

##### 1. subscribeOn(Schedulers.io())
在Schedulers.io()方法中，我们需要先传入一个Scheduler调度类，这里是传入了一个调度到io子线程的调度类，我们看看这个Schedulers.io()方法内部是怎么构造这个调度器的。

##### 2. Schedulers#io()
```java
static final Scheduler IO;

...

public static Scheduler io() {
    // 1
    return RxJavaPlugins.onIoScheduler(IO);
}

static {
    ...

    // 2
    IO = RxJavaPlugins.initIoScheduler(new IOTask());
}

static final class IOTask implements Callable<Scheduler> {
    @Override
    public Scheduler call() throws Exception {
        // 3
        return IoHolder.DEFAULT;
    }
}

static final class IoHolder {
    // 4
    static final Scheduler DEFAULT = new IoScheduler();
}
```
Schedulers这个类的代码很多，这里我只拿出有关Schedulers。io这个方法涉及的逻辑代码进行讲解。首先，在注释1处，同前面分析的订阅流程的处理一样，只是一个处理hook的逻辑，最终返回的还是传入的这个IO对象。再看到注释2处，在Schedulers的静态代码块中将IO对象进行了初始化，其实质就是新建了一个IOTask的静态内部类，在IOTask的call方法中，也就是注释3处，可以了解到使用了静态内部类的方式把创建的IOScheduler对象给返回出去了。绕了这么大圈子，Schedulers.io方法其实质就是返回了一个IOScheduler对象。

##### 3. Observable#subscribeOn()
```java
public final Observable<T> subscribeOn(Scheduler scheduler) {
    ...

    return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
}
```

##### 4. ObservableSubscribeOn#
```java
public final class ObservableSubscribeOn<T> extends AbstractObservableWithUpstream<T, T> {
    final Scheduler scheduler;

    public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
        // 1
        super(source);
        this.scheduler = scheduler;
    }

    @Override
    public void subscribeActual(final Observer<? super T> observer) {
        // 2
        final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer);

        // 3
        observer.onSubscribe(parent);

        // 4
        parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
    }

...
}
```
首先，在注释1处，将传进来的source和scheduler保存起来。接着，等到实际订阅的时候，就会执行到这个subscribeActual方法，在注释2处，将我们自定义的Observer包装成了一个SubscribeOnObserver对象。在注释3处，通知观察者订阅了被观察者。在注释4处，内部先创建了一个SubscribeTask对象，来看看它的实现。

##### 5. ObservableSubscribeOn#SubscribeTask
```java
final class SubscribeTask implements Runnable {
    private final SubscribeOnObserver<T> parent;

    SubscribeTask(SubscribeOnObserver<T> parent) {
        this.parent = parent;
    }

    @Override
    public void run() {
        source.subscribe(parent);
    }
}
```

##### 6. Scheduler#scheduleDirect()
```java
public Disposable scheduleDirect(@NonNull Runnable run) {
    return scheduleDirect(run, 0L, TimeUnit.NANOSECONDS);
}

public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {

    // 1
    final Worker w = createWorker();

    // 2
    final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);

    // 3
    DisposeTask task = new DisposeTask(decoratedRun, w);

    // 4
    w.schedule(task, delay, unit);

    return task;
}
```
scheduleActual


enqueue

#### 线程切换

#### 参考

https://jsonchao.github.io/2019/01/01/Android%E4%B8%BB%E6%B5%81%E4%B8%89%E6%96%B9%E5%BA%93%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%EF%BC%88%E4%BA%94%E3%80%81%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3RxJava%E6%BA%90%E7%A0%81%EF%BC%89/