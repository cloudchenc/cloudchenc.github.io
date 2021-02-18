---
title: Android屏幕刷新机制
tags: 

---

绘制的三种方式：Canvas，OpenGL，Vulkan

CPU负责收集绘制指令

GPU负责绘制渲染

View requestLayout invalidate

为什么使用双缓冲技术？

在Linux上通常使用Framebuffer来做显示输出，当用户进程更新Framebuffer中的数据后，显示驱动会把FrameBuffer中每个像素点的值更新到屏幕，但是如果上一帧数据还没显示完，Framebuffer中的数据又更新了，就会带来残影的问题，用户会觉得有闪烁感，所以采用了双缓冲技术。

双缓冲意味着要使用两个缓冲区（在上文提及的SharedBufferStack中），其中一个称为Front Buffer，另一个称为Back Buffer。UI总是先在Back Buffer中绘制，然后再和Front Buffer交换，渲染到显示设备中。即只有当另一个buffer的数据准备好后，通过io_ctl来通知显示设备切换Buffer。

ViewRootImpl

Choreographer

参考资料：

https://www.jianshu.com/p/0a54aa33ba7d


当你的应用正在往Back Buffer中填充数据时，系统会将Back Buffer锁定。如果到了GPU交换两个Buffer的时间点，你的应用还在往Back Buffer中填充数据，GPU会发现Back Buffer被锁定了，它会放弃这次交换。

这样做的后果就是手机屏幕仍然显示原先的图像，这就是我们常常说的丢帧，所以为了避免丢帧的发生，我们就要尽量减少布局层级，减少不必要的View的invalidate调用，减少大量对象的创建（GC也会占用CPU时间）等等。


Android在每一帧中实际上只是在完成三个操作，分别是输入(Input)、动画(Animation)、绘制(Draw)。

choreographer

这个类就可以解决vsync和绘制不同步的问题，其实它的原理用一句话总结就是往Choreographer里发一个消息，最快也要等到下一个vsync信号来的时候才会开始处理消息。

ViewRootImp#requestLayout()

ViewRootImp#scheduleTraversals()

Choreographer#postCallbackDelayedInternal()

Choreographer#scheduleFrameLocked()

DisplayEventReceiver#nativeScheduleVsync()

nativeScheduleVsync()会向SurfaceFlinger注册Vsync信号的监听，VSync信号由SurfaceFlinger实现并定时发送，当Vsync信号来的时候就会回调FrameDisplayEventReceiver#onVsync()，这个方法给发送一个带时间戳Runnable消息，这个Runnable消息的run()实现就是FrameDisplayEventReceiver# run()， 接着就会执行doFrame()。

FrameDisplayEventReceiver#onVsync()

FrameDisplayEventReceiver#run()

Choreographer#doFrame()

Choreographer#doCallbacks()

TraversalRunnable

ViewRootImp#doTraversal()

ViewRootImp#performTraversals()

真正开始绘制


同步消息屏障

结合handler异步消息属性一起使用

ViewRootImpl#scheduleTraversals()添加屏障
mHandler.getLooper().getQueue().postSyncBarrier()

ViewRootImpl#doTraversal()移除屏障
mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

在Choreographer#scheduleFrameLocked()和FrameDisplayEventReceiver#onVsync()中提到，我们会给与Message有关的绘制请求设置成异步消息(msg.setAsynchronous(true))，为什么要这么做呢？

这时候MessageQueue#postSyncBarrier()就发挥它的作用了，简单来说，它的作用就是一个同步消息屏障，能够把我们的异步消息（也就是绘制消息）的优先级提到最高。

MessageQueue#postSyncBarrier()

主线程的 Looper 会一直循环调用 MessageQueue 的 next() 来取出队头的 Message 执行，当 Message 执行完后再去取下一个。

当 next() 方法在取 Message 时发现队头是一个同步屏障的消息时，就会去遍历整个队列，只寻找设置了异步标志的消息，如果有找到异步消息，那么就取出这个异步消息来执行，否则就让 next() 方法陷入阻塞状态。

如果 next() 方法陷入阻塞状态，那么主线程此时就是处于空闲状态的，也就是没在干任何事。
所以，如果队头是一个同步屏障的消息的话，那么在它后面的所有同步消息就都被拦截住了，直到这个同步屏障消息被移除，否则主线程就一直不会去处理同步屏障后面的同步消息

