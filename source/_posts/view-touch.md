---
title: View事件分发机制
date: 2020-06-24 21:18:54
tags:
    - 事件分发
    - 设计模式
categories: Android
---

### 传递规则

当一个事件产生后，它的传递过程是这样的：Activity -> Window -> View, 分发机制受以下三个方法影响

+ public boolean dispatchTouchEvent(MotionEvent ev)  
用来进行事件的分发。返回结果受当前 View 的 onTouchEvent 和 子 View 的 dispatchTouchEvent 方法的影响，表示是否消费当前事件。

+ public boolean onInterceptTouchEvent(MotionEvent ev)  
在 dispatchTouchEvent 方法内部调用，用来判断是否拦截某个事件。如果当前 View 拦截了某个事件，那么在同一个事件序列当中，此方法不会被再次调用，返回结果表示是否拦截当前事件。

+ public boolean onTouchEvent(MotionEvent ev)  
在 dispatchTouchEvent 方法内部调用，用来处理点击事件。返回结果表示是否消费当前事件，如果不消费，那么在同一个事件序列中，当前 View 无法再次接受到事件。

需要注意的是，当一个 View 需要处理事件时，如果设置了 OnTouchListener，那么 OnTouchListener 中的 onTouch 方法会被回调。
如果 onTouch 返回了 false 不消费该事件，当前 View 的 onTouchEvent 方法才会被调用；
如果 onTouch 返回了 true 消费该事件，当前 View 的 onTouchEvent 方法就不会被调用。

如果设置了 OnClickListener，那么在 onTouchEvent 方法中会调用 OnClickListener 中的 onClick 方法。

> View处理事件的优先级：OnTouchListener 的 onTouch 方法 > onTouchEvent > OnClickListener 的 onClick 方法

### 源码解析

#### Input 系统

当用户触摸屏幕或者按键操作，首次触发的是硬件驱动，驱动收到事件后，将该相应事件写入到输入设备节点，这便产生了最原生态的内核事件。接着，输入系统取出原生态的事件，经过层层封装后成为KeyEvent或者MotionEvent ；最后，交付给相应的目标窗口(Window)来消费该输入事件。

1. 当屏幕被触摸，Linux内核会将硬件产生的触摸事件包装为Event存到/dev/input/event[x]目录下。

2. Input系统—InputReader线程：loop起来让EventHub调用getEvent()不断的从/dev/input/文件夹下读取输入事件。然后转换成EventEntry事件加入到InputDispatcher的mInboundQueue。

3. Input系统—InputDispatcher线程：从mInboundQueue队列取出事件，转换成DispatchEntry事件加入到connection的outboundQueue队列。再然后开始处理分发事件 （比如分发到ViewRootImpl的WindowInputEventReceiver中），取出outbound队列，放入waitQueue.

4. Input系统—UI线程：创建socket pair，分别位于”InputDispatcher”线程和focused窗口所在进程的UI主线程，可相互通信。

这里只说大概，详情请看gityuan的这篇文章[Input系统—事件处理全过程](http://gityuan.com/2016/12/31/input-ipc/)，文章3.3.3小节讲的是input系统事件从Native层分发Framework层的InputEventReceiver.dispachInputEvent()。

#### Framework 层

```java
// InputEventReceiver.dispachInputEvent()
private void dispatchInputEvent(int seq, InputEvent event) {
    mSeqMap.put(event.getSequenceNumber(), seq);
    onInputEvent(event); 
}
```

#### Native 层

##### ViewRootImpl

###### ViewRootImpl#WindowInputEventReceiver#onInputEvent

```java
final class WindowInputEventReceiver extends InputEventReceiver {
        ...
        @Override
        public void onInputEvent(InputEvent event, int displayId) {
            enqueueInputEvent(event, this, 0, true);
        }
        ...
    }
```

###### ViewRootImpl#enqueueInputEvent

```java
void enqueueInputEvent(InputEvent event,
            InputEventReceiver receiver, int flags, boolean processImmediately) {
        ...
        if (processImmediately) {
            doProcessInputEvents();
        } else {
            // 通过 handler 发送 MSG_PROCESS_INPUT_EVENTS，最终还是执行 doProcessInputEvents()
            scheduleProcessInputEvents();
        }
    }
```

###### ViewRootImpl#doProcessInputEvents

```java
void doProcessInputEvents() {
        // Deliver all pending input events in the queue.
        while (mPendingInputEventHead != null) {
            QueuedInputEvent q = mPendingInputEventHead;
            mPendingInputEventHead = q.mNext;
            if (mPendingInputEventHead == null) {
                mPendingInputEventTail = null;
            }
            q.mNext = null;

            mPendingInputEventCount -= 1;
            Trace.traceCounter(Trace.TRACE_TAG_INPUT, mPendingInputEventQueueLengthCounterName,
                    mPendingInputEventCount);

            long eventTime = q.mEvent.getEventTimeNano();
            long oldestEventTime = eventTime;
            if (q.mEvent instanceof MotionEvent) {
                MotionEvent me = (MotionEvent)q.mEvent;
                if (me.getHistorySize() > 0) {
                    oldestEventTime = me.getHistoricalEventTimeNano(0);
                }
            }
            mChoreographer.mFrameInfo.updateInputEventTime(eventTime, oldestEventTime);

            deliverInputEvent(q);
        }

        // We are done processing all input events that we can process right now
        // so we can clear the pending flag immediately.
        if (mProcessInputEventsScheduled) {
            mProcessInputEventsScheduled = false;
            mHandler.removeMessages(MSG_PROCESS_INPUT_EVENTS);
        }
    }
```

###### ViewRootImpl#deliverInputEvent

```java
private void deliverInputEvent(QueuedInputEvent q) {
        Trace.asyncTraceBegin(Trace.TRACE_TAG_VIEW, "deliverInputEvent",
                q.mEvent.getSequenceNumber());
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onInputEvent(q.mEvent, 0);
        }

        InputStage stage;
        if (q.shouldSendToSynthesizer()) {
            stage = mSyntheticInputStage;
        } else {
            stage = q.shouldSkipIme() ? mFirstPostImeInputStage : mFirstInputStage;
        }

        if (q.mEvent instanceof KeyEvent) {
            mUnhandledKeyManager.preDispatch((KeyEvent) q.mEvent);
        }

        if (stage != null) {
            handleWindowFocusChanged();
            stage.deliver(q);
        } else {
            finishInputEvent(q);
        }
    }
```

###### ViewRootImpl#ViewPostImeInputStage#onProcess

###### ViewRootImpl#ViewPostImeInputStage#processPointerEvent

##### View#dispatchPointerEvent

```java
public final boolean dispatchPointerEvent(MotionEvent event) {
        if (event.isTouchEvent()) {
            return dispatchTouchEvent(event);
        } else {
            return dispatchGenericMotionEvent(event);
        }
    }
```

##### DecorView#dispatchTouchEvent

```java
    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        final Window.Callback cb = mWindow.getCallback();
        return cb != null && !mWindow.isDestroyed() && mFeatureId < 0
                ? cb.dispatchTouchEvent(ev) : super.dispatchTouchEvent(ev);
    }
```

##### Activity#dispatchTouchEvent

##### PhoneWindow#superDispatchTouchEvent

##### DecorView

##### 

ViewRootImpl 
-> DecorView 通过 Activity 实现的 Window.Callback 接口，传递给 Activity
-> Activity
-> PhoneWindow
-> DecorView
-> 子View

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


### 参考

《Android开发艺术探索》
https://wanandroid.com/wenda/show/12119
http://gityuan.com/2015/09/19/android-touch/
https://juejin.im/post/5ec6966ae51d45788b599fa8

