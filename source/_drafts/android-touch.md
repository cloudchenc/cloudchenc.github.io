---
title: Android事件分发机制
tags:
    - 事件分发机制
    - 设计模式
categories: Android
---

dispatchTouchEvent，onInterceptTouchEvent，onTouchEvent

view, viewgroup, activity

ViewGroup.requestDisallowInterceptTouchEvent阻止父view调用onInterceptTouchEvent进行拦截，解决嵌套view的滑动冲突

view的dispatchPointerEvent方法 
--> decorview的dispatchTouchEvent方法 
--> activity的dispatchTouchEvent方法

decorview --> activity --> window --> activity

事件分发 activity --> viewgroup --> view
事件处理 view --> viewgroup --> activity

在 DOWN 事件中将 touch 事件分发给子 View —> 这一过程如果有子 View 捕获消费了 touch 事件，会对 mFirstTouchTarget 进行赋值； 
ViewGroup的mFirstTouchTarget变量，记录捕获消费 touch 事件的 View，是一个链表结构
最后一步，DOWN、MOVE、UP 事件都会根据 mFirstTouchTarget 是否为 null，决定是自己处理 touch 事件，还是再次分发给子 View。  
DOWN 事件是事件序列的起点；决定后续事件由谁来消费处理；  
CANCEL 事件的触发场景：当父视图先不拦截，然后在 MOVE 事件中重新拦截，此时子 View 会接收到一个 CANCEL 事件。  


Tips：（试试将某些问题放入公众号关注后发送消息查看，引流粉丝，或者进入个人博客查看）

如果一个事件序列的 ACTION_DOWN 事件被 ViewGroup 拦截，此时子 View 调用 requestDisallowInterceptTouchEvent 方法有没有用？   
没用，down事件已经被拦截，mFirstTouchTarget仍为null，所以后续事件无法被子view消息处理

ACTION_DOWN 事件被子 View 消费了，那 ViewGroup 能拦截剩下的事件吗？如果拦截了剩下事件，当前这个事件 ViewGroup 能消费吗？子 View 还会收到事件吗？
能，ViewGroup不能消费剩下的事件，View会收到CANCEL事件
  
当 View Disable 时，会消费事件吗？  
不会


参考资料：

http://gityuan.com/2015/09/19/android-touch/
https://juejin.im/post/5ec6966ae51d45788b599fa8

