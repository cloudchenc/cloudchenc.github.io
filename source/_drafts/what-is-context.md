---
title: what_is_context
tags:
categories:
---

装饰者模式

Context 是一个关于应用环境的抽象类，它的实现由安卓系统提供。用来访问一些应用内资源、类，也可以调用系统服务开启 Activity 、Service 、发送和接收广播等

四大组件里 Activity 和 Service 都是 Context , 应用的 Context 数就是 Activity 、Service、Application 的个数之和，顺便说一下 Application 如果是多进程应用就会有多个

应用里有 Activity 、Service、Application 这些 Context ，我们先说说它们的共同点，它们都是 ContextWrapper 的子类，而 ContextWrapper 的成员变量 mBase 可以用来存放系统实现的 ContextImpl，这样我们在调用如 Activity 的 Context 方法时，都是通过静态代理的方式最终调用到 ContextImpl 的方法。我们调用 ContextWrapper 的 getBaseContext 方法就能拿到 ContextImpl 的实例
再说它们的不同点，它们有各自不同的生命周期；在功能上，只有 Activity 显示界面，正因为如此，Activity 继承的是 ContextThemeWrapper 提供一些关于主题，界面显示的能力，间接继承了 ContextWrapper ； 而 Applicaiton 、Service 都是直接继承 ContextWrapper ，所以我们要记住一点，凡是跟 UI 有关的，都应该用 Activity 作为 Context 来处理，否则要么会报错，要么 UI 会使用系统默认的主题。



参考资料：

https://www.jianshu.com/p/492ec35ea552
https://juejin.im/post/5ec09a0a6fb9a0437f73bf4e
http://gityuan.com/2017/04/09/android_context/