---
title: ActivityThread工作原理
tags:
    - 源码
categories: Android
---

ActivityThread，也就是主线程，但不一定是UI线程

因为UI线程的赋值是在activity的attach方法中

```java 
final void attach(Context context, ...) {
 // 这里的Thread.currentThread()就是 ViewRootImpl 创建时的线程
 mUiThread = Thread.currentThread();
}
```

Activity 创建好之后，应用的主线程会调用 ActivityThread.handleResumeActivity，
这个方法会通过WindowManager的addView方法把 Activity 的 DecorView 添加到 WindowManger 里，
最终调用的是WindowManagerGlobal的addView(), 就是在这个方法中创建的 ViewRootImpl
所以Activity 的 DecorView 对应的 ViewRootImpl 是在主线程创建的，此时的mUiThread是主线程


但当 WindowManager.addView 在子线程调用时，ViewRootImpl创建的线程就会是子线程，
此时的mUiThread就被赋值为了子线程，这个子线程就是该view的UI线程，也就能在子线程中刷新view了


参考资料：

https://juejin.im/post/5eb0ea635188256d5b4d987f

https://juejin.im/post/5ec79fdc6fb9a047a226894c?utm_source=gold_browser_extension#heading-2

https://juejin.im/post/5ec09cc16fb9a0437f73bf50