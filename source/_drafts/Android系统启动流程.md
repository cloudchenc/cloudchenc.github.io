---
title: Android系统启动流程
tags: 
categories: Android
---

### 启动流程

#### Loader层

1. 当手机处于关机状态时，长按电源键开机，引导芯片开始从预设在Boot ROM里的代码开始执行，然后加载引导程序Boot Loader到RAM
2. Boot Loader被加载到RAM后开始执行，该程序主要完成检查RAM，初始化硬件参数等

#### Kernel层

3. 引导程序之后进入Android内核层，先启动swapper进程（idle进程），该进程用来初始化进程管理、内存管理、加载Display、Camera Driver、Binder Driver等
4. swapper进程之后在启动kthreadd进程，该进程会创建内核工作线程kworkder、软中断线程ksoftirqd、thernal等内核守护进程，kthreadd进程是所有内核进程的父进程

#### Native层

5. 接着会启动init进程，init进程是所有用户进程的父进程，它会接着孵化出ueventd、logd、healthd、installd、adbd、lmdk等用户守护进程，启动ServiceManager来管理系统服务，启动Bootnaim开机动画
6. init进程通过解析init.rc文件fork生成Zygote进程，该进程是Android系统的第一个Java进程，它是所有Java进程的父进程，该进程主要完成了加载ZygoteInit类，注册Zygote Socket服务套接字；加载虚拟机；预加载Class；预加载Resources

#### Framework层

7. init进程接着fork生成Media Server进程，该进程负责启动和管理整个C++ Framework（包含AudioFlinger、Camera Service等服务）
8. Zygote进程接着会fork生成System Server进程，该进程负责启动和管理整个Java Framework（包含ActivityManagerService、WindowManagerService等服务）

#### App层

9. Zygote进程孵化出的第一个应用进程是Launcher进程（桌面），它还会孵化出Browser进程（浏览器）、Phone进程（电话）等，创建的每个应用都是一个单独的进程

### 参考资料：

https://www.yuque.com/beesx/beesandroid/vhwgnp
https://www.jianshu.com/p/1dee610cf074  
https://juejin.im/post/5ed1238951882543023c0281
https://juejin.im/post/5ec09cc16fb9a0437f73bf50