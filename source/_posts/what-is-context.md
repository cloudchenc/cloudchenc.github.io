---
title: What is Context
date: 2020-12-04 14:34:10
tags: 
    - Context
categories: Android
---

### 概述

What is Context?   
这是一个好问题，官方的解释是这样:
```
/**
 * Interface to global information about an application environment.  This is
 * an abstract class whose implementation is provided by
 * the Android system.  It
 * allows access to application-specific resources and classes, as well as
 * up-calls for application-level operations such as launching activities,
 * broadcasting and receiving intents, etc.
 */
```
大致意思是一个应用环境的抽象类，它的实现由安卓系统提供。用来访问一些应用内资源、类，也可以调用系统服务开启 Activity 、Service 、发送和接收广播等

那么我们来看看Context的实现类 （图片来自Gityuan大佬）
![context](https://tvax3.sinaimg.cn/large/d7f9b0f4gy1glasf972q8j20t60lnmzj.jpg)

根据类关系图，可以看到四大组件里 Activity 和 Service 都是 Context , 应用的 Context 数就是 Activity 、Service、Application 的个数之和，顺便说一下 Application 如果是多进程应用就会有多个

应用里有 Activity 、Service、Application 这些 Context ，我们先说说它们的共同点，它们都是 ContextWrapper 的子类，而 ContextWrapper 的成员变量 mBase 可以用来存放系统实现的 ContextImpl，这样我们在调用如 Activity 的 Context 方法时，都是通过静态代理的方式最终调用到 ContextImpl 的方法。我们调用 ContextWrapper 的 getBaseContext 方法就能拿到 ContextImpl 的实例
再说它们的不同点，它们有各自不同的生命周期；在功能上，只有 Activity 显示界面，正因为如此，Activity 继承的是 ContextThemeWrapper 提供一些关于主题，界面显示的能力，间接继承了 ContextWrapper ； 而 Applicaiton 、Service 都是直接继承 ContextWrapper ，所以我们要记住一点，凡是跟 UI 有关的，都应该用 Activity 作为 Context 来处理，否则要么会报错，要么 UI 会使用系统默认的主题。

接下来我们就来看看四大组件是如何初始化的，另外由于组件是依托于Application的，同时就研究下Application的初始化过程  

p.s. Android Environment: API-28

### 组件创建流程

#### Activity创建-->performLaunchActivity

```java
public final class ActivityThread {

    /**  Core implementation of activity launch. */
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
        // 1. 获取 LoadedApk 对象
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }
        // ...
        // 2. 创建 ContextImpl 对象
        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            // 3. 通过 ContextImpl 获取类加载器，传入 Instrumentation 创建 Activity 对象
            java.lang.ClassLoader cl = appContext.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
            if (r.state != null) {
                r.state.setClassLoader(cl);
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to instantiate activity " + component
                    + ": " + e.toString(), e);
            }
        }

        try {
            // 4. 创建 Application 对象
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);

            if (activity != null) {
                CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
                Configuration config = new Configuration(mCompatConfiguration);
                if (r.overrideConfig != null) {
                    config.updateFrom(r.overrideConfig);
                }
                if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                        + r.activityInfo.name + " with config " + config);
                Window window = null;
                if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                    window = r.mPendingRemoveWindow;
                    r.mPendingRemoveWindow = null;
                    r.mPendingRemoveWindowManager = null;
                }
                appContext.setOuterContext(activity);
                // 4. 将 ContextImpl 与 Activity 绑定
                activity.attach(appContext, this, getInstrumentation(), r.token,
                        r.ident, app, r.intent, r.activityInfo, title, r.parent,
                        r.embeddedID, r.lastNonConfigurationInstances, config,
                        r.referrer, r.voiceInteractor, window, r.configCallback);

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                checkAndBlockForNetworkAccess();
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                activity.mCalled = false;
                // 5. 执行 Activity 的 onCreate()方法回调
                if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }
                if (!activity.mCalled) {
                    throw new SuperNotCalledException(
                        "Activity " + r.intent.getComponent().toShortString() +
                        " did not call through to super.onCreate()");
                }
                r.activity = activity;
            }
            r.setState(ON_CREATE);

            mActivities.put(r.token, r);

        } catch (SuperNotCalledException e) {
            throw e;

        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to start activity " + component
                    + ": " + e.toString(), e);
            }
        }

        return activity;
    }
}
```
#### Activity创建流程

1. 获取 LoadedApk 对象
2. 创建 ContextImpl 对象
3. 通过 ContextImpl 获取类加载器，传入 Instrumentation 创建 Activity 对象
4. 将 ContextImpl 与 Activity 绑定
5. 执行 Activity 的 onCreate()方法回调

#### Service创建-->handleCreateService

```java
public final class ActivityThread{

    private void handleCreateService(CreateServiceData data) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();
        // 1. 获取 LoadedApk 对象
        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);
        Service service = null;
        try {
            // 2. 通过 LoadedApk 获取类加载器，创建 Service 对象
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            service = packageInfo.getAppFactory()
                    .instantiateService(cl, data.info.name, data.intent);
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to instantiate service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }

        try {
            if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);
            // 3. 创建 ContextImpl 对象
            ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
            context.setOuterContext(service);
            // 4. 创建 Application 对象
            Application app = packageInfo.makeApplication(false, mInstrumentation);
            // 5. 将 ContextImpl 与 Service 绑定
            service.attach(context, this, data.info.name, data.token, app,
                    ActivityManager.getService());
            // 6. 执行 Service 的 onCreate()方法回调
            service.onCreate();
            mServices.put(data.token, service);
            try {
                // 7. 通知 ActivityManagerService 当前 Service 异步执行
                ActivityManager.getService().serviceDoneExecuting(
                        data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(service, e)) {
                throw new RuntimeException(
                    "Unable to create service " + data.info.name
                    + ": " + e.toString(), e);
            }
        }
    }
}
```

#### Service创建流程

1. 获取 LoadedApk 对象
2. 通过 LoadedApk 获取类加载器，创建 Service 对象
3. 创建 ContextImpl 对象
4. 创建 Application 对象
5. 将 ContextImpl, Application 与 Service 绑定
6. 执行 Service 的 onCreate()方法回调
7. 通知 ActivityManagerService 当前 Service 异步执行

#### BroadcastReceiver创建-->handleReceiver

```java
public final class ActivityThread{
    private void handleReceiver(ReceiverData data) {
        // If we are getting ready to gc after going to the background, well
        // we are back active so skip it.
        unscheduleGcIdler();

        String component = data.intent.getComponent().getClassName();
        // 1. 获取 LoadedApk 对象
        LoadedApk packageInfo = getPackageInfoNoCheck(
                data.info.applicationInfo, data.compatInfo);

        IActivityManager mgr = ActivityManager.getService();

        Application app;
        BroadcastReceiver receiver;
        ContextImpl context;
        try {
            // 2. 创建 Application 对象
            app = packageInfo.makeApplication(false, mInstrumentation);
            // 3. 获取 ContextImpl 对象
            context = (ContextImpl) app.getBaseContext();
            if (data.info.splitName != null) {
                context = (ContextImpl) context.createContextForSplit(data.info.splitName);
            }
            java.lang.ClassLoader cl = context.getClassLoader();
            data.intent.setExtrasClassLoader(cl);
            data.intent.prepareToEnterProcess();
            data.setExtrasClassLoader(cl);
            // 4. 通过 ContextImpl 获取类加载器，创建 BroadcastReceiver 对象 
            receiver = packageInfo.getAppFactory()
                    .instantiateReceiver(cl, data.info.name, data.intent);
        } catch (Exception e) {
            if (DEBUG_BROADCAST) Slog.i(TAG,
                    "Finishing failed broadcast to " + data.intent.getComponent());
            data.sendFinished(mgr);
            throw new RuntimeException(
                "Unable to instantiate receiver " + component
                + ": " + e.toString(), e);
        }

        try {
            sCurrentBroadcastIntent.set(data.intent);
            receiver.setPendingResult(data);
            // 5. 执行 BroadcastReceiver 的 onReceive()方法回调
            receiver.onReceive(context.getReceiverRestrictedContext(),
                    data.intent);
        } catch (Exception e) {
            if (DEBUG_BROADCAST) Slog.i(TAG,
                    "Finishing failed broadcast to " + data.intent.getComponent());
            data.sendFinished(mgr);
            if (!mInstrumentation.onException(receiver, e)) {
                throw new RuntimeException(
                    "Unable to start receiver " + component
                    + ": " + e.toString(), e);
            }
        } finally {
            sCurrentBroadcastIntent.set(null);
        }

        if (receiver.getPendingResult() != null) {
            data.finish();
        }
    }
}
```
#### BroadcastReceiver创建流程

1. 获取 LoadedApk 对象
2. 创建 Application 对象
3. 获取 ContextImpl 对象
4. 通过 ContextImpl 获取类加载器，创建 BroadcastReceiver 对象
5. 执行 BroadcastReceiver 的 onReceive()方法回调

#### ContentProvider创建-->installProvider

```java
public final class ActivityThread {

    private ContentProviderHolder installProvider(Context context,
            ContentProviderHolder holder, ProviderInfo info,
            boolean noisy, boolean noReleaseNeeded, boolean stable) {
        ContentProvider localProvider = null;
        IContentProvider provider;
        if (holder == null || holder.provider == null) {
            Context c = null;
            ApplicationInfo ai = info.applicationInfo;
            // 1. 获取 Context 对象
            if (context.getPackageName().equals(ai.packageName)) {
                c = context;
            } else if (mInitialApplication != null &&
                    mInitialApplication.getPackageName().equals(ai.packageName)) {
                c = mInitialApplication;
            } else {
                try {
                    c = context.createPackageContext(ai.packageName,
                            Context.CONTEXT_INCLUDE_CODE);
                } catch (PackageManager.NameNotFoundException e) {
                    // Ignore
                }
            }

            try {
                // 2. 通过 Context 获取类加载器，创建 ContentProvider 对象
                final java.lang.ClassLoader cl = c.getClassLoader();
                LoadedApk packageInfo = peekPackageInfo(ai.packageName, true);
                if (packageInfo == null) {
                    // System startup case.
                    packageInfo = getSystemContext().mPackageInfo;
                }
                localProvider = packageInfo.getAppFactory()
                        .instantiateProvider(cl, info.name);
                provider = localProvider.getIContentProvider();
                // 3. 将 Context 与 ContentProvider 绑定
                localProvider.attachInfo(c, info);
            } catch (java.lang.Exception e) {
                if (!mInstrumentation.onException(null, e)) {
                    throw new RuntimeException(
                            "Unable to get provider " + info.name
                            + ": " + e.toString(), e);
                }
                return null;
            }
        } else {
            provider = holder.provider;
        }
        // ...
        return retHolder;
    }
}
```

#### ContentProvider创建流程

1. 获取 Context 对象
2. 通过 Context 获取类加载器，创建 ContentProvider 对象
3. 将 Context 与 ContentProvider 绑定

#### Application创建-->handleBindApplication
```java
public final class ActivityThread {

    private void handleBindApplication(AppBindData data) {
        // ...
        // 1. 获取 LoadedApk 对象
        data.info = getPackageInfoNoCheck(data.appInfo, data.compatInfo);
        // ...
        // 2. 创建 ContextImpl 对象
        final ContextImpl appContext = ContextImpl.createAppContext(this, data.info);
        updateLocaleListFromAppContext(appContext,
                mResourcesManager.getConfiguration().getLocales());
        // ...
        if (ii != null) {
            final LoadedApk pi = getPackageInfo(instrApp, data.compatInfo,
                    appContext.getClassLoader(), false, true, false);
            final ContextImpl instrContext = ContextImpl.createAppContext(this, pi);

            try {
                // 3. 通过 ContextImpl 获取类加载器，创建 Instrumentation 对象
                final ClassLoader cl = instrContext.getClassLoader();
                mInstrumentation = (Instrumentation)
                    cl.loadClass(data.instrumentationName.getClassName()).newInstance();
            } catch (Exception e) {
                throw new RuntimeException(
                    "Unable to instantiate instrumentation "
                    + data.instrumentationName + ": " + e.toString(), e);
            }

            final ComponentName component = new ComponentName(ii.packageName, ii.name);
            mInstrumentation.init(this, instrContext, appContext, component,
                    data.instrumentationWatcher, data.instrumentationUiAutomationConnection);
        } else {
            mInstrumentation = new Instrumentation();
            mInstrumentation.basicInit(this);
        }

        Application app;
        try {
            // If the app is being launched for full backup or restore, bring it up in
            // a restricted environment with the base application class.
            // 4. 创建 Application 对象
            app = data.info.makeApplication(data.restrictedBackupMode, null);

            // Propagate autofill compat state
            app.setAutofillCompatibilityEnabled(data.autofillCompatibilityEnabled);

            mInitialApplication = app;

            // don't bring up providers in restricted mode; they may depend on the
            // app's custom Application class
            if (!data.restrictedBackupMode) {
                if (!ArrayUtils.isEmpty(data.providers)) {
                    // 5. 初始化 ContentProvider
                    installContentProviders(app, data.providers);
                    // For process that contains content providers, we want to
                    // ensure that the JIT is enabled "at some point".
                    mH.sendEmptyMessageDelayed(H.ENABLE_JIT, 10*1000);
                }
            }

            // Do this after providers, since instrumentation tests generally start their
            // test thread at this point, and we don't want that racing.
            try {
                mInstrumentation.onCreate(data.instrumentationArgs);
            }
            catch (Exception e) {
                throw new RuntimeException(
                    "Exception thrown in onCreate() of "
                    + data.instrumentationName + ": " + e.toString(), e);
            }
            try {
                // 6. 执行 Application 的 onCreate()方法回调
                mInstrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                if (!mInstrumentation.onException(app, e)) {
                    throw new RuntimeException(
                      "Unable to create application " + app.getClass().getName()
                      + ": " + e.toString(), e);
                }
            }
        }
    }
}
```

#### Application创建流程

1. 获取 LoadedApk 对象
2. 创建 ContextImpl 对象
3. 通过 ContextImpl 获取类加载器，创建 Instrumentation 对象
4. 创建 Application 对象
5. 初始化 ContentProvider
6. 执行 Application 的 onCreate()方法回调

### 核心对象

根据以上创建流程，可以发现 LoadedApk、ContextImpl、Application 是核心创建对象

#### LoadedApk 创建

##### ActivityThread.getPackageInfo
```java
public final class ActivityThread {
    public final LoadedApk getPackageInfo(ApplicationInfo ai, CompatibilityInfo compatInfo,
            int flags) {
        boolean includeCode = (flags&Context.CONTEXT_INCLUDE_CODE) != 0;
        boolean securityViolation = includeCode && ai.uid != 0
                && ai.uid != Process.SYSTEM_UID && (mBoundApplication != null
                        ? !UserHandle.isSameApp(ai.uid, mBoundApplication.appInfo.uid)
                        : true);
        boolean registerPackage = includeCode && (flags&Context.CONTEXT_REGISTER_PACKAGE) != 0;
        if ((flags&(Context.CONTEXT_INCLUDE_CODE
                |Context.CONTEXT_IGNORE_SECURITY))
                == Context.CONTEXT_INCLUDE_CODE) {
            if (securityViolation) {
                String msg = "Requesting code from " + ai.packageName
                        + " (with uid " + ai.uid + ")";
                if (mBoundApplication != null) {
                    msg = msg + " to be run in process "
                        + mBoundApplication.processName + " (with uid "
                        + mBoundApplication.appInfo.uid + ")";
                }
                throw new SecurityException(msg);
            }
        }
        return getPackageInfo(ai, compatInfo, null, securityViolation, includeCode,
                registerPackage);
    }
}
```
当 securityViolation 是true时，抛出SecurityException

##### ActivityThread.getPackageInfo
```java
public final class ActivityThread {
    private LoadedApk getPackageInfo(ApplicationInfo aInfo, CompatibilityInfo compatInfo,
            ClassLoader baseLoader, boolean securityViolation, boolean includeCode,
            boolean registerPackage) {
        final boolean differentUser = (UserHandle.myUserId() != UserHandle.getUserId(aInfo.uid));
        synchronized (mResourcesManager) {
            WeakReference<LoadedApk> ref;
            if (differentUser) {
                // Caching not supported across users
                ref = null;
            } else if (includeCode) {
                // 查询包名对应的 LoadedApk 弱引用对象
                ref = mPackages.get(aInfo.packageName);
            } else {
                ref = mResourcePackages.get(aInfo.packageName);
            }

            LoadedApk packageInfo = ref != null ? ref.get() : null;
            if (packageInfo == null || (packageInfo.mResources != null
                    && !packageInfo.mResources.getAssets().isUpToDate())) {
                // 创建 LoadedApk 对象
                packageInfo =
                    new LoadedApk(this, aInfo, compatInfo, baseLoader,
                            securityViolation, includeCode &&
                            (aInfo.flags&ApplicationInfo.FLAG_HAS_CODE) != 0, registerPackage);

                if (mSystemThread && "android".equals(aInfo.packageName)) {
                    packageInfo.installSystemApplicationInfo(aInfo,
                            getSystemContext().mPackageInfo.getClassLoader());
                }

                if (differentUser) {
                    // Caching not supported across users
                } else if (includeCode) {
                    // 创建 LoadedApk 弱饮用对象并加入到 mPackages
                    mPackages.put(aInfo.packageName,
                            new WeakReference<LoadedApk>(packageInfo));
                } else {
                    mResourcePackages.put(aInfo.packageName,
                            new WeakReference<LoadedApk>(packageInfo));
                }
            }
            return packageInfo;
        }
    }
}
```
'final ArrayMap<String, WeakReference<LoadedApk>> mPackages = new ArrayMap<>();' 记录了每一个包名对应的 LoadedApk 对象的弱引用,
当 mPackages 中找不到对应的 LoadedApk 对象时，则创建该 LoadedApk 对象并加入到 mPackages 中

#### Application 创建

##### LoadedApk.makeApplication
```java
public final class LoadedApk {
    public Application makeApplication(boolean forceDefaultAppClass,
            Instrumentation instrumentation) {
        // 保证一个 LoadedApk 对象只创建一个对应的 Application 对象
        if (mApplication != null) {
            return mApplication;
        }

        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "makeApplication");

        Application app = null;

        String appClass = mApplicationInfo.className;
        if (forceDefaultAppClass || (appClass == null)) {
            // 默认类名 或 forceDefaultAppClass 为 true
            appClass = "android.app.Application";
        }

        try {
            java.lang.ClassLoader cl = getClassLoader();
            if (!mPackageName.equals("android")) {
                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER,
                        "initializeJavaContextClassLoader");
                initializeJavaContextClassLoader();
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            }
            // 根据 mActivityThread 创建对应 ContextImpl 对象
            ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
            // 创建 Application 并把 ContextImpl 对象赋值给 Application 的 mBase 变量
            // 并把当前 LoadedApk 对象赋值给 Application 的 mLoadedApk 变量
            app = mActivityThread.mInstrumentation.newApplication(
                    cl, appClass, appContext);
            // 把 application 对象赋值给 ContextImpl 的 mOuterContext 变量
            appContext.setOuterContext(app);
        } catch (Exception e) {
            if (!mActivityThread.mInstrumentation.onException(app, e)) {
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                throw new RuntimeException(
                    "Unable to instantiate application " + appClass
                    + ": " + e.toString(), e);
            }
        }
        mActivityThread.mAllApplications.add(app);
        mApplication = app;

        if (instrumentation != null) {
            try {
                instrumentation.callApplicationOnCreate(app);
            } catch (Exception e) {
                if (!instrumentation.onException(app, e)) {
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                    throw new RuntimeException(
                        "Unable to create application " + app.getClass().getName()
                        + ": " + e.toString(), e);
                }
            }
        }
        return app;
    }    
}
```
流程梳理：
1. 根据包名是否包含"android"判断调用initializeJavaContextClassLoader()方法
2. 根据 mActivityThread 创建对应 ContextImpl 对象
3. 创建 Application 并赋值 mBase 变量和 mLoadedApk 变量
4. 把刚创建的plication 对象保存到 ContextImpl 的 mOuterContext 变量

关于应用类名采用的是Apk中声明的应用类名,即Manifest.xml中定义的类名. 有两种特殊情况会强制 设置应用类名为”android.app.Application”:
1. 当forceDefaultAppClass=true, 目前只有system_server进程初始化包名为”android”的apk过程会调用
2. Apk没有自定义应用类名的情况

##### Instrumentation.newApplication
```java
public class Instrumentation {
    public Application newApplication(ClassLoader cl, String className, Context context)
            throws InstantiationException, IllegalAccessException, 
            ClassNotFoundException {
        Application app = getFactory(context.getPackageName())
                .instantiateApplication(cl, className);
        app.attach(context); // 绑定 application 与 context
        return app;
    }
}
```

##### AppComponentFactory.instantiateApplication
```java
public class AppComponentFactory {
    public @NonNull Application instantiateApplication(@NonNull ClassLoader cl,
            @NonNull String className)
            throws InstantiationException, IllegalAccessException, ClassNotFoundException {
        // 创建 Application
        return (Application) cl.loadClass(className).newInstance();
    }
}
```

#### ContextImpl 创建

根据以上创建流程，可以发现不同组件创建 ContextImpl 的方法不同

- Activity: ActivityThread.createBaseContextForActivity()
- Service/Application: ContextImpl.createAppContext()
- BroadcastReceiver: Application.getBaseContext()
- ContentProvider: Context.createPackageContext()

##### Activity-->ActivityThread.createBaseContextForActivity
```java
public final class ActivityThread {
    private ContextImpl createBaseContextForActivity(ActivityClientRecord r) {
        final int displayId;
        try {
            displayId = ActivityManager.getService().getActivityDisplayId(r.token);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }

        ContextImpl appContext = ContextImpl.createActivityContext(
                this, r.packageInfo, r.activityInfo, r.token, displayId, r.overrideConfig);
        // ...
        return appContext;
    }
}
```
```java
class ContextImpl extends Context {
    static ContextImpl createActivityContext(ActivityThread mainThread,
            LoadedApk packageInfo, ActivityInfo activityInfo, IBinder activityToken, int displayId,
            Configuration overrideConfiguration) {
        if (packageInfo == null) throw new IllegalArgumentException("packageInfo");

        String[] splitDirs = packageInfo.getSplitResDirs();
        ClassLoader classLoader = packageInfo.getClassLoader();

        if (packageInfo.getApplicationInfo().requestsIsolatedSplitLoading()) {
            Trace.traceBegin(Trace.TRACE_TAG_RESOURCES, "SplitDependencies");
            try {
                classLoader = packageInfo.getSplitClassLoader(activityInfo.splitName);
                splitDirs = packageInfo.getSplitPaths(activityInfo.splitName);
            } catch (NameNotFoundException e) {
                // Nothing above us can handle a NameNotFoundException, better crash.
                throw new RuntimeException(e);
            } finally {
                Trace.traceEnd(Trace.TRACE_TAG_RESOURCES);
            }
        }

        ContextImpl context = new ContextImpl(null, mainThread, packageInfo, activityInfo.splitName,
                activityToken, null, 0, classLoader);
        return context;
    }
}
```

##### Service/Application-->ContextImpl.createAppContext
```java
class ContextImpl extends Context{
    static ContextImpl createAppContext(ActivityThread mainThread, LoadedApk packageInfo) {
        if (packageInfo == null) throw new IllegalArgumentException("packageInfo");
        ContextImpl context = new ContextImpl(null, mainThread, packageInfo, null, null, null, 0,
                null);
        context.setResources(packageInfo.getResources());
        return context;
    }
}
```
##### BroadcastReceiver-->Application.getBaseContext
getBaseContext()返回的mBase对象，就是在Application在创建时赋值的ContextImpl对象
```java
public class ContextWrapper extends Context {
    public Context getBaseContext() {
        return mBase;
    }
}
```

##### ContentProvider-->Context.createPackageContext
其实是调用了mBase的createPackageContext()方法，而mBase就是ContextImpl对象
```java
class ContextImpl extends Context {
    @Override
    public Context createPackageContext(String packageName, int flags)
            throws NameNotFoundException {
        return createPackageContextAsUser(packageName, flags, mUser);
    }

    @Override
    public Context createPackageContextAsUser(String packageName, int flags, UserHandle user)
            throws NameNotFoundException {
        if (packageName.equals("system") || packageName.equals("android")) {
            // The system resources are loaded in every application, so we can safely copy
            // the context without reloading Resources.
            return new ContextImpl(this, mMainThread, mPackageInfo, null, mActivityToken, user,
                    flags, null);
        }

        LoadedApk pi = mMainThread.getPackageInfo(packageName, mResources.getCompatibilityInfo(),
                flags | CONTEXT_REGISTER_PACKAGE, user.getIdentifier());
        if (pi != null) {
            ContextImpl c = new ContextImpl(this, mMainThread, pi, null, mActivityToken, user,
                    flags, null);

            final int displayId = mDisplay != null
                    ? mDisplay.getDisplayId() : Display.DEFAULT_DISPLAY;

            c.setResources(createResources(mActivityToken, pi, null, displayId, null,
                    getDisplayAdjustments(displayId).getCompatibilityInfo()));
            if (c.mResources != null) {
                return c;
            }
        }

        // Should be a better exception.
        throw new PackageManager.NameNotFoundException(
                "Application package " + packageName + " not found");
    }
}
```

### Context attach
从以上创建过程，可以看到Context的身影无处不在，那么Context是如何与四大组件和Application绑定的呢？
#### Activity & Context
```java
public class Activity extends ContextThemeWrapper {
    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback) {
        attachBaseContext(context); // 调用父类方法赋值mBase变量

        mFragments.attachHost(null /*parent*/);

        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(this);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        if (info.uiOptions != 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
        mUiThread = Thread.currentThread();

        mMainThread = aThread;
        mInstrumentation = instr;
        mToken = token;
        mIdent = ident;
        mApplication = application;
        mIntent = intent;
        mReferrer = referrer;
        mComponent = intent.getComponent();
        mActivityInfo = info;
        mTitle = title;
        mParent = parent;
        mEmbeddedID = id;
    }
}
```
将创建的ContextImpl对象赋值给父亲类ContextWrapper的mBase变量

#### Service & Context
```java
public abstract class Service extends ContextWrapper {
    public final void attach(
            Context context,
            ActivityThread thread, String className, IBinder token,
            Application application, Object activityManager) {
        attachBaseContext(context); // 调用父类方法赋值mBase变量
        mThread = thread;           // NOTE:  unused - remove?
        mClassName = className;
        mToken = token;
        mApplication = application;
        mActivityManager = (IActivityManager)activityManager;
        mStartCompatibility = getApplicationInfo().targetSdkVersion
                < Build.VERSION_CODES.ECLAIR;
    }
}
```
将创建的ContextImpl对象赋值给父亲类ContextWrapper的mBase变量

#### BroadcastReceiver & Context
`receiver.onReceive(context.getReceiverRestrictedContext(),data.intent);`
是将getOuterContext()获得的ContextImpl对象封装到ReceiverRestrictedContext对象里面，
然后通过onReceive()方法传递过去
```java
class ContextImpl extends Context {
    final Context getReceiverRestrictedContext() {
        if (mReceiverRestrictedContext != null) {
            return mReceiverRestrictedContext;
        }
        return mReceiverRestrictedContext = new ReceiverRestrictedContext(getOuterContext());
    }
}
```

#### ContentProvider & Context
```java
public abstract class ContentProvider {
    public void attachInfo(Context context, ProviderInfo info) {
        attachInfo(context, info, false);
    }
    private void attachInfo(Context context, ProviderInfo info, boolean testing) {
        mNoPerms = testing;

        /*
         * Only allow it to be set once, so after the content service gives
         * this to us clients can't change it.
         */
        if (mContext == null) {
            // 将创建的ContextImpl对象赋值给ContentProvider的mContext变量
            mContext = context;
            if (context != null) {
                mTransport.mAppOpsManager = (AppOpsManager) context.getSystemService(
                        Context.APP_OPS_SERVICE);
            }
            mMyUid = Process.myUid();
            if (info != null) {
                setReadPermission(info.readPermission);
                setWritePermission(info.writePermission);
                setPathPermissions(info.pathPermissions);
                mExported = info.exported;
                mSingleUser = (info.flags & ProviderInfo.FLAG_SINGLE_USER) != 0;
                setAuthorities(info.authority);
            }
            // 执行onCreate()回调
            ContentProvider.this.onCreate();
        }
    }
}
```
1. 将创建的ContextImpl对象赋值给ContentProvider的mContext变量
2. 执行ContentProvider的onCreate()方法回调

#### Application & Context
绑定过程发生在创建Application时，在Instrumentation类newApplication()方法中调用了attach()
```java
public class Application extends ContextWrapper {
    final void attach(Context context) {
        attachBaseContext(context); // 调用父类方法赋值mBase变量
        mLoadedApk = ContextImpl.getImpl(context).mPackageInfo;
    }
}
```
1. 将创建的ContextImpl对象赋值给父类ContextWrapper的mBase变量
2. 将当前的LoadedApk对象赋值给Application的mLoadedApk变量

#### Context 核心方法

1. Activity.getApplication() 返回Application 即当前Activity所属的Application
2. Service.getApplication() 返回Application 即当前Service所属的Application
3. ContextWrapper.getBaseContext() 返回ContextImpl 即mBase变量
4. ContextWrapper.getApplicationContext() 返回Application
5. ContextImpl.getApplicationContext() 返回Application
6. ContextImpl.getOuterContext() 返回ContextImpl 即mOuterContext变量
7. ContextImpl.getApplicationInfo() 返回ApplicationInfo 即mPackageInfo.mApplicationInfo

##### getApplication
- Activity 的 mApplication 变量是在 ActivityThread 类 performLaunchActivity()方法中调用 makeApplication()创建的，赋值是在 Activity 调用 attach()方法
- Service 的 mApplication 变量是在 ActivityThread 类 handleCreateService()方法中调用 makeApplication()创建的，赋值是在 Service 调用 attach() 方法
- BroadcastReceiver 虽然没有直接保存 mApplication 变量，但可以通过 onReceive()方法中的ContextImpl对象指向当前所在的Application
- ContentProvider 无法获取Application，因为Provider创建时并没有初始化application，所以其所在application不一定初始化

##### getApplicationContext
`ContextWrapper.getApplicationContext()`是通过mBase变量，也就是ContextImpl对象去获取applicationContext的
```java
class ContextImpl extends Context {
    @Override
    public Context getApplicationContext() {
        return (mPackageInfo != null) ?
                mPackageInfo.getApplication() : mMainThread.getApplication();
    }
}
```
mPackageInfo就是LoadedApk对象
```java
public final class LoadedApk {
    Application getApplication() {
        return mApplication;
    }
}
```
mMainThread就是ActivityThread对象
```java
public final class ActivityThread {
    public Application getApplication() {
        return mInitialApplication;
    }
}
```
- mPackageInfo.getApplication(): 返回的是LoadedApk.mApplication, 该变量初始化是在makeApplication()方法中，对于同一个apk只会执行一次
- mMainThread.getApplication(): 返回的是ActivityThread.mInitialApplication, 该变量赋值方法有两个：
-- ActivityThread.handleBindApplication()赋值;
-- system_server进程的ActivityThread.attach()赋值;

##### getOuterContext
ContextImpl的mOuterContext对象默认是在ContextImpl创建时初始化的，另外也可以通过setOuterContext()方法进行赋值。

- performLaunchActivity的createBaseContextForActivity过程, mOuterContext指向Activity;
- handleCreateService()过程, mOuterContext指向Service;
- BroadcastReceiver/Provider则采用默认值ContextImpl;
- makeApplication过程, mOuterContext指向Application;

### 总结

#### 组件创建
![component-initialization](https://tva1.sinaimg.cn/large/d7f9b0f4gy1glbxt9886qj20jp06kgm7.jpg)

每个Apk都对应唯一的Application对象和LoadedApk对象, 当Apk中任意组件的创建过程中, 当其所对应的的LoadedApk和Application没有初始化则会创建, 且只会创建一次.

另外大家会注意到唯有Provider在初始化过程并不会去创建所相应的Application对象.也就意味着当有多个Apk运行在同一个进程的情况下, 
第二个Apk通过Provider初始化过程再调用getContext().getApplicationContext()返回的并非Application对象, 而是NULL. 这里要注意会抛出空指针异常.

#### Context attach
Application:
- 调用attachBaseContext()将新创建ContextImpl赋值到父类ContextWrapper.mBase变量;
- 可通过getBaseContext()获取该ContextImpl;

Activity/Service:
- 调用attachBaseContext() 将新创建ContextImpl赋值到父类ContextWrapper.mBase变量;
- 可通过getBaseContext()获取该ContextImpl;
- 可通过getApplication()获取其所在的Application对象;

ContentProvider:
- 调用attachInfo()将新创建ContextImpl保存到ContentProvider.mContext变量;
- 可通过getContext()获取该ContextImpl;

BroadcastReceiver:
- 在onCreate过程通过参数将ReceiverRestrictedContext传递过去的.

ContextImpl:
- 可通过getApplicationContext()获取Application;

#### getApplicationContext
绝大多数情况下, getApplication()和getApplicationContext()这两个方法完全一致, 返回值也相同;
 那么两者到底有什么区别呢? 真正理解这个问题的人非常少. 接下来彻底地回答下这个问题:

getApplicationContext()这个的存在是Android历史原因. 我们都知道getApplication()只存在于Activity和Service对象;
那么对于BroadcastReceiver和ContentProvider却无法获取Application, 这时就需要一个能在Context上下文直接使用的方法, 那便是getApplicationContext().

两者对比:

对于Activity/Service来说, getApplication()和getApplicationContext()的返回值完全相同; 除非厂商修改过接口;
BroadcastReceiver在onReceive的过程, 能使用getBaseContext().getApplicationContext获取所在Application, 而无法使用getApplication;
ContentProvider能使用getContext().getApplicationContext()获取所在Application. 绝大多数情况下没有问题, 但是有可能会出现空指针的问题, 情况如下:
当同一个进程有多个apk的情况下, 对于第二个apk是由provider方式拉起的, 前面介绍过provider创建过程并不会初始化所在application,
此时执行 getContext().getApplicationContext()返回的结果便是NULL. 所以对于这种情况要做好判空.

Tips: 如果对于Application理解不够深刻, 建议getApplicationContext()方法谨慎使用, 做好是否为空的判定,防止出现空指针异常.

### 参考资料

http://gityuan.com/2017/04/09/android_context/  
https://www.yuque.com/beesx/beesandroid/sfzs75#d696d2a6  
