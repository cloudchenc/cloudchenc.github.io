---
title: Activity生命周期
date: 2020-06-28 15:43:36
tags:
    - Activity
categories: Android
---

### 正常情况下的生命周期

在有用户参与的情况下，Activity 生命周期的改变

#### onCreate()

表示 Activity 正在被创建。在 Activity 第一次被创建的时候调用，可以做一些初始化操作，比如加载布局，绑定事件，初始化Activity所需数据等。

#### onRestart()    ✳✳✳

表示 Activity 正在重新启动。`当当前 Activity 从不可见重新变为可见状态时，onRestart就会被调用。`  
这种情况一般是用户行为导致的，比如用户按 Home 键切换到桌面或者用户打开了一个新的 Activity，这时当前的 Activity 就会回调 onPause 和 onStop，接着用户又返回这个 Activity 时，就会回调 onRestart 方法。

#### onStart()

表示 Activity 正在启动。这时 Activity 从不可见变为可见，但是还没有出现在前台，无法和用户交互。

#### onResume()

表示 Activity 已经可见，而且出现在前台，可以和用户交互。此时的 Activity 一定位于返回栈的栈顶，并且处于运行状态。
注意和 onStart 的对比，都表示 Activity 可见，但是 onStart 的时候 Activity 在后台，而 onResume 的时候 Activity 在前台。

#### onPause()

表示 Activity 正在停止。正常情况下，onStop 接下来会被调用，特殊情况下，快速返回当前 Activity，onResume 会被调用。  
通常会在这个方法中将一些及其消耗 CPU 的资源释放掉（比如显示地图或者大规模图形），以及保存一些关键数据（比如用户输入的数据等等），但这个方法不能执行耗时的操作，不然会影响到新的栈顶 Activity 的 onResume()方法回调。

#### onStop()

表示 Activity 即将停止。可以做一些稍微重量级的资源回收，同样不能太耗时。
这个方法在活动完全不可见的时候调用。它和 onPause()方法的主要区别在于，如果启动的新活动是一个对话框式的活动，那么 onPause() 方法会得到执行，而 onStop()方法并不会执行。
> 另外注意Activity之间跳转时，onStop()不一定会被调用！！！比如 DialogActivity 或者 透明的Activity

#### onDestroy()

表示 Activity 即将被销毁。可以做一些回收工作和最终的资源释放。
这个方法在活动被销毁之前调用，之后活动的状态将变为销毁状态。

#### 常见生命周期调用

1. 当某个 Activity 第一次启动    
onCreate -> onStart -> onResume

2. 当跳转新的 Activity 或者 切换到桌面   
onPause -> onStop   
特殊情况，当新的 Activity 是 dialog 或者透明主题，当前 activity 不会回调 onStop

3. 当用户再次回到原 Activity    
onRestart -> onStart -> onResume

4. 当用户按 back 键回退     
onPause -> onStop -> onDestroy

5. 当 activity 被系统回收后重新打开，同 1.

### 异常情况下的生命周期

Activity 被系统回调或者由于当前设备 Configuration 横竖屏切换导致Activity被销毁重建

#### 系统配置改变导致 Activity 被杀死并重建

##### 默认 configChanges

默认情况下，当系统配置发生改变后，Activity 会被销毁，依次调用 onPause，onStop，onDestroy。
同时由于 Activity 是在异常情况下终止的，系统会调用 onSaveInstanceState 来保存当前 Activity 的状态。调用时机发生在 onStop 之前，另外可能在 onPause 前，也可能在 onPause 后。

当 Activity 被重新创建时，依次调用 onCreate，onStart，onResume。
同时系统会调用 onRestoreInstanceState，把 Activity 销毁时 onSaveInstanceState 方法保存的 Bundle 对象作为参数传递给 onRestoreInstanceState 和 onCreate 方法。
所以，可以在 onRestoreInstanceState 和 onCreate 方法判断 Activity 是否发生重建，如果被重建了，就通过 Bundle 中保存的数据来恢复状态。
当 onRestoreInstanceState 发生调用，其参数 Bundle 一定有值，但 onCreate 在正常启动时，其参数 Bundle 是 null，只有在异常时才是有值的。

> onSaveInstanceState 与 onRestoreInstanceState 均在异常情况下才会被执行，onSaveInstanceState 发生在 onStop 之前，onRestoreInstanceState 发生在 onStart 之后。  
> 当 Activity 被意外终止时，Activity 会调用 onSaveInstanceState 去保存数据，同时委托 Window 去保存数据，接着 Window 还会委托它上面的顶层容器 ViewGroup 去保存数据。  
> 顶层容器 ViewGroup 可能是 DecorView，最后顶层容器再通知子元素来保存数据，完成整个数据保存过程。  
> 这是一种委托思想，上层委托下层，父布局委托子元素。可以类比 View 绘制流程 以及 事件分发机制。   

##### 自定义 configChanges

1. orientation
屏幕方向发生了改变，如旋转了手机屏幕

2. keyboardHidden
键盘可访问性发生了改变，如用户调出了键盘

3. screenSize
屏幕尺寸发生了变化，如旋转了手机屏幕。API 小于 13 不会导致重启，否则会导致 Activity 重启，一般结合 orientation 一起使用

4. locale
设备的本地位置发生了改变，如切换了系统语言

5. smallestScreenSize
物理屏幕尺寸发生了变化，如切换了外部显示设备。API 小于 13 不会导致重启，否则会导致 Activity 重启。

> 当设置 orientation 和 screenSize 时，旋转屏幕 activity 不再重建，并且不会调用 onSaveInstanceState 和 onRestoreInstanceState 保存和恢复数据，但会调用 onConfigurationChanged 方法。

#### 系统内存不足导致 Activity 被杀死并重建

根据 Activity 优先级杀死目标 Activity，具体数据存储和恢复过程同情景 1.

+ 前台 Activity, 正在和用户交互，优先级最高
+ 可见 但 非前台 Activity，弹出 dialog 导致 activity 可见，但处于后台无法和用户直接交互
+ 后台 且 不可见 Activity, 对用户已经不可见，优先级最低

### UML For Lifecycle

![activity_lifecycle](https://tva3.sinaimg.cn/large/d7f9b0f4gy1ghba8tno1fj20ty0zcq6k.jpg)

### 面试知识点

1. onRestart()调用的时机     
Home 键切换桌面 或者 Activity 跳转后返回

2. onPause()为什么不能执行耗时操作？    
分析从 A activity 跳转到 B activity，A 的 onPause 与 B 的 onResume 调用顺序       
源码角度分析，在 Activity 的启动过程中，调用 ActivityStack 的 resumeTopActivityInnerLocked() 在新的 Activity 启动之前，先回调栈顶 Activity 的 onPause 方法。
最终会执行到 ActivityThread 的 handleLaunchActivity()方法，完成新 activity 的 onCreate，onStart，onResume 的调用过程。（具体细节在Activity启动流程文章中分析）

3. View绘制发生在哪一步？    
onResume() ps. 结合View绘制流程分析

4. onStart, onResume 和 onPause, onStop 的对比分析

5. Dialog弹出时，是否回调 onStop ？或者 DialogActivity or 透明的 Activity

6. onSaveInstanceState, onRestoreInstanceState 调用的时机

待补充

### 参考

《Android开发艺术探索》
