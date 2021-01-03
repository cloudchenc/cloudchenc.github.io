---
title: LeakCanary源码分析
tags:
categories:
---

在一个Activity执行完onDestroy()之后，将它放入WeakReference中，
然后将这个WeakReference类型的Activity对象与ReferenceQueque关联。

这时再从ReferenceQueque中查看是否有没有该对象，如果没有，执行gc，
再次查看，还是没有的话则判断发生内存泄露了。
最后用HAHA这个开源库去分析dump之后的heap内存。

