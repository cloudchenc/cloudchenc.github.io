---
title: View绘制流程
tags:
    - View绘制
    - Window
categories: Android
---

一个activity对应一个window，mWindow的赋值是在activity的attach()方法中，
创建phoneWindow对象赋值给mWindow，同时设置windowManager

activity的setCotentView()其实调用的是window的setContentView方法  

在phoneWindow的setContentView方法中，会初始化DecorView，及其内部的TitleView，和mContentParent
当设置FEATURE_NO_TITLE，TitleView不会被创建

setContentView方法设置的xml其实是mContentParent的一个子view

上面就是Activity启动的大致流程，紧接上面第7步，从ActivityThread开始执行到DecorView被添加到WindowManager中的过程如下（代码这里就省略了，有兴趣的自己可以搜索关键代码看一下），这里重点关注handleResumeActivity()方法

activityThread.handleLaunchActivity()   
—> activityThread.performLaunchActivity()  
—> activity.attach()   
—> activity.onCreate()  
—> activityThread.handleResumeActivity()   
—>activityThread.performResumeActivity()   
—> activity.onResume()   
—> windowManager.addView()  

```java
//只截取部分主要代码
final void handleResumeActivity(IBinder token, boolean clearHide, boolean isForward) { 
    ...
    // 这里会调用到activity.onResume()方法
    ActivityClientRecord r = performResumeActivity(token, clearHide); 

    if (r != null) {
        final Activity a = r.activity;

        ...
        if (r.window == null && !a.mFinished && willBeVisible) {
                // 获得Window对象
            r.window = r.activity.getWindow(); 
            // 获得Window中的DecorView对象
            View decor = r.window.getDecorView(); 
            decor.setVisibility(View.INVISIBLE);
            // 获得WindowManager对象
            //这个WindowManager就是在activity.attach()的时候和window一起创建的
            ViewManager wm = a.getWindowManager(); 
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            if (a.mVisibleFromClient) {
                a.mWindowAdded = true;
                // 调用windowManager.addView方法
                wm.addView(decor, l); 
            }
            ...
            if (!r.activity.mFinished && willBeVisible
                    && r.activity.mDecor != null && !r.hideForNow) {
                r.activity.mVisibleFromServer = true;
                mNumVisibleActivities++;
                if (r.activity.mVisibleFromClient) {
                        //设置decorView可见
                    r.activity.makeVisible();
                }
            }
        }
    }
}
```
通过调用windowManager.addView(decor, lp)，这样一来，DecorView和WindowManager就建立了联系。
WindowManager继承了ViewManger接口，但实际上其本身依然是一个接口，实现类是WindowManagerImpl

```java
//WindowManagerImpl implements WindowManager
//WindowManager extends ViewManager
public interface ViewManager
{
        public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}


//只截取部分主要代码
public final class WindowManagerImpl implements WindowManager {
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
    private final Context mContext;
    private final Window mParentWindow;

    public WindowManagerImpl(Context context) {
        this(context, null);
    }

    private WindowManagerImpl(Context context, Window parentWindow) {
        mContext = context;
        mParentWindow = parentWindow;
    }

    public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
        return new WindowManagerImpl(mContext, parentWindow);
    }

    public WindowManagerImpl createPresentationWindowManager(Context displayContext) {
        return new WindowManagerImpl(displayContext, mParentWindow);
    }

    //往Window中添加view
    @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }

    //更新布局
    @Override
    public void updateViewLayout(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.updateViewLayout(view, params);
    }

    //将删除View的消息发送到MessageQueue中，稍后删除
    @Override
    public void removeView(View view) {
        mGlobal.removeView(view, false);
    }

    //立刻删除Window中的view
    @Override
    public void removeViewImmediate(View view) {
        mGlobal.removeView(view, true);
    }
}
```

注意：同样实现了ViewManger接口的还有ViewGroup，我们知道ViewGroup也有addView方法，但是在ViewGroup中是将普通的view或者viewGroup作为Children加入，而在WindowManagerImpl是将DecorView作为根布局加入到PhoneWindow中去，所以两个方法的作用是截然不同的

我们发现，WindowManagerImpl并没有直接实现操作View的相关方法，而是全部交给WindowManagerGlobal。WindowManagerGlobal是一个单例类—即一个进程中最多仅有一个。创建WindowManagerGlobal对象的方式如下

public static WindowManagerGlobal getInstance() {
    synchronized (WindowManagerGlobal.class) {
        if (sDefaultWindowManager == null) {
            sDefaultWindowManager = new WindowManagerGlobal();
        }
        return sDefaultWindowManager;
    }
}

分析WindowManangerGlobal实际上就是分析其内部的三个方法

addView(View view, ViewGroup.LayoutParams params)

该方法的主要作用是将decorView添加到viewRootImp中，通过viewRootImp对其内部的view进行管理

```java
public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
            //参数检查
            if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
            ...
        //判断是否有父Window，从而调整当前窗口布局参数（layoutParams）
        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
        if (parentWindow != null) {
            //有，调整title和token
            parentWindow.adjustLayoutParamsForSubWindow(wparams);
        } else {
            //无，应用开启了硬件加速的话，decorview就开启
            final Context context = view.getContext();
            if (context != null
                    && (context.getApplicationInfo().flags
                            & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
                wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
            }
        }
        ViewRootImpl root;
        View panelParentView = null;
        synchronized (mLock) {
            if (mSystemPropertyUpdater == null) {
                //创建一个Runnable，用来遍历更新所有ViewRootImpl，更新系统参数
                mSystemPropertyUpdater = new Runnable() {
                    @Override public void run() {
                        synchronized (mLock) {
                            for (int i = mRoots.size() - 1; i >= 0; --i) {
                                mRoots.get(i).loadSystemProperties();
                            }
                        }
                    }
                };
                //添加到执行队列中
                SystemProperties.addChangeCallback(mSystemPropertyUpdater);
            }

            //看需要add的view是否已经在mViews中
            //WindowManager不允许同一个View被添加两次
            int index = findViewLocked(view, false);
            if (index >= 0) {
                // 如果被添加过，就看是否在死亡队列里也存在
                if (mDyingViews.contains(view)) {
                    // 如果存在，就将这个已经存在的view对应的window移除
                    mRoots.get(index).doDie();
                } else {
                    //否则，说明view已经被添加，不需要重新添加了
                    throw new IllegalStateException("View " + view
                            + " has already been added to the window manager.");
                }
            }
                        // 如果属于子Window层级
            if (wparams.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                    wparams.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
                final int count = mViews.size();
                //遍历viewRootImpl集合，看是否存在一个window的IBinder对象和需要添加的window的token一                                       //致，之后赋值引用
                for (int i = 0; i < count; i++) {
                    if (mRoots.get(i).mWindow.asBinder() == wparams.token) {
                        panelParentView = mViews.get(i);
                    }
                }
            }
                        //创建一个ViewRootImpl对象并保存在root变量中
            root = new ViewRootImpl(view.getContext(), display);
                        //给需要添加的view设置params，也就是decorView
            view.setLayoutParams(wparams);
                        //将decoView、布局参数以及新建的ViewRootImpl保存在三个集合中
            mViews.add(view);
            mRoots.add(root);
            mParams.add(wparams);
            try {
                //将decorView设置给ViewRootImpl
                //ViewRootImpl向WMS添加新的窗口、申请Surface以及decorView在Surface上的重绘动作
                //这才是真正意义上完成了窗口的添加操作
                root.setView(view, wparams, panelParentView);
            } catch (RuntimeException e) {
                // BadTokenException or InvalidDisplayException, clean up.
                if (index >= 0) {
                    removeViewLocked(index, true);
                }
                throw e;
            }
        }
    }
```
为新窗口创建一个ViewRootImpl对象。顾名思义，ViewRootImpl实现了一个控件树的根。它负责与WMS进行直接的通讯，负责管理Surface，负责触发控件的测量与布局，负责触发控件的绘制，同时也是输入事件的中转站。总之，ViewRootImpl是整个控件系统正常运转的动力所在，无疑是本章最关键的一个组件。


updateViewLayout(View view, ViewGroup.LayoutParams params)

该方法的主要作用是更新decorView的layoutParams，如layoutParams.width从100变为了200，则需要将这个变化通知给WMS使其调整Surface的大小，并让窗口进行重绘

```java
public void updateViewLayout(View view, ViewGroup.LayoutParams params) {
        //参数检查
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }
    if (!(params instanceof WindowManager.LayoutParams)) {
        throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
    }
    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;
    //将layoutparams保存到decorView中
    view.setLayoutParams(wparams);
    synchronized (mLock) {
        // 获取decorView在三个集合中的索引
        int index = findViewLocked(view, true);
        ViewRootImpl root = mRoots.get(index);
        mParams.remove(index);
        //更新layoutParams到集合中
        mParams.add(index, wparams);
        //调用viewRootImpl的setLayoutParams()使得新的layoutParams生效
        root.setLayoutParams(wparams, false);
    }
}
```

removeView(View view)

该方法的作用是从3个集合中删除此Window所对应的元素，包括decorView、layoutPrams以及viewRootImpl，并要求viewRootImpl从WMS中删除对应的Window，并释放一切需要回收的资源

```java
public void removeView(View view, boolean immediate) {
    if (view == null) {
        throw new IllegalArgumentException("view must not be null");
    }
  
    synchronized (mLock) {
        int index = findViewLocked(view, true);
        View curView = mRoots.get(index).getView();
        removeViewLocked(index, immediate);//其内部会调用root.die(immediate)
        if (curView == view) {
            return;
        }
      
        throw new IllegalStateException("Calling with view " + view
                + " but the ViewAncestor is attached to " + curView);
    }
}
```

ViewRootImpl实现了ViewParent接口，它是WindowManagerGlobal工作的实际实现者

```java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
     synchronized (this) {
         if (mView == null) {
            //mView保存了decorView
             mView = view;
            //mWindowAttributes保存了窗口所对应的LayoutParams
             mWindowAttributes.copyFrom(attrs);     
             attrs = mWindowAttributes;
             ...
                     //请求UI开始绘制
             requestLayout();   
             //初始化mInputChannel
             //InputChannel是窗口接受来自InputDispatcher的输入事件的管道
             //注意，仅当窗口的属性inputFeatures不含有INPUT_FEATURE_NO_INPUT_CHANNEL时
             //才会创建InputChannel，否则mInputChannel为空，从而导致此窗口无法接受任何输入事件
             mInputChannel = new InputChannel();   
             try {
                //通知WindowManagerService添加一个窗口，注册一个事件监听管道
                //用来监听 按键(KeyEvent)和触摸(MotionEvent)事件
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mInputChannel);
                }
                ...
        }
}
```

setView()内部的执行过程  
viewRootImpl.setView()   
—> viewRootImpl.requestLayout()   
—> viewRootIml;.scheduleTraversals()   
—> viewRootImpl.doTraversal()  
—> viewRootImpl.performTraversals()  
—>进入view的绘制流程  

```java
public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            //线程检查，这里就是判断更新UI的操作是否在主线程的方法
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
}

void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }

final class TraversalRunnable implements Runnable {
    @Override
    public void run() {
        doTraversal();
    }
}

void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

            if (mProfile) {
                Debug.startMethodTracing("ViewAncestor");
            }
            //开始view的绘制, 调用measure，layout，draw方法
            performTraversals();

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
}
```

onMeasure

MeasureSpec概括了父布局对子view的布局要求，包括测量模式和测量大小
int值是32位，前2位是模式，后30位是大小

UNSPECIFIED：对view大小不做限制
EXACTLY：固定的宽高或者match_parent
AT_MOST：大小不超过某值，wrap_content

自定义view：直接测量自己的大小，调用setMeasuredDimension进行设置
自定义viewgroup：遍历子view，测量子view大小，结合子view的尺寸再调用setMeasuredDimension设置自己的尺寸


onLayout

自定义view：无需重写，xml已定义
自定义viewgroup：递归调用子view的layout方法

view的layout方法  
调用layout方法，通过setFrame方法对view的left，right，top，bottom进行赋值，从而确定view的大小和位置


onDraw

自定义view：调用paint类在canvas上进行绘制
自定义viewgroup：无需重写，子view来重写


参考资料：

https://www.jianshu.com/p/c5df0ac39e01  
https://www.jianshu.com/p/3366e4bec7ce  