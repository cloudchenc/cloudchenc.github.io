---
title: View.post()方法分析
tags:
categories:
---

scheduleTraversals()

TraversalRunnable

doTraversal()

performTraversals()

performTraversals()方法在执行时是在TraversalRunnable中，所以需要等到方法执行结束，才会从主线程的消息队列中取出View#post()方法缓存的HandlerAction中的Runnable