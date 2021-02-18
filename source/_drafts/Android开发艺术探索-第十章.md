---
title: Android开发艺术探索-第十章
tags:
categories:
---

1. Messagequeue的数据结构是单链表而不是队列？

2. ViewRootImpl中的checkThread方法限制了只能在主线程中访问UI，那么子线程中更新UI怎么操作？

3. threadLocal的使用场景一是当数据以线程为作用域且不同线程有不同的数据副本时，二是复杂逻辑下的对象传递，前提是不涉及切换线程，比如监听器传递

4. threadLocal的set和get的key是什么？Values？？

5. 单链表对比队列在插入和删除上有优势

6. 子线程在创建looper后，在处理完消息应该调用quit方法终止消息循环，否则会一直等待

7. 在looper调用loop方法时获取到新消息时，调用了handler的dispatchMessage，这样就切换到了指定的线程中执行

8. activityThread通过applicationThread和ams进行进程间通信，ams以进程间通信的方式完成activityThread的请求后会回调applicationThread的binder方法，然后applicationThread会向mH发送消息，mH收到消息后会将applicationThread切换到activityThread中去执行，也就是切换到了主线程执行

