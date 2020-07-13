---
title: Service
tags:
categories:
---

Service 在主线程中运行

#### 1. onCreate()

#### 2. 启动Service

##### 2.1 startService() --> stopSelf() 或 stopService() 对应生命周期 onStartCommand()

##### 2.2. bindService() --> unBindService() 对应生命周期 onBind() --> onUnbind()

#### 3. onDestroy()

IntentService开启子线程执行耗时操作

https://juejin.im/post/59ffb33951882540f362ef49