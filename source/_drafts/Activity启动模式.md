---
title: Activity启动模式
tags:
categories:
---

### 启动模式

1. Standard



2. SingleTop：

当且仅当启动的 Activity 和上一个 Activity 一致的时候才会通过调用 onNewIntent() 方法重用 Activity

3. SingleTask：

只要栈中已经存在了该 Activity 的实例，就会直接调用 onNewIntent() 方法来实现重用实例。

4. SingleInstance:

一旦该模式的 Activity 实例已经存在于某个栈中，任何应用再激活该 Activity 时都会重用该栈中的实例，是的，依然是调用 onNewIntent() 方法。

### Intent Flag

通过代码来设置 Activity 的启动模式的方式，优先级比清单文件设置更高

FLAG_ACTIVITY_NEW_TASK

FLAG_ACTIVITY_CLEAR_TOP

FLAG_ACTIVITY_SINGLE_TOP