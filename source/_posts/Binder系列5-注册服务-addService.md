---
title: Binder系列5-注册服务(addService)
tags:
    - Binder
categories: Android
---

> Source Code Environment  
>
> Android Version Pie 9.0.0_r3  
> Kernel Version 3.18  
>
> 本文参考Gityuan大神的博客，结合高版本源码，讲解如何向ServiceManager注册Native层的服务的过程

```
Reference Source Code

// framework
/frameworks/native/libs/binder/ProcessState.cpp
/external/fio/os/windows/posix/include/sys/mman.h
/frameworks/av/media/libmediaplayerservice/MediaPlayerService.cpp
/frameworks/native/libs/binder/IServiceManager.cpp
/frameworks/native/libs/binder/Parcel.cpp
/frameworks/native/libs/binder/Binder.cpp
/frameworks/native/libs/binder/IPCThreadState.cpp
/frameworks/native/libs/binder/BpBinder.cpp
/frameworks/native/cmds/servicemanager/binder.c
/frameworks/native/cmds/servicemanager/service_manager.c

// kernel
/drivers/staging/android/binder.c
```

## # 1. 概述

### # 1.1 media服务注册

media入口函数是 `/frameworks/av/media/mediaserver/main_mediaserver.cpp`中的main()方法
```
int main(int argc __unused, char **argv __unused)
{
    // 获取 ProcessState 实例对象【见小节2.1】
    sp<ProcessState> proc(ProcessState::self());
    // 获取 BpServiceManager 对象
    sp<IServiceManager> sm(defaultServiceManager());
    InitializeIcuOrDie();
    // 注册多媒体服务【见小节3.1】
    MediaPlayerService::instantiate();
    ResourceManagerService::instantiate();
    registerExtensions();
    // 启动Binder线程池
    ProcessState::self()->startThreadPool();
    // 把当前线程加入到线程池
    IPCThreadState::self()->joinThreadPool();
}
```
调用流程：

    1. 获取ServiceManager：讲解了defaultServiceManager()返回的是BpServiceManager对象，用于跟ServiceManager进程通信；
    
    2. 理解Binder线程池的管理：讲解了startThreadPool()和joinThreadPool()过程
    
本文是为了讲解Native层注册服务的过程

### # 1.2 类图

下面以media服务为例，分析注册服务的过程（来自Gityuan大佬）

![add_media_player_service](https://tva1.sinaimg.cn/large/d7f9b0f4gy1gld47tdeegj20s60k2gog.jpg)

- 蓝色代表的是注册MediaPlayerService服务所涉及的类
- 绿色代表的是Binder架构中与Binder驱动通信过程中的核心类
- 紫色代表的是注册服务和获取服务的公共接口/父类

### # 1.3 时序图

根据时序图，来看看media服务启动过程是如何向ServiceManager注册服务的（来自Gityuan大佬）

![addService](https://tva4.sinaimg.cn/large/d7f9b0f4gy1gld4cpzg6fj20sa0q5414.jpg)

## # 2. ProcessState

### # 2.1 ProcessState::self

[/frameworks/native/libs/binder/ProcessState.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/ProcessState.cpp)
```
sp<ProcessState> ProcessState::self()
{
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != NULL) {
        return gProcess;
    }
    gProcess = new ProcessState("/dev/binder");
    return gProcess;
}
```
获得 ProcessState 对象：单例模式，保证了每个进程只有一个 ProcessState 对象。其中 gProcess 和 gProcessMutex 是保存在 Static.cpp 类的全局变量

### # 2.2 ProcessState初始化

[/frameworks/native/libs/binder/ProcessState.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/ProcessState.cpp)
```
ProcessState::ProcessState(const char *driver)
    : mDriverName(String8(driver))
    , mDriverFD(open_driver(driver)) // 打开Binder驱动【见小节2.3】
    , mVMStart(MAP_FAILED)
    , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
    , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
    , mExecutingThreadsCount(0)
    , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
    , mStarvationStartTimeMs(0)
    , mManagesContexts(false)
    , mBinderContextCheckFunc(NULL)
    , mBinderContextUserData(NULL)
    , mThreadPoolStarted(false)
    , mThreadPoolSeq(1)
{
    if (mDriverFD >= 0) {
        // 采用内存映射函数mmap，给binder分配一块虚拟地址空间【见小节2.4】
        mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        if (mVMStart == MAP_FAILED) {
            close(mDriverFD); // 没有足够空间分配给/dev/binder，则关闭驱动
            mDriverFD = -1;
            mDriverName.clear();
        }
    }
}
```
- ProcessState 的单例模式的唯一性，因此一个进程只打开binder设备一次，其中 ProcessState 的成员变量 mDriverFD 记录binder驱动的fd，用于访问binder设备
- BINDER_VM_SIZE = ((1 *  * ) - sysconf(_SC_PAGE_SIZE) * 2)，binder分配的默认内存大小为1M-8K
- DEFAULT_MAX_BINDER_THREADS = 15，binder默认的最大可并发访问的线程数为15

### # 2.3 open_driver

[/frameworks/native/libs/binder/ProcessState.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/ProcessState.cpp)
```
static int open_driver(const char *driver)
{
    int fd = open(driver, O_RDWR | O_CLOEXEC);
    if (fd >= 0) {
        int vers = 0;
        status_t result = ioctl(fd, BINDER_VERSION, &vers);
        if (result == -1) {
            close(fd);
            fd = -1;
        }
        if (result != 0 || vers != BINDER_CURRENT_PROTOCOL_VERSION) {
            close(fd);
            fd = -1;
        }
        size_t maxThreads = DEFAULT_MAX_BINDER_THREADS;
        result = ioctl(fd, BINDER_SET_MAX_THREADS, &maxThreads);
        if (result == -1) {
            ALOGE("Binder ioctl to set max threads failed: %s", strerror(errno));
        }
    } else {
        ALOGW("Opening '%s' failed: %s\n", driver, strerror(errno));
    }
    return fd;
}
```
open_driver作用是打开/dev/binder设备，设定binder支持的最大线程数。关于binder驱动的相应方法，见文章Binder Driver初探

ProcessState采用单例模式，保证每一个进程都只打开一次Binder Driver

### # 2.4 mmap

[/external/fio/os/windows/posix/include/sys/mman.h](http://androidxref.com/9.0.0_r3/xref/external/fio/os/windows/posix/include/sys/mman.h)
```
void *mmap(void *addr, size_t len, int prot, int flags,
		int fildes, off_t off);

int munmap(void *addr, size_t len);
```
参数说明：
- addr: 代表映射到进程地址空间的起始地址，当值等于0则由内核选择合适地址，此时为0
- size: 代表需要映射的内存地址空间的大小，此处为1M-8K
- prot: 代表内存映射区的读写等属性值，此处为PROT_READ(可读取)
- flags: 标志位，此处为MAP_PRIVATE(私有映射，多进程间不共享内容的改变)和MAP_NORESERVE(不保留交换空间)
- fd: 代表mmap所关联的文件描述符，此处为mDriverFD
- offset: 偏移量，此处为0

mmap()经过系统调用，执行binder_mmap过程

## # 3. 服务注册

### # 3.1 instantiate

[/frameworks/av/media/libmediaplayerservice/MediaPlayerService.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/av/media/libmediaplayerservice/MediaPlayerService.cpp)
```
void MediaPlayerService::instantiate() {
    // 注册服务【见小节3.2】
    defaultServiceManager()->addService(
            Stringmedia.player"), new MediaPlayerService());
}
```
注册服务 MediaPlayerService：由 defaultServiceManager()返回的是BpServiceManager，同时会创建ProcessState对象和BpBinder对象。
故此处等价于调用BpServiceManager->addService。其中MediaPlayerService位于libmediaplayerservice库

### # 3.2 BpServiceManager.addService

[/frameworks/native/libs/binder/IServiceManager.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/IServiceManager.cpp)
```
    virtual status_t addService(const Stringname, const sp<IBinder>& service,
                                bool allowIsolated, int dumpsysPriority) {
        Parcel data, reply; // Parcel是数据通信包
        // 写入头信息"android.os.IServiceManager"
        data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
        data.writeStringame); // name为"media.player"
        data.writeStrongBinder(service); // MediaPlayerService对象【见小节3.2.1】
        data.writeInt32(allowIsolated ? 1 : 0); // allowIsolated = false
        data.writeInt32(dumpsysPriority); 
        // remote()指向的是BpBinder对象【见小节3.3】
        status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);
        return err == NO_ERROR ? reply.readExceptionCode() : err;
    }
```
服务注册过程：向ServiceManager注册服务MediaPlayerService，服务名为"media.player"

#### # 3.2.1 writeStrongBinder

[/frameworks/native/libs/binder/Parcel.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/Parcel.cpp)
```
status_t Parcel::writeStrongBinder(const sp<IBinder>& val)
{
    return flatten_binder(ProcessState::self(), val, this);
}
```

#### # 3.2.2 flatten_binder

[/frameworks/native/libs/binder/Parcel.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/Parcel.cpp)
```
status_t flatten_binder(const sp<ProcessState>& /*proc*/,
    const sp<IBinder>& binder, Parcel* out)
{
    flat_binder_object obj;

    // 是否允许后台运行，设置节点优先级
    if (IPCThreadState::self()->backgroundSchedulingDisabled()) {
        /* minimum priority for all nodes is nice 0 */
        obj.flags = FLAT_BINDER_FLAG_ACCEPTS_FDS;
    } else {
        /* minimum priority for all nodes is MAX_NICE(19) */
        obj.flags = 0x13 | FLAT_BINDER_FLAG_ACCEPTS_FDS;
    }

    if (binder != NULL) {
        IBinder *local = binder->localBinder(); // 本地Binder不为空
        if (!local) {
            BpBinder *proxy = binder->remoteBinder();
            const int32_t handle = proxy ? proxy->handle() : 0;
            obj.hdr.type = BINDER_TYPE_HANDLE;
            obj.binder = 0; /* Don't pass uninitialized stack data to a remote process */
            obj.handle = handle;
            obj.cookie = 0;
        } else { // 进入该分支
            obj.hdr.type = BINDER_TYPE_BINDER;
            obj.binder = reinterpret_cast<uintptr_t>(local->getWeakRefs());
            obj.cookie = reinterpret_cast<uintptr_t>(local);
        }
    } else {
        obj.hdr.type = BINDER_TYPE_BINDER;
        obj.binder = 0;
        obj.cookie = 0;
    }
    // 【见小节3.2.4】
    return finish_flatten_binder(binder, obj, out);
}
```
将Binder对象扁平化，转换成flat_binder_object对象

- 对于Binder实体，则cookie记录Binder实体的指针
- 对于Binder代理，则用handle记录Binder代理的句柄

关于localBinder，代码见[/frameworks/native/libs/binder/Binder.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/Binder.cpp)
```
BBinder* IBinder::localBinder()
{
    return NULL;
}

BBinder* BBinder::localBinder()
{
    return this;
}
```

#### # 3.2.3 backgroundSchedulingDisabled

[/frameworks/native/libs/binder/IPCThreadState.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/IPCThreadState.cpp)
```
// 是否禁用后台运行，默认是false
static bool gDisableBackgroundScheduling = false;
bool IPCThreadState::backgroundSchedulingDisabled()
{
    return gDisableBackgroundScheduling;
}
```

#### # 3.2.4 finish_flatten_binder

[/frameworks/native/libs/binder/Parcel.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/Parcel.cpp)
```
inline static status_t finish_flatten_binder(
    const sp<IBinder>& /*binder*/, const flat_binder_object& flat, Parcel* out)
{
    return out->writeObject(flat, false);
}
```
将flat_binder_object写入out

### # 3.3 BpBinder::transact

[/frameworks/native/libs/binder/BpBinder.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/BpBinder.cpp)
```
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        // code = ADD_SERVICE_TRANSACTION【见小节3.4】
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }

    return DEAD_OBJECT;
}
```
Binder代理类调用transact()方法，真正工作还是交给 IPCThreadState 来进行 transact 工作。那么来看看 IPCThreadState 初始化的过程

#### # 3.3.1 IPCThreadState::self

[/frameworks/native/libs/binder/IPCThreadState.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/IPCThreadState.cpp)
```
IPCThreadState* IPCThreadState::self()
{
    if (gHaveTLS) {
restart:
        const pthread_key_t k = gTLS;
        IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
        if (st) return st;
        return new IPCThreadState; // 初始化 IPCThreadState【见小节3.3.2】
    }

    if (gShutdown) {
        return NULL;
    }

    pthread_mutex_lock(&gTLSMutex);
    if (!gHaveTLS) { // 首次进入gHaveTLS为false
        int key_create_value = pthread_key_create(&gTLS, threadDestructor); // 创建线程的TLS
        if (key_create_value != 0) {
            pthread_mutex_unlock(&gTLSMutex);
            return NULL;
        }
        gHaveTLS = true;
    }
    pthread_mutex_unlock(&gTLSMutex);
    goto restart;
}
```
TLS是指 Thread Local Storage(线程本地储存空间)，每个线程都有自己的TLS，并且是私有空间，线程之间不会共享。
通过 pthread_getspecific/pthread_setspecific 函数可以获取/设置这些空间中的内容。
从线程本地存储空间中获得保存在其中的 IPCThreadState 对象。

#### # 3.3.2 IPCThreadState初始化

[/frameworks/native/libs/binder/IPCThreadState.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/IPCThreadState.cpp)
```
IPCThreadState::IPCThreadState()
    : mProcess(ProcessState::self()),
      mStrictModePolicy(0),
      mLastTransactionBinderFlags(0)
{
    pthread_setspecific(gTLS, this);
    clearCaller();
    mIn.setDataCapacity(256);
    mOut.setDataCapacity(256);
}
```
每个线程都有一个IPCThreadState，每个IPCThreadState中都有一个mIn、一个mOut。成员变量mProcess保存了ProcessState变量（每个进程只有一个）
- mIn用来接收来自Binder设备的数据，默认大小为256字节
- mOut用来存储发往Binder设备的数据，默认大小为256字节

### # 3.4 IPCThreadState::transact

[/frameworks/native/libs/binder/IPCThreadState.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/IPCThreadState.cpp)
```
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    status_t err;
    flags |= TF_ACCEPT_FDS;
    // 传输数据
    err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);

    if (err != NO_ERROR) {
        if (reply) reply->setError(err);
        return (mLastError = err);
    }

    if ((flags & TF_ONE_WAY) == 0) {
        if (reply) {
            err = waitForResponse(reply); // 等待响应
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
    } else {
        err = waitForResponse(NULL, NULL);
    }
    return err;
}
```
IPCThreadState进行transact事务处理分为两部分：
- writeTransactionData(): 传输数据
- waitForResponse(): 等待响应

### # 3.5 IPCThreadState.writeTransactionData

[/frameworks/native/libs/binder/IPCThreadState.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/IPCThreadState.cpp)
```
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    binder_transaction_data tr;
    tr.target.ptr = 0; /* Don't pass uninitialized stack data to a remote process */
    tr.target.handle = handle; // handle = 0
    tr.code = code; // code = ADD_SERVICE_TRANSACTION
    tr.flags = binderFlags; // binderFlags = 0
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;

    // data为记录Media服务信息的Parcel对象，进行数据错误检查
    const status_t err = data.errorCheck();
    if (err == NO_ERROR) {
        tr.data_size = data.ipcDataSize(); // mDataSize
        tr.data.ptr.buffer = data.ipcData(); // mData
        tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t); // mObjectsSize
        tr.data.ptr.offsets = data.ipcObjects(); // mObjects
    } else if (statusBuffer) {
        tr.flags |= TF_STATUS_CODE;
        *statusBuffer = err;
        tr.data_size = sizeof(status_t);
        tr.data.ptr.buffer = reinterpret_cast<uintptr_t>(statusBuffer);
        tr.offsets_size = 0;
        tr.data.ptr.offsets = 0;
    } else {
        return (mLastError = err);
    }
    mOut.writeInt32(cmd); // cmd = BC_TRANSACTION
    mOut.write(&tr, sizeof(tr)); // 写入binder_transaction_data数据
    return NO_ERROR;
}
```
其中handle的值用来标示目的端，注册服务过程的目的端为service manager，此处handle=0所对应的是 binder_context_mgr_node 对象，正是service manager所对应的binder实体对象。
binder_transaction_data 结构体是binder驱动通信的数据结构，该过程最终是把Binder请求码 BC_TRANSACTION 和 binder_transaction_data 结构体写入到 mOut

transact 过程，先写完 binder_transaction_data 数据，其中 Parcel data 的重要成员变量：
- mDataSize：保存在 data_size，binder_transaction_data 的数据大小
- mData：保存在 ptr.buffer，binder_transaction_data 的数据的起始地址
- mObjectsSize：保存在 offsets_size，记录了 flat_binder_object 结构体的个数
- mObjects：保存在 ptr.offsets，记录了 flat_binder_object 结构体在数据中的偏移量

接下来执行 waitForResponse()方法

### # 3.6 IPCThreadState.waitForResponse

[/frameworks/native/libs/binder/IPCThreadState.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/IPCThreadState.cpp)
```
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;
    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break; // 【见小节3.7】
        err = mIn.errorCheck(); // 数据错误检查
        if (err < NO_ERROR) break;
        if (mIn.dataAvail() == 0) continue;

        cmd = (uint32_t)mIn.readInt32();
        switch (cmd) {
        case BR_TRANSACTION_COMPLETE:
        case BR_DEAD_REPLY:
        case BR_FAILED_REPLY:
        case BR_ACQUIRE_RESULT:
            goto finish;

        case BR_REPLY:
            {
                binder_transaction_data tr;
                err = mIn.read(&tr, sizeof(tr));
                if (err != NO_ERROR) goto finish; // 如果数据报错，直接跳转结束

                if (reply) {
                    if ((tr.flags & TF_STATUS_CODE) == 0) {
                        // 将返回的 binder_transaction_data 结构体数据保存到 Parcel 对象中
                        reply->ipcSetDataReference(
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t),
                            freeBuffer, this);
                    } else {
                        // 释放资源...
                    }
                } else {
                    // 释放资源...
                }
            }
            goto finish;

        default:
            // 默认执行命令，见小节3.8
            err = executeCommand(cmd);
            if (err != NO_ERROR) goto finish;
            break;
        }
    }
    // finish...
    return err;
}
```
调用过程：
1. 首先执行 BR_TRANSACTION_COMPLETE 命令
2. 当目标进程收到事务后，处理 BR_TRANSACTION 事务
3. 然后发送给当前进程，再执行 BR_REPLY 命令，读取接收数据

### # 3.7 IPCThreadState.talkWithDriver

[/frameworks/native/libs/binder/IPCThreadState.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/IPCThreadState.cpp)
```
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    binder_write_read bwr;

    const bool needRead = mIn.dataPosition() >= mIn.dataSize();
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;

    bwr.write_size = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data();

    if (doReceive && needRead) {
        // 接收数据缓冲区信息的填充，如果以后收到数据，就直接填充在mIn中
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }

    // 当读写缓冲都为空，则直接返回
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
        // 通过 ioctl()一直进行读写操作，与Binder Driver进行通信
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        else
            err = -errno;
    } while (err == -EINTR); // 当被中断，则继续执行

    if (err >= NO_ERROR) {
        // 当存在已经写入的命令协议时，移除mOut中对应的数据
        if (bwr.write_consumed > 0) {
            if (bwr.write_consumed < mOut.dataSize())
                mOut.remove(0, bwr.write_consumed);
            else {
                mOut.setDataSize(0);
                processPostWriteDerefs();
            }
        }
        // 当存在已经读取的返回协议时，保存数据到mIn中
        if (bwr.read_consumed > 0) {
            mIn.setDataSize(bwr.read_consumed);
            mIn.setDataPosition(0);
        }
        return NO_ERROR;
    }
    return err;
}
```
binder_write_read 结构体用来与Binder设备交换数据，通过 ioctl 与 mDriverFD通信，是真正与Binder驱动进行数据读写交互的过程。主要是操作mOut和mIn变量

ioctl() 经过系统调用后进入 Binder Driver

### # 3.8 executeCommand

[/frameworks/native/libs/binder/IPCThreadState.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/IPCThreadState.cpp)
```
status_t IPCThreadState::executeCommand(int32_t cmd)
{
    BBinder* obj;
    RefBase::weakref_type* refs;
    status_t result = NO_ERROR;

    switch ((uint32_t)cmd) {
    case BR_ERROR: ...
    case BR_OK:
    case BR_ACQUIRE: ...
    case BR_RELEASE: ...
    case BR_INCREFS: ...
    case BR_DECREFS: ...
    case BR_ATTEMPT_ACQUIRE: ...
    case BR_DEAD_BINDER: ...
    case BR_CLEAR_DEATH_NOTIFICATION_DONE: ...
    case BR_FINISHED:
    case BR_NOOP:
        break;
        
    case BR_TRANSACTION:
        {
            binder_transaction_data tr;
            result = mIn.read(&tr, sizeof(tr));
            if (result != NO_ERROR) break;

            Parcel buffer;
            // 读取 binder_transaction_data 结构体保存到 Parcel 对象中
            buffer.ipcSetDataReference(
                reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                tr.data_size,
                reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                tr.offsets_size/sizeof(binder_size_t), freeBuffer, this);

            const pid_t origPid = mCallingPid;
            const uid_t origUid = mCallingUid;
            const int32_t origStrictModePolicy = mStrictModePolicy;
            const int32_t origTransactionBinderFlags = mLastTransactionBinderFlags;

            mCallingPid = tr.sender_pid;
            mCallingUid = tr.sender_euid;
            mLastTransactionBinderFlags = tr.flags;

            Parcel reply;
            status_t error;
            if (tr.target.ptr) {
                if (reinterpret_cast<RefBase::weakref_type*>(
                        tr.target.ptr)->attemptIncStrong(this)) {
                    error = reinterpret_cast<BBinder*>(tr.cookie)->transact(tr.code, buffer,
                            &reply, tr.flags);
                    reinterpret_cast<BBinder*>(tr.cookie)->decStrong(this);
                } else {
                    error = UNKNOWN_TRANSACTION;
                }

            } else {
                error = the_context_object->transact(tr.code, buffer, &reply, tr.flags);
            }

            if ((tr.flags & TF_ONE_WAY) == 0) {
                if (error < NO_ERROR) reply.setError(error);
                sendReply(reply, 0);
            }

            mCallingPid = origPid;
            mCallingUid = origUid;
            mStrictModePolicy = origStrictModePolicy;
            mLastTransactionBinderFlags = origTransactionBinderFlags;
        }
        break;
    case BR_SPAWN_LOOPER:
        // 创建新的Binder线程
        mProcess->spawnPooledThread(false);
        break;
    return result;
}
```

## # 4. Binder Driver

ioctl -> binder_ioctl -> binder_ioctl_write_read

### # 4.1 binder_ioctl_write_read

[/drivers/staging/android/binder.c](http://androidxref.com/kernel_3.18/xref/drivers/staging/android/binder.c)
```
static int binder_ioctl_write_read(struct file *filp,
				unsigned int cmd, unsigned long arg,
				struct binder_thread *thread)
{
	int ret = 0;
	struct binder_proc *proc = filp->private_data;
	unsigned int size = _IOC_SIZE(cmd);
	void __user *ubuf = (void __user *)arg;
	struct binder_write_read bwr;
    // 检查数据大小
	if (size != sizeof(struct binder_write_read)) {
		ret = -EINVAL;
		goto out;
	}
    // 将用户空间 binder_write_read 结构体拷贝到内核空间
    copy_from_user(&bwr, ubuf, sizeof(bwr))
    // ...
	if (bwr.write_size > 0) {
        // 将数据放入目标进程【见小节4.2】
		ret = binder_thread_write(proc, thread,
					  bwr.write_buffer,
					  bwr.write_size,
					  &bwr.write_consumed);
        // ...
	}
	if (bwr.read_size > 0) {
        // 读取自己进程的数据【见小节4.3】
		ret = binder_thread_read(proc, thread, bwr.read_buffer,
					 bwr.read_size,
					 &bwr.read_consumed,
					 filp->f_flags & O_NONBLOCK);
		if (!list_empty(&proc->todo))
			wake_up_interruptible(&proc->wait);
        // ...
	}
    // 将内核空间 binder_write_read 结构体拷贝到用户空间
    copy_to_user(ubuf, &bwr, sizeof(bwr))
    // ...
}
```

### # 4.2 binder_thread_write

[/drivers/staging/android/binder.c](http://androidxref.com/kernel_3.18/xref/drivers/staging/android/binder.c)
```
static int binder_thread_write(struct binder_proc *proc,
			struct binder_thread *thread,
			binder_uintptr_t binder_buffer, size_t size,
			binder_size_t *consumed)
{
	uint32_t cmd;
	void __user *buffer = (void __user *)(uintptr_t)binder_buffer;
	void __user *ptr = buffer + *consumed;
	void __user *end = buffer + size;

	while (ptr < end && thread->return_error == BR_OK) {
        // 拷贝用户空间的 cmd 命令，此时为 BC_TRANSACTION
		if (get_user(cmd, (uint32_t __user *)ptr)) return -EFAULT;
		ptr += sizeof(uint32_t);
        // ...
		switch (cmd) {
        // ...
		case BC_TRANSACTION:
		case BC_REPLY: {
			struct binder_transaction_data tr;
            // 拷贝用户空间的 binder_transaction_data
			if (copy_from_user(&tr, ptr, sizeof(tr))) return -EFAULT;
			ptr += sizeof(tr);
            // 见小节4.3
			binder_transaction(proc, thread, &tr, cmd == BC_REPLY);
			break;
		}
        // ...
		*consumed = ptr - buffer;
	}
	return 0;
}
```

### # 4.3 binder_transaction

[/drivers/staging/android/binder.c](http://androidxref.com/kernel_3.18/xref/drivers/staging/android/binder.c)
```
static void binder_transaction(struct binder_proc *proc,
			       struct binder_thread *thread,
			       struct binder_transaction_data *tr, int reply)
{
	struct binder_transaction *t;
	struct binder_work *tcomplete;
    // ...

	if (reply) {
        // ...
	} else {
		if (tr->target.handle) {
            // ...
		} else {
            // handle = 0 则找到servicemanager实体
			target_node = binder_context_mgr_node;
		}
		target_proc = target_node->proc;
        // ...
	}
	if (target_thread) {
		// ...
	} else {
        // 找到serivcemanager进程的todo队列
		target_list = &target_proc->todo;
		target_wait = &target_proc->wait;
	}

	t = kzalloc(sizeof(*t), GFP_KERNEL);
	tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);

    // 非 TF_ONE_WAY 的通信方式，把当前 thread 保存到 transaction 的 from 字段
	if (!reply && !(tr->flags & TF_ONE_WAY))
		t->from = thread;
	else
		t->from = NULL;

	t->sender_euid = task_euid(proc->tsk);
	t->to_proc = target_proc; // 此次通信目标进程为 servicemanager进程
	t->to_thread = target_thread;
	t->code = tr->code; // 此次通信 code = ADD_SERVICE_TRANSACTION
	t->flags = tr->flags; // 此次通信 flags = 0
	t->priority = task_nice(current);

    // 从servicemanager进程中分配buffer
	t->buffer = binder_alloc_buf(target_proc, tr->data_size,
		tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));

	t->buffer->allow_user_free = 0;
	t->buffer->transaction = t;
	t->buffer->target_node = target_node;

    // 引用计数+1
	if (target_node)
		binder_inc_node(target_node, 1, 0, NULL);

	offp = (binder_size_t *)(t->buffer->data +
				 ALIGN(tr->data_size, sizeof(void *)));

    // 拷贝用户空间的 binder_transaction_data 中的 ptr.buffer 到内核
	if (copy_from_user(t->buffer->data, (const void __user *)(uintptr_t)
			   tr->data.ptr.buffer, tr->data_size)) {
		// ...
	}
    // 拷贝用户空间的 binder_transaction_data 中的 ptr.offsets 到内核
	if (copy_from_user(offp, (const void __user *)(uintptr_t)
			   tr->data.ptr.offsets, tr->offsets_size)) {
		// ...
	}
	off_end = (void *)offp + tr->offsets_size;
	off_min = 0;
	for (; offp < off_end; offp++) {
		struct flat_binder_object *fp;
		fp = (struct flat_binder_object *)(t->buffer->data + *offp);
		off_min = *offp + sizeof(struct flat_binder_object);
		switch (fp->type) {
		case BINDER_TYPE_BINDER:
		case BINDER_TYPE_WEAK_BINDER: {
			struct binder_ref *ref;
            // 见【4.3.1】
			struct binder_node *node = binder_get_node(proc, fp->binder);

			if (node == NULL) {
                // 服务所在进程 创建 binder_node 实体【见4.3.2】
				node = binder_new_node(proc, fp->binder, fp->cookie);
                // ...
			}
            // servicemanager 进程 binder_ref【见4.3.3】
			ref = binder_get_ref_for_node(target_proc, node);
            // ...
            // 调整type为HANDLE类型
			if (fp->type == BINDER_TYPE_BINDER)
				fp->type = BINDER_TYPE_HANDLE;
			else
				fp->type = BINDER_TYPE_WEAK_HANDLE;
			fp->handle = ref->desc; // 设置 handle 值
			binder_inc_ref(ref, fp->type == BINDER_TYPE_HANDLE,
				       &thread->todo);
		} break;
        // ...
	}
	if (reply) {
		// ...
	} else if (!(t->flags & TF_ONE_WAY)) {
        // BC_TRANSACTION 且非 TF_ONE_WAY，则设置事务栈信息
		t->need_reply = 1;
		t->from_parent = thread->transaction_stack;
		thread->transaction_stack = t;
	} else {
		// ...
	}
    // 将 BINDER_WORK_TRANSACTION 添加到目标队列，本次通信的目标队列为 target_proc->todo
	t->work.type = BINDER_WORK_TRANSACTION;
	list_add_tail(&t->work.entry, target_list);

    // 将 BINDER_WORK_TRANSACTION_COMPLETE 添加到当前线程的todo队列
	tcomplete->type = BINDER_WORK_TRANSACTION_COMPLETE;
	list_add_tail(&tcomplete->entry, &thread->todo);

    // 唤醒等待队列，本次通信的目标队列为 target_proc->wait
	if (target_wait)
		wake_up_interruptible(target_wait);
	return;
// ...
}
```
注册服务的过程，传递的是BBinder对象，所以【小节3.2.1】的writeStrongBinder()过程中 localBinder 不为空，从而 flat_binder_object.type 等于 BINDER_TYPE_BINDER

服务注册过程是在服务所在进程创建 binder_node，在 ServiceManager 进程创建 binder_ref。对于同一个 binder_node，每个进程只会创建一个 binder_ref 对象

向 ServiceManager 的 binder_proc->todo 添加 BINDER_WORK_TRANSACTION_COMPLETE 事务，接下来进入 ServiceManager 进程

#### # 4.3.1 binder_get_node

```
static struct binder_node *binder_get_node(struct binder_proc *proc,
					   binder_uintptr_t ptr)
{
	struct rb_node *n = proc->nodes.rb_node;
	struct binder_node *node;

	while (n) {
		node = rb_entry(n, struct binder_node, rb_node);

		if (ptr < node->ptr)
			n = n->rb_left;
		else if (ptr > node->ptr)
			n = n->rb_right;
		else
			return node;
	}
	return NULL;
}
```
根据 binder 指针 ptr 值从 binder_proc 来查询响应的 binder_node

#### # 4.3.2 binder_new_node

```
static struct binder_node *binder_new_node(struct binder_proc *proc,
					   binder_uintptr_t ptr,
					   binder_uintptr_t cookie)
{
	struct rb_node **p = &proc->nodes.rb_node;
	struct rb_node *parent = NULL;
	struct binder_node *node;

    // 红黑树位置查找
	while (*p) {
		parent = *p;
		node = rb_entry(parent, struct binder_node, rb_node);

		if (ptr < node->ptr)
			p = &(*p)->rb_left;
		else if (ptr > node->ptr)
			p = &(*p)->rb_right;
		else
			return NULL;
	}
    // 将新创建的 binder_node 分配内核空间
	node = kzalloc(sizeof(*node), GFP_KERNEL);

    // 将新创建的 node 添加到 proc 红黑树
	rb_link_node(&node->rb_node, parent, p);

	rb_insert_color(&node->rb_node, &proc->nodes);
	node->debug_id = ++binder_last_id;
	node->proc = proc;
	node->ptr = ptr;
	node->cookie = cookie;
	node->work.type = BINDER_WORK_NODE; // 设置 binder_work 的 type
	INIT_LIST_HEAD(&node->work.entry);
	INIT_LIST_HEAD(&node->async_todo);
	return node;
}
```

#### # 4.3.3 binder_get_ref_for_node

```
static struct binder_ref *binder_get_ref_for_node(struct binder_proc *proc,
						  struct binder_node *node)
{
	struct rb_node *n;
	struct rb_node **p = &proc->refs_by_node.rb_node;
	struct rb_node *parent = NULL;
	struct binder_ref *ref, *new_ref;

    // 从 refs_by_node 红黑树，找到 binder_ref 则直接返回
	while (*p) {
		parent = *p;
		ref = rb_entry(parent, struct binder_ref, rb_node_node);

		if (node < ref->node)
			p = &(*p)->rb_left;
		else if (node > ref->node)
			p = &(*p)->rb_right;
		else
			return ref;
	}
    // 将新创建的 bind_ref 分配内核空间
	new_ref = kzalloc(sizeof(*ref), GFP_KERNEL);

	new_ref->debug_id = ++binder_last_id;
	new_ref->proc = proc; // 记录进程信息
	new_ref->node = node; // 记录binder节点
	rb_link_node(&new_ref->rb_node_node, parent, p);
	rb_insert_color(&new_ref->rb_node_node, &proc->refs_by_node);

    // 计算 binder 引用的 handle 值，该值返回给 target_proc 进程
	new_ref->desc = (node == binder_context_mgr_node) ? 0 : 1;
    // 从红黑树最左边的handle对比，依次递增，直到红黑树遍历结束或者找到更大的handle则结束
	for (n = rb_first(&proc->refs_by_desc); n != NULL; n = rb_next(n)) {
		ref = rb_entry(n, struct binder_ref, rb_node_desc);
		if (ref->desc > new_ref->desc)
			break;
		new_ref->desc = ref->desc + 1;
	}

    // 将新创建的 new_ref 传入 proc->refs_by_desc 红黑树
	p = &proc->refs_by_desc.rb_node;
	while (*p) {
		parent = *p;
		ref = rb_entry(parent, struct binder_ref, rb_node_desc);

		if (new_ref->desc < ref->desc)
			p = &(*p)->rb_left;
		else if (new_ref->desc > ref->desc)
			p = &(*p)->rb_right;
		else
			BUG();
	}
	rb_link_node(&new_ref->rb_node_desc, parent, p);
	rb_insert_color(&new_ref->rb_node_desc, &proc->refs_by_desc);
	if (node) {
		hlist_add_head(&new_ref->node_entry, &node->refs);
	} 
	return new_ref;
}
```
handle 值计算方法规律：
- 每个进程 binder_proc 所记录的 binder_ref 的 handle 值是从 1 开始递增的
- 所有进程 binder_proc 所记录的 handle = 0 的 binder_ref 都指向 ServiceManager
- 同一个服务的 binder_node 在不同进程的 binder_ref 的 handle 值可以不同

## # 5. ServiceManager

由 [Binder系列3-启动ServiceManager]() 已介绍其原理，循环在 binder_loop()过程，会调用 binder_parse()方法

### # 5.1 binder_parse

[/frameworks/native/cmds/servicemanager/binder.c](http://androidxref.com/9.0.0_r3/xref/frameworks/native/cmds/servicemanager/binder.c#217)
```
int binder_parse(struct binder_state *bs, struct binder_io *bio,
                 uintptr_t ptr, size_t size, binder_handler func)
{
    int r = 1;
    uintptr_t end = ptr + (uintptr_t) size;

    while (ptr < end) {
        uint32_t cmd = *(uint32_t *) ptr;
        ptr += sizeof(uint32_t);
        switch(cmd) {
        // ...
        case BR_TRANSACTION: {
            struct binder_transaction_data *txn = (struct binder_transaction_data *) ptr;
            // ...
            binder_dump_txn(txn);
            if (func) {
                unsigned rdata[256/4];
                struct binder_io msg;
                struct binder_io reply;
                int res;

                bio_init(&reply, rdata, sizeof(rdata), 4);
                bio_init_from_txn(&msg, txn); // 从txn解析出 binder_io 信息
                // 收到 Binder 事务【见小节5.2】
                res = func(bs, txn, &msg, &reply);
                if (txn->flags & TF_ONE_WAY) {
                    binder_free_buffer(bs, txn->data.ptr.buffer);
                } else {
                    // 发送 reply 事件【见小节5.4】
                    binder_send_reply(bs, &reply, txn->data.ptr.buffer, res);
                }
            }
            ptr += sizeof(*txn);
            break;
        }
        // ...
    }
    return r;
}
```

### # 5.2 svcmgr_handler

[/frameworks/native/cmds/servicemanager/service_manager.c](http://androidxref.com/9.0.0_r3/xref/frameworks/native/cmds/servicemanager/service_manager.c#svcmgr_handler)
```
int svcmgr_handler(struct binder_state *bs,
                   struct binder_transaction_data *txn,
                   struct binder_io *msg,
                   struct binder_io *reply)
{
    struct svcinfo *si;
    uint16_t *s;
    size_t len;
    uint32_t handle;
    uint32_t strict_policy;
    int allow_isolated;
    uint32_t dumpsys_priority;
    // ...
    strict_policy = bio_get_uint32(msg);
    s = bio_get_string16(msg, &len);
    // ...
    switch(txn->code) {
    // ...
    case SVC_MGR_ADD_SERVICE:
        s = bio_get_string16(msg, &len);
        // ...
        handle = bio_get_ref(msg); // 获取 handle
        allow_isolated = bio_get_uint32(msg) ? 1 : 0;
        dumpsys_priority = bio_get_uint32(msg);
        // 注册指定服务【见小节5.3】
        if (do_add_service(bs, s, len, handle, txn->sender_euid, allow_isolated, dumpsys_priority,
                           txn->sender_pid))
            return -1;
        break;
    // ...
    }
    bio_put_uint32(reply, 0);
    return 0;
}
```

### # 5.3 do_add_service

[/frameworks/native/cmds/servicemanager/service_manager.c](http://androidxref.com/9.0.0_r3/xref/frameworks/native/cmds/servicemanager/service_manager.c#svcmgr_handler)
```
int do_add_service(struct binder_state *bs, const uint16_t *s, size_t len, uint32_t handle,
                   uid_t uid, int allow_isolated, uint32_t dumpsys_priority, pid_t spid) {
    struct svcinfo *si;

    if (!handle || (len == 0) || (len > 127))
        return -1;

    // 权限检查
    if (!svc_can_register(s, len, spid, uid)) {
        return -1;
    }
    
    // 服务检索
    si = find_svc(s, len);
    if (si) {
        if (si->handle) {
            svcinfo_death(bs, si); // 服务已注册时，释放相应的服务
        }
        si->handle = handle;
    } else {
        si = malloc(sizeof(*si) + (len + 1) * sizeof(uint16_t));
        if (!si) { // 内存不足，无法分配足够内存
            return -1;
        }
        si->handle = handle;
        si->len = len;
        memcpy(si->name, s, (len + 1) * sizeof(uint16_t)); // 内存拷贝服务信息
        si->name[len] = '\0';
        si->death.func = (void*) svcinfo_death;
        si->death.ptr = si;
        si->allow_isolated = allow_isolated;
        si->dumpsys_priority = dumpsys_priority;
        si->next = svclist; // svclist保存所有已注册的服务
        svclist = si;
    }

    // 以 BC_ACQUIRE 命令，handle为目标的信息，通过ioctl发送给binder驱动
    binder_acquire(bs, handle);
    // 以 BC_REQUEST_DEATH_NOTIFICATION 命令的信息，通过ioctl发送给binder驱动，主要用于清理内存等
    binder_link_to_death(bs, handle, &si->death);
    return 0;
}
```
svcinfo 记录着服务名和 handle 信息，保存到 svclist 列表

### # 5.4 binder_send_reply

[/frameworks/native/cmds/servicemanager/binder.c](http://androidxref.com/9.0.0_r3/xref/frameworks/native/cmds/servicemanager/binder.c#217)
```
void binder_send_reply(struct binder_state *bs,
                       struct binder_io *reply,
                       binder_uintptr_t buffer_to_free,
                       int status)
{
    struct {
        uint32_t cmd_free;
        binder_uintptr_t buffer;
        uint32_t cmd_reply;
        struct binder_transaction_data txn;
    } __attribute__((packed)) data;

    data.cmd_free = BC_FREE_BUFFER; // free buffer命令
    data.buffer = buffer_to_free;
    data.cmd_reply = BC_REPLY; // reply 命令
    data.txn.target.ptr = 0;
    data.txn.cookie = 0;
    data.txn.code = 0;
    if (status) {
        // ...
    } else {
        data.txn.flags = 0;
        data.txn.data_size = reply->data - reply->data0;
        data.txn.offsets_size = ((char*) reply->offs) - ((char*) reply->offs0);
        data.txn.data.ptr.buffer = (uintptr_t)reply->data0;
        data.txn.data.ptr.offsets = (uintptr_t)reply->offs0;
    }
    // 向binder驱动通信
    binder_write(bs, &data, sizeof(data));
}
```
binder_write 进入 binder 驱动后，将 BC_FREE_BUFFER 和 BC_REPLY 命令协议发送给 Binder 驱动，向 client 端发送 reply

### # 5.4 binder_write

```
int binder_write(struct binder_state *bs, void *data, size_t len)
{
    struct binder_write_read bwr;
    int res;

    bwr.write_size = len;
    bwr.write_consumed = 0;
    bwr.write_buffer = (uintptr_t) data;
    bwr.read_size = 0;
    bwr.read_consumed = 0;
    bwr.read_buffer = 0;
    // 以 BC_REPLY 命令的信息，通过ioctl发送给binder驱动，
    res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);

    return res;
}
```

## # 6. 总结

服务注册过程（addService）核心功能：在服务所在进程创建 binder_node，在 ServiceManager 进程创建 bind_ref。其中 bind_ref 的 handle 在同一个进程中是唯一的：
- 每个进程 binder_proc 所记录的 binder_ref 的 handle 值是从 1 开始递增的
- 所有进程 binder_proc 所记录的 handle = 0 的 binder_ref 都指向 ServiceManager
- 同一个服务的 binder_node 在不同进程的 binder_ref 的 handle 值可以不同

Media 服务注册的过程涉及到 MediaPlayerService（作为Client进程）和 ServiceManager（作为Service进程），通信流程图：(来自Gityuan大佬)

![media_player_service_ipc](https://tva4.sinaimg.cn/large/d7f9b0f4gy1glhqk8iq93j20qh0kb74y.jpg)

过程分析：

1. MediaPlayerService 进程调用 ioctl()向Binder驱动发送IPC数据，该过程可以理解成一个事务 binder_transaction（记为T1），
    执行当前操作的线程 binder_thread（记为thread1），则 T1->from_parent=NULL, T1->from=thread1, thread1->transaction_stack=T1。
    其中IPC数据内容包括：
    - Binder协议为 BC_TRANSACTION
    - Handle = 0
    - RPC代码为 ADD_SERVICE
    - RPC数据为 "media.player"
    
2. Binder 驱动收到该 Binder 请求，生成 BR_TRANSACTION 命令，选择目标处理该请求的线程，即 ServiceManager 的binder线程（记为thread2），
    则 T1->to_parent=NULL, T1->to_thread=thread2。并将整个binder_transaction（记为T2）数据插入到目标线程的todo队列

3. ServiceManager的线程 thead2 收到 T2后，调用服务注册函数将服务 "media.player"注册到服务目录中。当服务注册完成后，生成IPC应答数据（BC_REPLY），T2->from_parent=T1,
    T2->from = thread2, thread2->transaction_stack = T2
    
4. Binder驱动收到该Binder应答请求，生成 BR_RETRY 命令，T2->to_parent = T1, T2->to_thread=thread1, thread->transaction_stack=T2.
    在MediaPlayerService收到该命令后，知道服务注册完成便可以正常使用

整个过程中，BC_TRANSACTION 和 BR_TRANSACTION 过程是一个完整的事务过程；BC_REPLY 和 BR_REPLY 是一个完整的事务过程。
到此，其他进程便可以获取该服务，使用服务提供的方法，下一篇文章将会讲述[如何获取服务]()。

## # 7. 参考

http://gityuan.com/2015/11/14/binder-add-service/

