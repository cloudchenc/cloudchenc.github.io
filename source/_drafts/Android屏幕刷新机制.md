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

参考资料：

https://www.jianshu.com/p/0a54aa33ba7d