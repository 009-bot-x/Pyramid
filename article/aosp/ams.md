> 本文基于 Android 9.0 , 代码仓库地址 ： [android_9.0.0_r45](https://github.com/lulululbj/android_9.0.0_r45)
>
> 系列文章目录：
>
> [Java 世界的盘古和女娲 —— Zygote](https://juejin.im/post/5d8f73bf51882555b149dc64)
>
> [Zygote 家的大儿子 —— SystemServer](https://juejin.im/post/5da341f451882561ba64b9da)
>
> [Android 世界中，谁喊醒了 Zygote ？](https://juejin.im/post/5da5e7da518825740064f951)


>
> 文中相关源码链接：
>
> [SystemServer.java](https://github.com/lulululbj/android_9.0.0_r45/blob/master/frameworks/base/services/java/com/android/server/SystemServer.java)
>
> [ActivityManagerService.java](https://github.com/lulululbj/android_9.0.0_r45/blob/master/frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java)

之前介绍 [SystemServer 启动流程](https://juejin.im/post/5da341f451882561ba64b9da) 的时候说到，SystemServer 进程启动了一系列的系统服务，**ActivityManagerService** 是其中最核心的服务之一。它和四大组件的启动、切换、调度及应用进程的管理和调度息息相关，其重要性不言而喻。本文主要介绍其启动流程，它是在 `SystemServer` 的 `main()` 中启动的，整个启动流程经历了 `startBootstrapServices` 、`startCoreService()` 、`startOtherService()`。下面就顺着源码来捋一捋 `ActivityManagerService` 的启动流程，下文中简称 `AMS`。

```java
private void startBootstrapServices() {
    ...

    // 1. AMS 初始化
    mActivityManagerService = mSystemServiceManager.startService(
            ActivityManagerService.Lifecycle.class).getService();
    // 设置 AMS 的系统服务管理器
    mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
    // 设置 AMS 的应用安装器
    mActivityManagerService.setInstaller(installer);
    ...
    mActivityManagerService.initPowerManagement();
    ...
    // 2. AMS.setSystemProcess()
    mActivityManagerService.setSystemProcess();
    ...
}

private void startCoreServices{
    ...

    mActivityManagerService.setUsageStatsManager(
            LocalServices.getService(UsageStatsManagerInternal.class));
    ...
}

private void startherService{
    ...

    // 3. 安装系统 Provider
    mActivityManagerService.installSystemProviders();
    ...
    final Watchdog watchdog = Watchdog.getInstance();
            watchdog.init(context, mActivityManagerService);
    ...
    mActivityManagerService.setWindowManager(wm);
    ...
    networkPolicy = new NetworkPolicyManagerService(context, mActivityManagerService,
                        networkManagement);
    ...
    if (safeMode) {
        traceBeginAndSlog("EnterSafeModeAndDisableJitCompilation");
        mActivityManagerService.enterSafeMode();
        // Disable the JIT for the system_server process
        VMRuntime.getRuntime().disableJitCompilation();
        traceEnd();
    }
    ...
    mPowerManagerService.systemReady(mActivityManagerService.getAppOpsService());
    ...
    // 4. AMS.systemReady()
    mActivityManagerService.systemReady(() -> {
        ...
        mActivityManagerService.startObservingNativeCrashes();
    }
}
```

## AMS 初始化

```java
mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
```

AMS 通过 `SystemServiceManager.startService()` 方法初始化，`startService()` 在之前的文章中已经分析过，其作用是根据参数传入的类通过反射进行实例化，并回调其 `onStart()` 方法。注意这里传入的是 `ActivityManagerService.Lifecycle.class`，`LifeCycle` 是 AMS 的一个静态内部类。

```java
public static final class Lifecycle extends SystemService {
    private final ActivityManagerService mService;

    // 构造函数中新建 ActivityManagerService 对象
    public Lifecycle(Context context) {
        super(context);
        mService = new ActivityManagerService(context);
    }

    @Override
    public void onStart() {
        mService.start();
    }

    @Override
    public void onBootPhase(int phase) {
        mService.mBootPhase = phase;
        if (phase == PHASE_SYSTEM_SERVICES_READY) {
            mService.mBatteryStatsService.systemServicesReady();
            mService.mServices.systemServicesReady();
        }
    }

    @Override
    public void onCleanupUser(int userId) {
        mService.mBatteryStatsService.onCleanupUser(userId);
    }

    public ActivityManagerService getService() {
        return mService;
    }
}
```

`Lifecycle` 的构造函数中初始化了 AMS。再来看看 AMS 的构造函数。

```java
public ActivityManagerService(Context systemContext) {
    mInjector = new Injector();
    // AMS 上下文
    mContext = systemContext;

    mFactoryTest = FactoryTest.getMode();
    // ActivityThread 对象
    mSystemThread = ActivityThread.currentActivityThread();
    // ContextImpl 对象
    mUiContext = mSystemThread.getSystemUiContext();

    mPermissionReviewRequired = mContext.getResources().getBoolean(
            com.android.internal.R.bool.config_permissionReviewRequired);

    // 线程名为 ActivityManager 的前台线程，ServiceThread 继承于 HandlerThread
    mHandlerThread = new ServiceThread(TAG,
            THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
    mHandlerThread.start();
    // 获取 mHandlerThread 的 Handler 对象
    mHandler = new MainHandler(mHandlerThread.getLooper());
    // 创建名为 android.ui 的线程
    mUiHandler = mInjector.getUiHandler(this);

    // 不知道什么作用
    mProcStartHandlerThread = new ServiceThread(TAG + ":procStart",
            THREAD_PRIORITY_FOREGROUND, false /* allowIo */);
    mProcStartHandlerThread.start();
    mProcStartHandler = new Handler(mProcStartHandlerThread.getLooper());

    mConstants = new ActivityManagerConstants(this, mHandler);

    /* static; one-time init here */
    // 根据优先级 kill 后台应用进程
    if (sKillHandler == null) {
        sKillThread = new ServiceThread(TAG + ":kill",
                THREAD_PRIORITY_BACKGROUND, true /* allowIo */);
        sKillThread.start();
        sKillHandler = new KillHandler(sKillThread.getLooper());
    }

    // 前台广播队列，超时时间为 10 秒
    mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
            "foreground", BROADCAST_FG_TIMEOUT, false);
    // 后台广播队列，超时时间为 60 秒
    mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
            "background", BROADCAST_BG_TIMEOUT, true);
    mBroadcastQueues[0] = mFgBroadcastQueue;
    mBroadcastQueues[1] = mBgBroadcastQueue;

    // 创建 ActiveServices
    mServices = new ActiveServices(this);
    mProviderMap = new ProviderMap(this);
    // 创建 AppErrors，用于处理应用中的错误
    mAppErrors = new AppErrors(mUiContext, this);

    // 创建 /data/system 目录
    File dataDir = Environment.getDataDirectory();
    File systemDir = new File(dataDir, "system");
    systemDir.mkdirs();

    mAppWarnings = new AppWarnings(this, mUiContext, mHandler, mUiHandler, systemDir);

    // TODO: Move creation of battery stats service outside of activity manager service.
    // 创建 BatteryStatsService，其信息保存在 /data/system/procstats 中
    // 这里有个 TODO，打算把 BatteryStatsService 的创建移除 AMS
    mBatteryStatsService = new BatteryStatsService(systemContext, systemDir, mHandler);
    mBatteryStatsService.getActiveStatistics().readLocked();
    mBatteryStatsService.scheduleWriteToDisk();
    mOnBattery = DEBUG_POWER ? true
            : mBatteryStatsService.getActiveStatistics().getIsOnBattery();
    mBatteryStatsService.getActiveStatistics().setCallback(this);

    // 创建 ProcessStatsService，并将其信息保存在 /data/system/procstats 中
    mProcessStats = new ProcessStatsService(this, new File(systemDir, "procstats"));

    mAppOpsService = mInjector.getAppOpsService(new File(systemDir, "appops.xml"), mHandler);

    // 定义 ContentProvider 访问指定 Uri 数据的权限
    mGrantFile = new AtomicFile(new File(systemDir, "urigrants.xml"), "uri-grants");

    // 多用户管理
    mUserController = new UserController(this);

    mVrController = new VrController(this);

    // 获取 OpenGL 版本
    GL_ES_VERSION = SystemProperties.getInt("ro.opengles.version",
        ConfigurationInfo.GL_ES_VERSION_UNDEFINED);

    if (SystemProperties.getInt("sys.use_fifo_ui", 0) != 0) {
        mUseFifoUiScheduling = true;
    }

    mTrackingAssociations = "1".equals(SystemProperties.get("debug.track-associations"));
    mTempConfig.setToDefaults();
    mTempConfig.setLocales(LocaleList.getDefault());
    mConfigurationSeq = mTempConfig.seq = 1;
    // 创建 ActivityStackSupervisor ，用于管理 Activity 任务栈
    mStackSupervisor = createStackSupervisor();
    mStackSupervisor.onConfigurationChanged(mTempConfig);
    mKeyguardController = mStackSupervisor.getKeyguardController();
    mCompatModePackages = new CompatModePackages(this, systemDir, mHandler);
    mIntentFirewall = new IntentFirewall(new IntentFirewallInterface(), mHandler);
    mTaskChangeNotificationController =
            new TaskChangeNotificationController(this, mStackSupervisor, mHandler);
    // 创建 ActivityStartController 对象，用于管理 Activity 的启动
    mActivityStartController = new ActivityStartController(this);
    // 创建最近任务栈 RecentTask 对象
    mRecentTasks = createRecentTasks();
    mStackSupervisor.setRecentTasks(mRecentTasks);
    mLockTaskController = new LockTaskController(mContext, mStackSupervisor, mHandler);
    mLifecycleManager = new ClientLifecycleManager();

    // 创建 CpuTracker 线程，追踪 CPU 状态
    mProcessCpuThread = new Thread("CpuTracker") {
        @Override
        public void run() {
            synchronized (mProcessCpuTracker) {
                mProcessCpuInitLatch.countDown();
                mProcessCpuTracker.init(); // 初始化 ProcessCpuTracker。注意同步问题
            }
            while (true) {
                try {
                    try {
                        synchronized(this) {
                            final long now = SystemClock.uptimeMillis();
                            long nextCpuDelay = (mLastCpuTime.get()+MONITOR_CPU_MAX_TIME)-now;
                            long nextWriteDelay = (mLastWriteTime+BATTERY_STATS_TIME)-now;
                            //Slog.i(TAG, "Cpu delay=" + nextCpuDelay
                            //        + ", write delay=" + nextWriteDelay);
                            if (nextWriteDelay < nextCpuDelay) {
                                nextCpuDelay = nextWriteDelay;
                            }
                            if (nextCpuDelay > 0) {
                                mProcessCpuMutexFree.set(true);
                                this.wait(nextCpuDelay);
                            }
                        }
                    } catch (InterruptedException e) {
                    }
                    // 更新 Cpu 统计信息
                    updateCpuStatsNow();
                } catch (Exception e) {
                    Slog.e(TAG, "Unexpected exception collecting process stats", e);
                }
            }
        }
    };

    // hidden api 设置
    mHiddenApiBlacklist = new HiddenApiSettings(mHandler, mContext);

    // 设置 Watchdog 监控
    Watchdog.getInstance().addMonitor(this);
    Watchdog.getInstance().addThread(mHandler);

    // bind background thread to little cores
    // this is expected to fail inside of framework tests because apps can't touch cpusets directly
    // make sure we've already adjusted system_server's internal view of itself first
    // 更新进程的 oom_adj 值
    updateOomAdjLocked();
    try {
        Process.setThreadGroupAndCpuset(BackgroundThread.get().getThreadId(),
                Process.THREAD_GROUP_BG_NONINTERACTIVE);
    } catch (Exception e) {
        Slog.w(TAG, "Setting background thread cpuset failed");
    }
}
```

AMS 的构造函数中做了很多事情，代码中作了很多注释，这里就不再展开细说了。

`LifeCycle` 的构造函数中初始化了 AMS，然后会调用 `LifeCycle.onStart()`，最终调用的是 `AMS.start()` 方法。

```java
    private void start() {
    // 移除所有进程组
    removeAllProcessGroups();
    // 启动构造函数中创建的 CpuTracker 线程，监控 cpu 使用情况
    mProcessCpuThread.start();

    // 启动 BatteryStatsService，统计电池信息
    mBatteryStatsService.publish();
    mAppOpsService.publish(mContext);
    // 启动 LocalService ，将 ActivityManagerInternal 加入服务列表
    LocalServices.addService(ActivityManagerInternal.class, new LocalService());
    // 等待 mProcessCpuThread 线程中的同步代码块执行完毕。
    // 在执行 mProcessCpuTracker.init() 方法时访问 mProcessCpuTracker 将阻塞
    try {
        mProcessCpuInitLatch.await();
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        throw new IllegalStateException("Interrupted wait during start");
    }
}
```

AMS 的初始化工作到这里就基本结束了，我们再回到 `startBootstrapServices()` 中，看看 AMS 的下一步动作 `setSystemProcess()` 。

## AMS.setSystemProcess()

```java
public void setSystemProcess() {
    try {
        // 注册各种服务
        // 注册 AMS
        ServiceManager.addService(Context.ACTIVITY_SERVICE, this, /* allowIsolated= */ true,
                DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PRIORITY_NORMAL | DUMP_FLAG_PROTO);
        // 注册进程统计服务
        ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
        // 注册内存信息服务
        ServiceManager.addService("meminfo", new MemBinder(this), /* allowIsolated= */ false,
                DUMP_FLAG_PRIORITY_HIGH);
        // 注册 GraphicsBinder
        ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
        // 注册 DbBinder
        ServiceManager.addService("dbinfo", new DbBinder(this));
        if (MONITOR_CPU_USAGE) {
            // 注册 DbBinder
            ServiceManager.addService("cpuinfo", new CpuBinder(this),
                    /* allowIsolated= */ false, DUMP_FLAG_PRIORITY_CRITICAL);
        }
        // 注册权限管理者 PermissionController
        ServiceManager.addService("permission", new PermissionController(this
        // 注册进程信息服务 ProcessInfoService
        ServiceManager.addService("processinfo", new ProcessInfoService(this));

        // 获取包名为 android 的应用信息，framework-res.apk
        ApplicationInfo info = mContext.getPackageManager().getApplicationInfo(
                "android", STOCK_PM_FLAGS | MATCH_SYSTEM_ONLY);
        mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());

        synchronized (this) {
            // 创建 ProcessRecord
            ProcessRecord app = newProcessRecordLocked(info, info.processName, false, 0);
            app.persistent = true;
            app.pid = MY_PID;
            app.maxAdj = ProcessList.SYSTEM_ADJ;
            app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
            synchronized (mPidsSelfLocked) {
                mPidsSelfLocked.put(app.pid, app);
            }
            // 更新 mLruProcesses
            updateLruProcessLocked(app, false, null);
            // 更新进程对应的 oom_adj 值
            updateOomAdjLocked();
        }
    } catch (PackageManager.NameNotFoundException e) {
        throw new RuntimeException(
                "Unable to find android system package", e);
    }

    // Start watching app ops after we and the package manager are up and running.
    // 当 packager manager 启动并运行时开始监听 app ops
    mAppOpsService.startWatchingMode(AppOpsManager.OP_RUN_IN_BACKGROUND, null,
            new IAppOpsCallback.Stub() {
                @Override public void opChanged(int op, int uid, String packageName) {
                    if (op == AppOpsManager.OP_RUN_IN_BACKGROUND && packageName != null) {
                        if (mAppOpsService.checkOperation(op, uid, packageName)
                                != AppOpsManager.MODE_ALLOWED) {
                            runInBackgroundDisabled(uid);
                        }
                    }
                }
            });
}
```

`setSystemProcess()` 的主要工作就是向 ServiceManager 注册关联的系统服务。

## AMS.installSystemProviders()
```java
public final void installSystemProviders() {
        List<ProviderInfo> providers;
        synchronized (this) {
            ProcessRecord app = mProcessNames.get("system", SYSTEM_UID);
            providers = generateApplicationProvidersLocked(app);
            if (providers != null) {
                for (int i=providers.size()-1; i>=0; i--) {
                    ProviderInfo pi = (ProviderInfo)providers.get(i);
                    if ((pi.applicationInfo.flags&ApplicationInfo.FLAG_SYSTEM) == 0) {
                        Slog.w(TAG, "Not installing system proc provider " + pi.name
                                + ": not system .apk");
                        // 移除非系统 Provier
                        providers.remove(i);
                    }
                }
            }
        }

        // 安装系统 Provider
        if (providers != null) {
            mSystemThread.installSystemProviders(providers);
        }

        synchronized (this) {
            mSystemProvidersInstalled = true;
        }

        mConstants.start(mContext.getContentResolver());
        // 创建 CoreSettingsObserver ，监控核心设置的变化
        mCoreSettingsObserver = new CoreSettingsObserver(this);
        // 创建 FontScaleSettingObserver，监控字体的变化
        mFontScaleSettingObserver = new FontScaleSettingObserver();
        // 创建 DevelopmentSettingsObserver
        mDevelopmentSettingsObserver = new DevelopmentSettingsObserver();
        GlobalSettingsToPropertiesMapper.start(mContext.getContentResolver());

        // Now that the settings provider is published we can consider sending
        // in a rescue party.
        RescueParty.onSettingsProviderPublished(mContext);

        //mUsageStatsService.monitorPackages();
    }
```

`installSystemProviders()` 的主要工作是安装系统 Proviers。

## AMS.systemReady()

`AMS.systemReady()` 是 AMS 启动流程的最后一步了。

```java
public void systemReady(final Runnable goingCallback, TimingsTraceLog traceLog) {
    ...
    synchronized(this) {
        if (mSystemReady) { // 首次调用 mSystemReady 为 false
            // If we're done calling all the receivers, run the next "boot phase" passed in
            // by the SystemServer
            if (goingCallback != null) {
                goingCallback.run();
            }
            return;
        }

        // 一系列 systemReady()
        mHasHeavyWeightFeature = mContext.getPackageManager().hasSystemFeature(
                PackageManager.FEATURE_CANT_SAVE_STATE);
        mLocalDeviceIdleController
                = LocalServices.getService(DeviceIdleController.LocalService.class);
        mAssistUtils = new AssistUtils(mContext);
        mVrController.onSystemReady();
        // Make sure we have the current profile info, since it is needed for security checks.
        mUserController.onSystemReady();
        mRecentTasks.onSystemReadyLocked();
        mAppOpsService.systemReady();
        mSystemReady = true;
    }

    ...

    ArrayList<ProcessRecord> procsToKill = null;
    synchronized(mPidsSelfLocked) {
        for (int i=mPidsSelfLocked.size()-1; i>=0; i--) {
            ProcessRecord proc = mPidsSelfLocked.valueAt(i);
            if (!isAllowedWhileBooting(proc.info)){
                if (procsToKill == null) {
                    procsToKill = new ArrayList<ProcessRecord>();
                }
                procsToKill.add(proc);
            }
        }
    }

    synchronized(this) {
        if (procsToKill != null) {
            for (int i=procsToKill.size()-1; i>=0; i--) {
                ProcessRecord proc = procsToKill.get(i);
                removeProcessLocked(proc, true, false, "system update done");
            }
        }

        // Now that we have cleaned up any update processes, we
        // are ready to start launching real processes and know that
        // we won't trample on them any more.
        mProcessesReady = true;
    }

    ...

    if (goingCallback != null) goingCallback.run();
    mBatteryStatsService.noteEvent(BatteryStats.HistoryItem.EVENT_USER_RUNNING_START,
            Integer.toString(currentUserId), currentUserId);
    mBatteryStatsService.noteEvent(BatteryStats.HistoryItem.EVENT_USER_FOREGROUND_START,
            Integer.toString(currentUserId), currentUserId);
    // 回调所有 SystemService 的 onStartUser() 方法
    mSystemServiceManager.startUser(currentUserId);

    synchronized (this) {
        // Only start up encryption-aware persistent apps; once user is
        // unlocked we'll come back around and start unaware apps
        startPersistentApps(PackageManager.MATCH_DIRECT_BOOT_AWARE);

        // Start up initial activity.
        mBooting = true;
        // Enable home activity for system user, so that the system can always boot. We don't
        // do this when the system user is not setup since the setup wizard should be the one
        // to handle home activity in this case.
        if (UserManager.isSplitSystemUser() &&
                Settings.Secure.getInt(mContext.getContentResolver(),
                     Settings.Secure.USER_SETUP_COMPLETE, 0) != 0) {
            ComponentName cName = new ComponentName(mContext, SystemUserHomeActivity.class);
            try {
                AppGlobals.getPackageManager().setComponentEnabledSetting(cName,
                        PackageManager.COMPONENT_ENABLED_STATE_ENABLED, 0,
                        UserHandle.USER_SYSTEM);
            } catch (RemoteException e) {
                throw e.rethrowAsRuntimeException();
            }
        }

        // 启动桌面 Home 应用
        startHomeActivityLocked(currentUserId, "systemReady");

       ...

        long ident = Binder.clearCallingIdentity();
        try {
            // 发送广播 USER_STARTED
            Intent intent = new Intent(Intent.ACTION_USER_STARTED);
            intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY
                    | Intent.FLAG_RECEIVER_FOREGROUND);
            intent.putExtra(Intent.EXTRA_USER_HANDLE, currentUserId);
            broadcastIntentLocked(null, null, intent,
                    null, null, 0, null, null, null, OP_NONE,
                    null, false, false, MY_PID, SYSTEM_UID,
                    currentUserId);
            // 发送广播 USER_STARTING
            intent = new Intent(Intent.ACTION_USER_STARTING);
            intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
            intent.putExtra(Intent.EXTRA_USER_HANDLE, currentUserId);
            broadcastIntentLocked(null, null, intent,
                    null, new IIntentReceiver.Stub() {
                        @Override
                        public void performReceive(Intent intent, int resultCode, String data,
                                Bundle extras, boolean ordered, boolean sticky, int sendingUser)
                                throws RemoteException {
                        }
                    }, 0, null, null,
                    new String[] {INTERACT_ACROSS_USERS}, OP_NONE,
                    null, true, false, MY_PID, SYSTEM_UID, UserHandle.USER_ALL);
        } catch (Throwable t) {
            Slog.wtf(TAG, "Failed sending first user broadcasts", t);
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
        mStackSupervisor.resumeFocusedStackTopActivityLocked();
        mUserController.sendUserSwitchBroadcasts(-1, currentUserId);

        BinderInternal.nSetBinderProxyCountWatermarks(6000,5500);
        BinderInternal.nSetBinderProxyCountEnabled(true);
        BinderInternal.setBinderProxyCountCallback(
            new BinderInternal.BinderProxyLimitListener() {
                @Override
                public void onLimitReached(int uid) {
                    if (uid == Process.SYSTEM_UID) {
                        Slog.i(TAG, "Skipping kill (uid is SYSTEM)");
                    } else {
                        killUid(UserHandle.getAppId(uid), UserHandle.getUserId(uid),
                                "Too many Binders sent to SYSTEM");
                    }
                }
            }, mHandler);
    }
}
```

`systemReady()` 方法源码很长，上面做了很多删减。注意其中的 `startHomeActivityLocked()` 方法会启动桌面 Activity 。

```java
boolean startHomeActivityLocked(int userId, String reason) {
    ...
    Intent intent = getHomeIntent();
    ActivityInfo aInfo = resolveActivityInfo(intent, STOCK_PM_FLAGS, userId);
    if (aInfo != null) {
        intent.setComponent(new ComponentName(aInfo.applicationInfo.packageName, aInfo.name));
        // Don't do this if the home app is currently being
        // instrumented.
        aInfo = new ActivityInfo(aInfo);
        aInfo.applicationInfo = getAppInfoForUser(aInfo.applicationInfo, userId);
        ProcessRecord app = getProcessRecordLocked(aInfo.processName,
                aInfo.applicationInfo.uid, true);
        if (app == null || app.instr == null) {
            intent.setFlags(intent.getFlags() | FLAG_ACTIVITY_NEW_TASK);
            final int resolvedUserId = UserHandle.getUserId(aInfo.applicationInfo.uid);
            // For ANR debugging to verify if the user activity is the one that actually
            // launched.
            final String myReason = reason + ":" + userId + ":" + resolvedUserId;
            // 启动桌面 Activity
            mActivityStartController.startHomeActivity(intent, aInfo, myReason);
        }
    } else {
        Slog.wtf(TAG, "No home screen found for " + intent, new Throwable());
    }

    return true;
}
```

`ActivityStartController` 负责启动 Activity，至此，桌面应用就启动了。

## 最后

整篇写下来感觉就像小学生流水日记一样，但是读源码，就像有钱人的生活一样，往往是那么朴实无华，且枯燥。我不经意间看了看我的劳力士，太晚了，时序图后面再补上吧！

最后，下集预告，接着这篇，`Activity` 的启动流程分析，敬请期待！
> 文章首发微信公众号： **`秉心说`** ， 专注 Java 、 Android 原创知识分享，LeetCode 题解。
>
> 更多最新原创文章，扫码关注我吧！

![](https://user-gold-cdn.xitu.io/2019/4/27/16a5f352eab602c4?w=2800&h=800&f=jpeg&s=178470)
