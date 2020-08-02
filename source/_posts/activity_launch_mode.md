---
title: Activity启动模式
date: 2020-06-28 16:50:13
tags:
    - Activity
categories: Android
---

### LaunchMode

#### Standard

标准模式 or 默认模式。每次启动一个 activity 都会重新创建一个新的实例，不管这个实例是否已经存在。

> 当使用 ApplicationContext 启动 Standard 模式的 Activity 的时候会报错，
> 这是因为 Standard 模式 的 activity 默认会进入启动它的 activity 所在的任务栈中，但是非 activity 类型的 Context 并没有任务栈。
> 解决方法是 为待启动的 activity 指定标记为 FLAG_ACTIVITY_NEW_TASK，这样启动的时候就会创建一个新的任务栈，实际上是以 SingleTask 模式启动的。

#### SingleTop

栈顶复用模式。如果新 activity 已经位于任务栈的顶部，那么此 activity 不会被重新创建，同时回调 onNewIntent 方法。
如果新 activity 的实例已经存在但不是在栈顶，那么新的 activity 仍然会重新创建。

#### SingleTask

栈内复用模式。只要 Activity 在一个栈内存在，那么多次启动该 Activity 都不会重新创建实例，同时回调 onNewIntent 方法。
> 当一个具有 SingleTask 模式的 Activity 请求启动后，比如 Activity A, 系统首先会寻找是否存在 A `想要的任务栈`
> 如果不存在，就重新创建一个任务栈，然后创建 A 的实例后放入栈中。
> 如果存在 A `所需的任务栈`，就查看栈中是否存在 A 的实例，
> 如果不存在，就创建 A 的实例后放入栈中，
> 如果存在，就会弹出 A 上面的activity，把 A 调到栈顶(具有 clearTop 的效果)，并回调 onNewIntent 方法。

p.s. 这里有提到所需的任务栈，其实就是参数 TaskAffinity，标示了一个 Activity 所需要的任务栈的名字。
默认情况下，所有 Activity 所需的任务栈的名字是应用的包名。自定义的话，属性值是字符串，且中间必须含有包名分隔符 `.`

一般情况下 TaskAffinity 结合 singleTask 启动模式 或者 allowTaskReparenting 属性一起使用。

+ TaskAffinity and singleTask ：待启动的 Activity 会运行在名字和 TaskAffinity 相同的任务栈中。
+ TaskAffinity and allowTaskReparenting ：跨应用启动 Activity 时，allowTaskReparenting 如果为 true，待启动的 Activity 会转移到该应用的任务栈中。

任务栈分为前台任务栈和后台任务栈，后台任务栈中的 Activity 处于暂停状态，可以通过启动后台任务栈中的 Activity，将后台任务栈再次调到前台。

#### SingleInstance

单实例模式。具有此种模式的 Activity 只能单独位于一个任务栈中，当被复用时，同样会回调 onNewIntent 方法。

### Intent Flag

通过代码来设置 Activity 的启动模式的方式，优先级比清单文件设置更高

+ FLAG_ACTIVITY_NEW_TASK  
等同 SingleTask

+ FLAG_ACTIVITY_SINGLE_TOP  
等同 SingleTop

+ FLAG_ACTIVITY_CLEAR_TOP  
具有此标记的 Activity 启动时，在同一个任务栈内所有位于它上面的 Activity 都要出栈。

+ FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS  
具有此标记的 Activity 不会出现在历史 Activity 的列表中。等同 XML 中指定 Activity 的属性 `android:excludeFromRecents="true""`

### 总结

以上就是给 Activity 指定启动模式的两种方式了。当两种同时存在时，优先级以 Intent Flag 为准。  

另外，LaunchMode 无法`直接`给 Activity 设置 FLAG_ACTIVITY_CLEAR_TOP 标示，而 Intent Flag 无法为 Activity 指定 SingleInstance 模式。

注意上面说的是 直接，其实 SingleTask 默认具有 clearTop 的效果。

### 参考

《Android开发艺术探索》
