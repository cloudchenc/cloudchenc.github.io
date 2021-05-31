---
title: PMS
date: 2021-05-31 22:00:45
tags:
categories: 
---

本文基于android-29源码分析

# PMS 

PackageManagerService是Android系统服务，在SystemServer中作为引导服务创建

## PMS创建

zygote --> SystemServer#main()

### SystemServer#run()

```java 
    // com.android.server.SystemServer#main
    /**
     * The main entry point from zygote.
     */
    public static void main(String[] args) {
        new SystemServer().run();
    }
```

```java 
    // com.android.server.SystemServer#run
    private void run() {
        try {
            traceBeginAndSlog("InitBeforeStartServices");

            // Record the process start information in sys props.
            SystemProperties.set(SYSPROP_START_COUNT, String.valueOf(mStartCount));
            SystemProperties.set(SYSPROP_START_ELAPSED, String.valueOf(mRuntimeStartElapsedTime));
            SystemProperties.set(SYSPROP_START_UPTIME, String.valueOf(mRuntimeStartUptime));

            EventLog.writeEvent(EventLogTags.SYSTEM_SERVER_START,
                    mStartCount, mRuntimeStartUptime, mRuntimeStartElapsedTime);

            // If a device's clock is before 1970 (before 0), a lot of
            // APIs crash dealing with negative numbers, notably
            // java.io.File#setLastModified, so instead we fake it and
            // hope that time from cell towers or NTP fixes it shortly.
            if (System.currentTimeMillis() < EARLIEST_SUPPORTED_TIME) {
                Slog.w(TAG, "System clock is before 1970; setting to 1970.");
                SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
            }

            //
            // Default the timezone property to GMT if not set.
            //
            String timezoneProperty = SystemProperties.get("persist.sys.timezone");
            if (timezoneProperty == null || timezoneProperty.isEmpty()) {
                Slog.w(TAG, "Timezone not set; setting to GMT.");
                SystemProperties.set("persist.sys.timezone", "GMT");
            }

            // If the system has "persist.sys.language" and friends set, replace them with
            // "persist.sys.locale". Note that the default locale at this point is calculated
            // using the "-Duser.locale" command line flag. That flag is usually populated by
            // AndroidRuntime using the same set of system properties, but only the system_server
            // and system apps are allowed to set them.
            //
            // NOTE: Most changes made here will need an equivalent change to
            // core/jni/AndroidRuntime.cpp
            if (!SystemProperties.get("persist.sys.language").isEmpty()) {
                final String languageTag = Locale.getDefault().toLanguageTag();

                SystemProperties.set("persist.sys.locale", languageTag);
                SystemProperties.set("persist.sys.language", "");
                SystemProperties.set("persist.sys.country", "");
                SystemProperties.set("persist.sys.localevar", "");
            }

            // The system server should never make non-oneway calls
            Binder.setWarnOnBlocking(true);
            // The system server should always load safe labels
            PackageItemInfo.forceSafeLabels();

            // Default to FULL within the system server.
            SQLiteGlobal.sDefaultSyncMode = SQLiteGlobal.SYNC_MODE_FULL;

            // Deactivate SQLiteCompatibilityWalFlags until settings provider is initialized
            SQLiteCompatibilityWalFlags.init(null);

            // Here we go!
            Slog.i(TAG, "Entered the Android system server!");
            int uptimeMillis = (int) SystemClock.elapsedRealtime();
            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_SYSTEM_RUN, uptimeMillis);
            if (!mRuntimeRestart) {
                MetricsLogger.histogram(null, "boot_system_server_init", uptimeMillis);
            }

            // In case the runtime switched since last boot (such as when
            // the old runtime was removed in an OTA), set the system
            // property so that it is in sync. We can | xq oqi't do this in
            // libnativehelper's JniInvocation::Init code where we already
            // had to fallback to a different runtime because it is
            // running as root and we need to be the system user to set
            // the property. http://b/11463182
            SystemProperties.set("persist.sys.dalvik.vm.lib.2", VMRuntime.getRuntime().vmLibrary());

            // Mmmmmm... more memory!
            VMRuntime.getRuntime().clearGrowthLimit();

            // The system server has to run all of the time, so it needs to be
            // as efficient as possible with its memory usage.
            VMRuntime.getRuntime().setTargetHeapUtilization(0.8f);

            // Some devices rely on runtime fingerprint generation, so make sure
            // we've defined it before booting further.
            Build.ensureFingerprintProperty();

            // Within the system server, it is an error to access Environment paths without
            // explicitly specifying a user.
            Environment.setUserRequired(true);

            // Within the system server, any incoming Bundles should be defused
            // to avoid throwing BadParcelableException.
            BaseBundle.setShouldDefuse(true);

            // Within the system server, when parceling exceptions, include the stack trace
            Parcel.setStackTraceParceling(true);

            // Ensure binder calls into the system always run at foreground priority.
            BinderInternal.disableBackgroundScheduling(true);

            // Increase the number of binder threads in system_server
            BinderInternal.setMaxThreads(sMaxBinderThreads);

            // Prepare the main looper thread (this thread).
            android.os.Process.setThreadPriority(
                    android.os.Process.THREAD_PRIORITY_FOREGROUND);
            android.os.Process.setCanSelfBackground(false);
            Looper.prepareMainLooper();
            Looper.getMainLooper().setSlowLogThresholdMs(
                    SLOW_DISPATCH_THRESHOLD_MS, SLOW_DELIVERY_THRESHOLD_MS);

            // Initialize native services.
            System.loadLibrary("android_servers");

            // Debug builds - allow heap profiling.
            if (Build.IS_DEBUGGABLE) {
                initZygoteChildHeapProfiling();
            }

            // Check whether we failed to shut down last time we tried.
            // This call may not return.
            performPendingShutdown();

            // Initialize the system context.
            createSystemContext();

            // Create the system service manager.
            mSystemServiceManager = new SystemServiceManager(mSystemContext);
            mSystemServiceManager.setStartInfo(mRuntimeRestart,
                    mRuntimeStartElapsedTime, mRuntimeStartUptime);
            LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
            // Prepare the thread pool for init tasks that can be parallelized
            SystemServerInitThreadPool.get();
        } finally {
            traceEnd();  // InitBeforeStartServices
        }

        // Start services.
        try {
            traceBeginAndSlog("StartServices");
            startBootstrapServices();
            startCoreServices();
            startOtherServices();
            SystemServerInitThreadPool.shutdown();
        } catch (Throwable ex) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting system services", ex);
            throw ex;
        } finally {
            traceEnd();
        }

        StrictMode.initVmDefaults(null);

        if (!mRuntimeRestart && !isFirstBootOrUpgrade()) {
            int uptimeMillis = (int) SystemClock.elapsedRealtime();
            MetricsLogger.histogram(null, "boot_system_server_ready", uptimeMillis);
            final int MAX_UPTIME_MILLIS = 60 * 1000;
            if (uptimeMillis > MAX_UPTIME_MILLIS) {
                Slog.wtf(SYSTEM_SERVER_TIMING_TAG,
                        "SystemServer init took too long. uptimeMillis=" + uptimeMillis);
            }
        }

        // Diagnostic to ensure that the system is in a base healthy state. Done here as a common
        // non-zygote process.
        if (!VMRuntime.hasBootImageSpaces()) {
            Slog.wtf(TAG, "Runtime is not running with a boot image!");
        }

        // Loop forever.
        Looper.loop();
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

```java 
    // com.android.server.SystemServer#startBootstrapServices
    /**
     * Starts the small tangle of critical services that are needed to get the system off the
     * ground.  These services have complex mutual dependencies which is why we initialize them all
     * in one place here.  Unless your service is also entwined in these dependencies, it should be
     * initialized in one of the other functions.
     */
    private void startBootstrapServices() {
        // Start the watchdog as early as possible so we can crash the system server
        // if we deadlock during early boot
        traceBeginAndSlog("StartWatchdog");
        final Watchdog watchdog = Watchdog.getInstance();
        watchdog.start();
        traceEnd();

        Slog.i(TAG, "Reading configuration...");
        final String TAG_SYSTEM_CONFIG = "ReadingSystemConfig";
        traceBeginAndSlog(TAG_SYSTEM_CONFIG);
        SystemServerInitThreadPool.get().submit(SystemConfig::getInstance, TAG_SYSTEM_CONFIG);
        traceEnd();

        // Wait for installd to finish starting up so that it has a chance to
        // create critical directories such as /data/user with the appropriate
        // permissions.  We need this to complete before we initialize other services.
        traceBeginAndSlog("StartInstaller");
        Installer installer = mSystemServiceManager.startService(Installer.class);
        traceEnd();

        // In some cases after launching an app we need to access device identifiers,
        // therefore register the device identifier policy before the activity manager.
        traceBeginAndSlog("DeviceIdentifiersPolicyService");
        mSystemServiceManager.startService(DeviceIdentifiersPolicyService.class);
        traceEnd();

        // Uri Grants Manager.
        traceBeginAndSlog("UriGrantsManagerService");
        mSystemServiceManager.startService(UriGrantsManagerService.Lifecycle.class);
        traceEnd();

        // Activity manager runs the show.
        traceBeginAndSlog("StartActivityManager");
        // TODO: Might need to move after migration to WM.
        ActivityTaskManagerService atm = mSystemServiceManager.startService(
                ActivityTaskManagerService.Lifecycle.class).getService();
        mActivityManagerService = ActivityManagerService.Lifecycle.startService(
                mSystemServiceManager, atm);
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);
        mWindowManagerGlobalLock = atm.getGlobalLock();
        traceEnd();

        // Power manager needs to be started early because other services need it.
        // Native daemons may be watching for it to be registered so it must be ready
        // to handle incoming binder calls immediately (including being able to verify
        // the permissions for those calls).
        traceBeginAndSlog("StartPowerManager");
        mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);
        traceEnd();

        traceBeginAndSlog("StartThermalManager");
        mSystemServiceManager.startService(ThermalManagerService.class);
        traceEnd();

        // Now that the power manager has been started, let the activity manager
        // initialize power management features.
        traceBeginAndSlog("InitPowerManagement");
        mActivityManagerService.initPowerManagement();
        traceEnd();

        // Bring up recovery system in case a rescue party needs a reboot
        traceBeginAndSlog("StartRecoverySystemService");
        mSystemServiceManager.startService(RecoverySystemService.class);
        traceEnd();

        // Now that we have the bare essentials of the OS up and running, take
        // note that we just booted, which might send out a rescue party if
        // we're stuck in a runtime restart loop.
        RescueParty.noteBoot(mSystemContext);

        // Manages LEDs and display backlight so we need it to bring up the display.
        traceBeginAndSlog("StartLightsService");
        mSystemServiceManager.startService(LightsService.class);
        traceEnd();

        traceBeginAndSlog("StartSidekickService");
        // Package manager isn't started yet; need to use SysProp not hardware feature
        if (SystemProperties.getBoolean("config.enable_sidekick_graphics", false)) {
            mSystemServiceManager.startService(WEAR_SIDEKICK_SERVICE_CLASS);
        }
        traceEnd();

        // Display manager is needed to provide display metrics before package manager
        // starts up.
        traceBeginAndSlog("StartDisplayManager");
        mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);
        traceEnd();

        // We need the default display before we can initialize the package manager.
        traceBeginAndSlog("WaitForDisplay");
        mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);
        traceEnd();

        // Only run "core" apps if we're encrypting the device.
        String cryptState = VoldProperties.decrypt().orElse("");
        if (ENCRYPTING_STATE.equals(cryptState)) {
            Slog.w(TAG, "Detected encryption in progress - only parsing core apps");
            mOnlyCore = true;
        } else if (ENCRYPTED_STATE.equals(cryptState)) {
            Slog.w(TAG, "Device encrypted - only parsing core apps");
            mOnlyCore = true;
        }

        // Start the package manager.
        if (!mRuntimeRestart) {
            MetricsLogger.histogram(null, "boot_package_manager_init_start",
                    (int) SystemClock.elapsedRealtime());
        }
        traceBeginAndSlog("StartPackageManagerService");
        try {
            Watchdog.getInstance().pauseWatchingCurrentThread("packagemanagermain");
            mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                    mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
        } finally {
            Watchdog.getInstance().resumeWatchingCurrentThread("packagemanagermain");
        }
        mFirstBoot = mPackageManagerService.isFirstBoot();
        mPackageManager = mSystemContext.getPackageManager();
        traceEnd();
        if (!mRuntimeRestart && !isFirstBootOrUpgrade()) {
            MetricsLogger.histogram(null, "boot_package_manager_init_ready",
                    (int) SystemClock.elapsedRealtime());
        }
        // Manages A/B OTA dexopting. This is a bootstrap service as we need it to rename
        // A/B artifacts after boot, before anything else might touch/need them.
        // Note: this isn't needed during decryption (we don't have /data anyways).
        if (!mOnlyCore) {
            boolean disableOtaDexopt = SystemProperties.getBoolean("config.disable_otadexopt",
                    false);
            if (!disableOtaDexopt) {
                traceBeginAndSlog("StartOtaDexOptService");
                try {
                    Watchdog.getInstance().pauseWatchingCurrentThread("moveab");
                    OtaDexoptService.main(mSystemContext, mPackageManagerService);
                } catch (Throwable e) {
                    reportWtf("starting OtaDexOptService", e);
                } finally {
                    Watchdog.getInstance().resumeWatchingCurrentThread("moveab");
                    traceEnd();
                }
            }
        }

        traceBeginAndSlog("StartUserManagerService");
        mSystemServiceManager.startService(UserManagerService.LifeCycle.class);
        traceEnd();

        // Initialize attribute cache used to cache resources from packages.
        traceBeginAndSlog("InitAttributerCache");
        AttributeCache.init(mSystemContext);
        traceEnd();

        // Set up the Application instance for the system process and get started.
        traceBeginAndSlog("SetSystemProcess");
        mActivityManagerService.setSystemProcess();
        traceEnd();

        // Complete the watchdog setup with an ActivityManager instance and listen for reboots
        // Do this only after the ActivityManagerService is properly started as a system process
        traceBeginAndSlog("InitWatchdog");
        watchdog.init(mSystemContext, mActivityManagerService);
        traceEnd();

        // DisplayManagerService needs to setup android.display scheduling related policies
        // since setSystemProcess() would have overridden policies due to setProcessGroup
        mDisplayManagerService.setupSchedulerPolicies();

        // Manages Overlay packages
        traceBeginAndSlog("StartOverlayManagerService");
        mSystemServiceManager.startService(new OverlayManagerService(mSystemContext, installer));
        traceEnd();

        traceBeginAndSlog("StartSensorPrivacyService");
        mSystemServiceManager.startService(new SensorPrivacyService(mSystemContext));
        traceEnd();

        if (SystemProperties.getInt("persist.sys.displayinset.top", 0) > 0) {
            // DisplayManager needs the overlay immediately.
            mActivityManagerService.updateSystemUiContext();
            LocalServices.getService(DisplayManagerInternal.class).onOverlayChanged();
        }

        // The sensor service needs access to package manager service, app ops
        // service, and permissions service, therefore we start it after them.
        // Start sensor service in a separate thread. Completion should be checked
        // before using it.
        mSensorServiceStart = SystemServerInitThreadPool.get().submit(() -> {
            TimingsTraceLog traceLog = new TimingsTraceLog(
                    SYSTEM_SERVER_TIMING_ASYNC_TAG, Trace.TRACE_TAG_SYSTEM_SERVER);
            traceLog.traceBegin(START_SENSOR_SERVICE);
            startSensorService();
            traceLog.traceEnd();
        }, START_SENSOR_SERVICE);
    }
```

### PackageManagerService#main()

```java 
    // com.android.server.pm.PackageManagerService#main
    public static PackageManagerService main(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
        // Self-check for initial settings.
        PackageManagerServiceCompilerMapping.checkProperties();

        PackageManagerService m = new PackageManagerService(context, installer,
                factoryTest, onlyCore);
        m.enableSystemUserPackages();
        ServiceManager.addService("package", m);
        final PackageManagerNative pmn = m.new PackageManagerNative();
        ServiceManager.addService("package_native", pmn);
        return m;
    }
```

#### BOOT_PROGRESS_PMS_START

```java 

```

#### BOOT_PROGRESS_PMS_SYSTEM_SCAN_START

```java 

```

#### BOOT_PROGRESS_PMS_DATAS_SCAN_START

```java 

```

#### BOOT_PROGRESS_PMS_SCAN_END

```java 

```

#### BOOT_PROGRESS_PMS_READY

```java 

```

## PackageInstaller创建

PMS的作用是对APK进行安装、解析、删除以及卸载等，那么这里以安装APK为例子分析，我们需要知道从应用安装器PackageInstaller到PMS处理APK安装的流程

## PackageInstaller安装APK

```java 

```

## PMS处理APK

```java 

```

## PackageParser解析APK

```java 

```

## 流程总结



