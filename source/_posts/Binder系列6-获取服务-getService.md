---
title: Binder系列6-获取服务-getService
tags:
    - Binder
categories: Android
---

> Source Code Environment  
>
> Android Version Pie 9.0.0_r 
> Kernel Version 3.18  
>
> 本文参考Gityuan大神的博客，结合高版本源码，讲解Client如何向Server获取服务的过程

```
Reference Source Code

// framework
/frameworks/av/media/libmedia/IMediaDeathNotifier.cpp
/frameworks/native/libs/binder/IServiceManager.cpp
/frameworks/native/libs/binder/BpBinder.cpp
/frameworks/native/libs/binder/IPCThreadState.cpp
/frameworks/native/libs/binder/Parcel.cpp
/frameworks/native/libs/binder/ProcessState.cpp

// kernel
/drivers/staging/android/binder.c
```

# # 1. 获取服务

在Native层的服务注册，以media为例展开详解，先来看看media的类关系图

## # 1.1 类图

![get_media_player_service](https://tva1.sinaimg.cn/large/d7f9b0f4gy1glik8ezr1yj20s60kaq5t.jpg)

- 蓝色：代表获取 MediaPlayerService 服务相关的类
- 绿色：代表 Binder 架构中与 Binder 驱动通信过程中的核心类
- 紫色：代表注册服务和获取服务的公共接口/父类

# # 2. 获取Media服务

## # 2.1 getMediaPlayerService

[/frameworks/av/media/libmedia/IMediaDeathNotifier.cpp](http://androidxref.com/9.0.0_rxref/frameworks/av/media/libmedia/IMediaDeathNotifier.cpp#)
```
/*static*/const sp<IMediaPlayerService>
IMediaDeathNotifier::getMediaPlayerService()
{
    Mutex::Autolock _l(sServiceLock);
    if (sMediaPlayerService == 0) {
        sp<IServiceManager> sm = defaultServiceManager(); // 获取 ServiceManager
        sp<IBinder> binder;
        do {
            // 获取名为 "media.player" 的服务【见小节2.2】
            binder = sm->getService(Stringmedia.player"));
            if (binder != 0) {
                break;
            }
            usleep(500000); // 0.5s
        } while (true);

        if (sDeathNotifier == NULL) {
            sDeathNotifier = new DeathNotifier(); // 创建死亡通知对象
        }
        // 将死亡通知连接到 binder
        binder->linkToDeath(sDeathNotifier);
        sMediaPlayerService = interface_cast<IMediaPlayerService>(binder);
    }
    return sMediaPlayerService;
}
```
其中 defaultServiceManager()过程在[获取ServiceManager]()已分析，返回 BpServiceManager

在请求获取名为"media.player"的服务过程中，采用不断循环获取的方法。由于 MediaPlayerService 服务可能还没向 ServiceManager 注册完成或者尚未启动完成等情况，
所以当 binder 返回为 NULL 时，休眠 0.5s 后继续请求，直到获取服务

## # 2.2 BpServiceManager.getService

[/frameworks/native/libs/binder/IServiceManager.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/IServiceManager.cpp)
```
virtual sp<IBinder> getService(const Stringname) const
    {
        // 检索指定服务是否存在
        sp<IBinder> svc = checkService(name);
        if (svc != NULL) return svc;

        // 注意这里是 5s
        const long timeout = uptimeMillis() + 5000;
        if (!gSystemBootCompleted) {
            char bootCompleted[PROPERTY_VALUE_MAX];
            property_get("sys.boot_completed", bootCompleted, "0");
            // 判断开机是否完成，设置休眠时间
            gSystemBootCompleted = strcmp(bootCompleted, "1") == 0 ? true : false;
        }
        const long sleepTime = gSystemBootCompleted ? 1000 : 100;

        while (uptimeMillis() < timeout) {
            usleep(1000*sleepTime);
            // 检索指定服务是否存在
            sp<IBinder> svc = checkService(name);
            if (svc != NULL) return svc;
        }
        return NULL;
    }
```
通过 BpServiceManager 来获取MediaPlayer服务：检索服务是否存在，当服务存在则返回相应的服务，当服务不存在则根据开机是否完成，休眠 1s/0.1s，然后再继续检索服务。
该循环超时时间是 5s，这估计跟 Android 的 Activity ANR 时间为 5s 相关。

## # 2.3 BpServiceManager.checkService

[/frameworks/native/libs/binder/IServiceManager.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/IServiceManager.cpp)
```
virtual sp<IBinder> checkService( const Stringname) const
    {
        Parcel data, reply;
        // 写入 RPC 头
        data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
        // 写入服务名
        data.writeStringame);
        remote()->transact(CHECK_SERVICE_TRANSACTION, data, &reply); // 【见小节2.4】
        return reply.readStrongBinder(); // 【见小节2.9】
    }
```
检索指定服务是否存在，其中 remote()为 BpBinder

## # 2.4 transact

[/frameworks/native/libs/binder/BpBinder.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/BpBinder.cpp)
```
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        // code = ADD_SERVICE_TRANSACTION【见小节2.5】
        status_t status = IPCThreadState::self()->transact(
            mHandle, code, data, reply, flags);
        if (status == DEAD_OBJECT) mAlive = 0;
        return status;
    }

    return DEAD_OBJECT;
}
```
Binder 代理类调用 transact()方法，最终还是由 IPCThreadState 调用 transact()方法

### # 2.4.1 IPCThreadState::self

[/frameworks/native/libs/binder/IPCThreadState.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/IPCThreadState.cpp)
```
IPCThreadState* IPCThreadState::self()
{
    if (gHaveTLS) {
restart:
        const pthread_key_t k = gTLS;
        IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
        if (st) return st;
        return new IPCThreadState; // 初始化 IPCThreadState【见小节2.4.2】
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

### # 2.4.2 IPCThreadState初始化

[/frameworks/native/libs/binder/IPCThreadState.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/IPCThreadState.cpp)
```
IPCThreadState::IPCThreadState()
    : mProcess(ProcessState::self()),
      mStrictModePolicy(0),
      mLastTransactionBinderFlags(0)
{
    pthread_setspecific(gTLS, this);
    clearCaller();
    mIn.setDataCapacity();
    mOut.setDataCapacity();
}
```
每个线程都有一个 IPCThreadState，每个 IPCThreadState 中 都有一个 mIn、一个 mOut。成员变量 mProcess 保存了 ProcessState 变量（每个进程只有一个）
- mIn用来接收来自Binder设备的数据，默认大小为字节
- mOut用来存储发往Binder设备的数据，默认大小为字节

## # 2.5 IPCThreadState::transact

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

    if ((flags & TF_ONE_WAY) == 0) { // flags = 0 进入该分支
        if (reply) {
            err = waitForResponse(reply); // 等待响应
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
    } else {
        // 不需要响应消息的binder则进入该分支
        err = waitForResponse(NULL, NULL);
    }
    return err;
}
```

## # 2.6 IPCThreadState.writeTransactionData

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

## # 2.7 IPCThreadState.waitForResponse

[/frameworks/native/libs/binder/IPCThreadState.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/IPCThreadState.cpp)
```
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;
    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break; // 【见小节2.8】
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

## # 2.8 IPCThreadState.talkWithDriver

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
        // 通过 ioctl()一直进行读写操作，与Binder Driver进行通信【见小节2.8.1】
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
binder_write_read结构体用来与Binder设备交换数据的结构, 通过ioctl与mDriverFD通信，是真正与Binder驱动进行数据读写交互的过程。 
先向service manager进程发送查询服务的请求(BR_TRANSACTION)，见[Binder系列3—启动ServiceManager]()。
当service manager进程收到该命令后，会执行do_find_service() 查询服务所对应的handle，然后再调用binder_send_reply()应答发起者，
发送BC_REPLY协议，然后调用binder_transaction()，再向服务请求者的Todo队列插入事务。

接下来，再看看binder_transaction过程

### # 2.8.1 binder_transaction

[/drivers/staging/android/binder.c](http://androidxref.com/kernel_3.18/xref/drivers/staging/android/binder.c)
```
static void binder_transaction(struct binder_proc *proc,
			       struct binder_thread *thread,
			       struct binder_transaction_data *tr, int reply)
{
	struct binder_transaction *t;
	struct binder_work *tcomplete;
    // ...
	struct binder_proc *target_proc; // 目标进程
	struct binder_thread *target_thread = NULL; // 目标线程
	struct binder_node *target_node = NULL; // 目标binder节点
	struct list_head *target_list; // 目标todo队列
	wait_queue_head_t *target_wait; // 目标等待队列
    // ...

    // 分配两个结构体内存
	t = kzalloc(sizeof(*t), GFP_KERNEL);
	tcomplete = kzalloc(sizeof(*tcomplete), GFP_KERNEL);

    // 从 target_proc 中分配buffer
	t->buffer = binder_alloc_buf(target_proc, tr->data_size,
		tr->offsets_size, !reply && (t->flags & TF_ONE_WAY));

	for (; offp < off_end; offp++) {
		switch (fp->type) {
		// ...
        case BINDER_TYPE_HANDLE:
        case BINDER_TYPE_WEAK_HANDLE: {
            struct binder_ref *ref = binder_get_ref(proc, fp->handle);
            // ...
            // 此时运行在servicemanager进程，所以ref->node是指向服务所在进程的binder实体
            // 而 target_proc 为请求服务所在的进程，所以并不相等
            if (ref->node->proc == target_proc) {
                if (fp->type == BINDER_TYPE_HANDLE)
        		    fp->type = BINDER_TYPE_BINDER;
                else
        	        fp->type = BINDER_TYPE_WEAK_BINDER;
        		fp->binder = ref->node->ptr;
        		fp->cookie = ref->node->cookie; // BBinder服务的地址
        		binder_inc_node(ref->node, fp->type == BINDER_TYPE_BINDER, 0, NULL);
        	} else {
                struct binder_ref *new_ref;
                // 请求服务所在进程并非服务所在进程，则为请求服务所在进程创建binder_ref
               	new_ref = binder_get_ref_for_node(target_proc, ref->node);
        		fp->handle = new_ref->desc; // 重新赋值 handle
        		binder_inc_ref(new_ref, fp->type == BINDER_TYPE_HANDLE, NULL);
        	}
        } break;
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
这个过程非常重要，分两种情况：
1. 当请求服务的进程与服务属于不同进程，则为请求服务所在进程创建 binder_ref 对象，指向服务进程中的 binder_node
2. 当请求服务的进程与服务属于同一进程，则不再创建新对象，只是引用计数+1，并且修改 type 为 BINDER_TYPE_BINDER 或 BINDER_TYPE_WEAK_BINDER

### # 2.8.2 binder_thread_read

[/drivers/staging/android/binder.c](http://androidxref.com/kernel_3.18/xref/drivers/staging/android/binder.c)
```
static int binder_thread_read(struct binder_proc *proc,
			      struct binder_thread *thread,
			      binder_uintptr_t binder_buffer, size_t size,
			      binder_size_t *consumed, int non_block)
{
    // ...
    // 当线程todo队列有数据则执行往下运行; 当线程todo队列没有数据则进入休眠等待状态
    ret = wait_event_freezable(thread->wait, binder_has_thread_work(thread));
    // ...
    while (1) {
	    uint32_t cmd;
		struct binder_transaction_data tr;
		struct binder_work *w;
		struct binder_transaction *t = NULL;
        // 先从线程todo队列获取事务数据
		if (!list_empty(&thread->todo)) {
			w = list_first_entry(&thread->todo, struct binder_work,entry);
        // 线程todo队列没有数据，则从进程todo队列获取事务数据
		} else if (!list_empty(&proc->todo) && wait_for_proc_work) {
			w = list_first_entry(&proc->todo, struct binder_work,entry);
		}

		switch (w->type) {
		    case BINDER_WORK_TRANSACTION: {
                // 获取 transaction 数据
			    t = container_of(w, struct binder_transaction, work);
		    } break;
		}

        // 只有 BINDER_WORK_TRANSACTION 命令才能继续往下执行
		if (!t) continue;

		if (t->buffer->target_node) {
            // ...
		} else {
			tr.target.ptr = 0;
			tr.cookie = 0;
			cmd = BR_REPLY; // 设置命令为 BR_REPLY
		}
		tr.code = t->code;
		tr.flags = t->flags;
		tr.sender_euid = from_kuid(current_user_ns(), t->sender_euid);

		if (t->from) {
			struct task_struct *sender = t->from->proc->tsk;
            // 当非 TF_ONE_WAY 的情况下，将调用者进程的pid保存到sender_pid
			tr.sender_pid = task_tgid_nr_ns(sender,
							task_active_pid_ns(current));
		} else {
			tr.sender_pid = 0;
		}

		tr.data_size = t->buffer->data_size;
		tr.offsets_size = t->buffer->offsets_size;
		tr.data.ptr.buffer = (binder_uintptr_t)(
					(uintptr_t)t->buffer->data +
					proc->user_buffer_offset);
		tr.data.ptr.offsets = tr.data.ptr.buffer +
					ALIGN(t->buffer->data_size,
					    sizeof(void *));

        // 将 cmd 和数据写回用户空间
		if (put_user(cmd, (uint32_t __user *)ptr))
			return -EFAULT;
		ptr += sizeof(uint32_t);
		if (copy_to_user(ptr, &tr, sizeof(tr)))
			return -EFAULT;
		ptr += sizeof(tr);

		list_del(&t->work.entry);
		t->buffer->allow_user_free = 1;
		if (cmd == BR_TRANSACTION && !(t->flags & TF_ONE_WAY)) {
			// ...
		} else {
			t->buffer->transaction = NULL;
			kfree(t); // 通信完成则运行释放
		}
		break;
	}
done:
	*consumed = ptr - buffer;
	if (proc->requested_threads + proc->ready_threads == 0 &&
	    proc->requested_threads_started < proc->max_threads &&
	    (thread->looper & (BINDER_LOOPER_STATE_REGISTERED |
	     BINDER_LOOPER_STATE_ENTERED)) /* the user-space code fails to */
		proc->requested_threads++;
        // 生成 BR_SPAWN_LOOPER 命令，用于创建新的线程
		if (put_user(BR_SPAWN_LOOPER, (uint32_t __user *)buffer))
			return -EFAULT;
		binder_stat_br(proc, thread, BR_SPAWN_LOOPER);
	}
	return 0;
}
```

## # 2.9 readStrongBinder

[/frameworks/native/libs/binder/Parcel.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/Parcel.cpp#2140)
```
sp<IBinder> Parcel::readStrongBinder() const
{
    sp<IBinder> val;
    unflatten_binder(ProcessState::self(), *this, val);
    return val;
}
```
readStrongBinder 的功能是 flat_binder_object 解析并创建 BpBinder 对象

### # 2.9.1 unflatten_binder

[/frameworks/native/libs/binder/Parcel.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/Parcel.cpp)
```
status_t unflatten_binder(const sp<ProcessState>& proc,
    const Parcel& in, sp<IBinder>* out)
{
    const flat_binder_object* flat = in.readObject(false);
    if (flat) {
        switch (flat->hdr.type) {
            case BINDER_TYPE_BINDER:
                // 当请求服务的进程与服务属于同一进程
                *out = reinterpret_cast<IBinder*>(flat->cookie);
                return finish_unflatten_binder(NULL, *flat, in);
            case BINDER_TYPE_HANDLE:
                // 请求服务的进程与服务属于不同进程【见小节2.9.2】
                *out = proc->getStrongProxyForHandle(flat->handle);
                // 创建 BpBinder 对象
                return finish_unflatten_binder(
                    static_cast<BpBinder*>(out->get()), *flat, in);
        }
    }
    return BAD_TYPE;
}
```

### # 2.9.2 getStrongProxyForHandle

[/frameworks/native/libs/binder/ProcessState.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/ProcessState.cpp#)
```
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;

    AutoMutex _l(mLock);
    // 查找handle对应的资源项【见小节2.9.3】
    handle_entry* e = lookupHandleLocked(handle);

    if (e != NULL) {
        IBinder* b = e->binder;
        if (b == NULL || !e->refs->attemptIncWeak(this)) {
            // ...
            // 当 handle 值所对应的 IBinder 不存在或弱引用无效时，则创建 BpBinder 对象
            b = BpBinder::create(handle);
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            result.force_set(b);
            e->refs->decWeak(this);
        }
    }

    return result;
}
```


### # 2.9.3 lookupHandleLocked

[/frameworks/native/libs/binder/ProcessState.cpp](http://androidxref.com/9.0.0_r3/xref/frameworks/native/libs/binder/ProcessState.cpp#)
```
ProcessState::handle_entry* ProcessState::lookupHandleLocked(int32_t handle)
{
    const size_t N=mHandleToObject.size();
    // 当 handle 大于 mHandleToObject 的长度时，进入
    if (N <= (size_t)handle) {
        handle_entry e;
        e.binder = NULL;
        e.refs = NULL;
        // 从 mHandleToObject 的第N个位置开始，插入（handle+1-1）个e到队列中
        status_t err = mHandleToObject.insertAt(e, N, handle+1-N);
        if (err < NO_ERROR) return NULL;
    }
    return &mHandleToObject.editItemAt(handle);
}
```
根据handle值来查找对应的handle_entry

# # 3. 总结

请求服务(getService)的过程, 就是向 ServiceManager 进程查询指定服务，当执行 binder_transaction()时，会区分请求服务所在进程情况：
1. 请求服务的进程与服务属于不同进程时，则为请求服务所在进程创建 binder_ref 对象，指向服务进程中的 binder_node
    - 最终调用readStrongBinder(), 返回的是 BpBinder 对象
2. 请求服务的进程与服务属于同一进程时，则不再创建新对象，只是引用计数+1，并且修改 type 为 BINDER_TYPE_BINDER 或 BINDER_TYPE_WEAK_BINDER
    - 最终调用readStrongBinder(), 返回的是 IBinder 对象

# # 4. 参考

http://gityuan.com/2015/11/15/binder-get-service/




