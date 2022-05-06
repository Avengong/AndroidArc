# AMS 启动流程

# systemcontext

# localservices

# localservice

# 一、缘起 SystemServer 进程

在 SystemServer 进程被拉起后，会开启binder线程，不断的从驱动读写数据。 然后会进入 SystemServer.java 类的systemMain()
方法，调用startBootstrapServices()。 开启一系列的 service 服务。其中AMS就在其中。

```
private void startBootstrapServices(@NonNull TimingsTraceAndSlog t) {
...
1 启动 Installer service
Installer installer = mSystemServiceManager.startService(Installer.class);
...

// Activity manager runs the show. Activity的管理类，开始初始化
// TODO: Might need to move after migration to WM.
//  2 启动一个 ActivityTaskManagerService 
ActivityTaskManagerService atm = mSystemServiceManager.startService(
        ActivityTaskManagerService.Lifecycle.class).getService();
//3， 启动 AMS 
mActivityManagerService = ActivityManagerService.Lifecycle.startService(
        mSystemServiceManager, atm);
mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
//设置installer 
mActivityManagerService.setInstaller(installer);
mWindowManagerGlobalLock = atm.getGlobalLock();
...


// 4 注册服务 
 mActivityManagerService.setSystemProcess();
 
 
//PMS
mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);

...
mActivityManagerService.initPowerManagement();
...


mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);

mSystemServiceManager.startBootPhase(t, SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);

//PMS
mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
        mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);

    //...

}


 
 
```

## 1.1 Installer service

```
// 继承了 SystemService
public class Installer extends SystemService
```

继承了 SystemService。

```
1 启动 Installer service
Installer installer = mSystemServiceManager.startService(Installer.class);
```

启动install service。会回调 onStart()方法：

```
 @Override
    public void onStart() {
        if (mIsolated) {
            mInstalld = null;
        } else {
            // 开始从SM中获取已注册的服务
            connect();
        }
    }

    private void connect() {
        IBinder binder = ServiceManager.getService("installd");
        if (binder != null) {
            try {
                // 如果binder服务已存在，则注册死亡通知，再次连接
                binder.linkToDeath(new DeathRecipient() {
                    @Override
                    public void binderDied() {
                        Slog.w(TAG, "installd died; reconnecting");
                        connect();
                    }
                }, 0);
            } catch (RemoteException e) {
                binder = null;
            }
        }

        if (binder != null) {
            //mInstalld 转成本地代理服务对象
            mInstalld = IInstalld.Stub.asInterface(binder);
            try {
                // 
                invalidateMounts();
            } catch (InstallerException ignored) {
            }
        } else {
            Slog.w(TAG, "installd not found; trying again");
            BackgroundThread.getHandler().postDelayed(() -> {
                connect();
            }, DateUtils.SECOND_IN_MILLIS);// 如果还未注册，则每隔1s 在检测一次
        }
    }
```

总结：

从 IInstalled 接口 可以看出，提供了 createAppData、clearAppData、getAppSize、dexopt、destroyUserData、 等等功能。
在启动其他服务之前我们需要 Installed 服务去创建关键性的目录，如 /data/user。

```
interface IInstalld {
21      void createUserData(@nullable @utf8InCpp String uuid, int userId, int userSerial, int flags);
22      void destroyUserData(@nullable @utf8InCpp String uuid, int userId, int flags);
23  
24      long createAppData(@nullable @utf8InCpp String uuid, in @utf8InCpp String packageName,
25              int userId, int flags, int appId, in @utf8InCpp String seInfo, int targetSdkVersion);
26      void restoreconAppData(@nullable @utf8InCpp String uuid, @utf8InCpp String packageName,
27              int userId, int flags, int appId, @utf8InCpp String seInfo);
28      void migrateAppData(@nullable @utf8InCpp String uuid, @utf8InCpp String packageName,
29              int userId, int flags);
30      void clearAppData(@nullable @utf8InCpp String uuid, @utf8InCpp String packageName,
31              int userId, int flags, long ceDataInode);
32      void destroyAppData(@nullable @utf8InCpp String uuid, @utf8InCpp String packageName,
33              int userId, int flags, long ceDataInode);
34  
35      void fixupAppData(@nullable @utf8InCpp String uuid, int flags);
36  
37      long[] getAppSize(@nullable @utf8InCpp String uuid, in @utf8InCpp String[] packageNames,
38              int userId, int flags, int appId, in long[] ceDataInodes,
39              in @utf8InCpp String[] codePaths);
40      long[] getUserSize(@nullable @utf8InCpp String uuid, int userId, int flags, in int[] appIds);
41      long[] getExternalSize(@nullable @utf8InCpp String uuid, int userId, int flags, in int[] appIds);
42  
43      void setAppQuota(@nullable @utf8InCpp String uuid, int userId, int appId, long cacheQuota);
44  
45      void moveCompleteApp(@nullable @utf8InCpp String fromUuid, @nullable @utf8InCpp String toUuid,
46              @utf8InCpp String packageName, @utf8InCpp String dataAppName, int appId,
47              @utf8InCpp String seInfo, int targetSdkVersion);
48  
49      void dexopt(@utf8InCpp String apkPath, int uid, @nullable @utf8InCpp String packageName,
50              @utf8InCpp String instructionSet, int dexoptNeeded,
51              @nullable @utf8InCpp String outputPath, int dexFlags,
52              @utf8InCpp String compilerFilter, @nullable @utf8InCpp String uuid,
53              @nullable @utf8InCpp String sharedLibraries,
54              @nullable @utf8InCpp String seInfo, boolean downgrade, int targetSdkVersion,
55              @nullable @utf8InCpp String profileName,
56              @nullable @utf8InCpp String dexMetadataPath,
57              @nullable @utf8InCpp String compilationReason);
58      boolean compileLayouts(@utf8InCpp String apkPath, @utf8InCpp String packageName,
59              @utf8InCpp String outDexFile, int uid);
60  
61      void rmdex(@utf8InCpp String codePath, @utf8InCpp String instructionSet);
62  
63      boolean mergeProfiles(int uid, @utf8InCpp String packageName, @utf8InCpp String profileName);
64      boolean dumpProfiles(int uid, @utf8InCpp String packageName, @utf8InCpp String  profileName,
65              @utf8InCpp String codePath);
66      boolean copySystemProfile(@utf8InCpp String systemProfile, int uid,
67              @utf8InCpp String packageName, @utf8InCpp String profileName);
68      void clearAppProfiles(@utf8InCpp String packageName, @utf8InCpp String profileName);
69      void destroyAppProfiles(@utf8InCpp String packageName);
70  
71      boolean createProfileSnapshot(int appId, @utf8InCpp String packageName,
72              @utf8InCpp String profileName, @utf8InCpp String classpath);
73      void destroyProfileSnapshot(@utf8InCpp String packageName, @utf8InCpp String profileName);
74  
75      void idmap(@utf8InCpp String targetApkPath, @utf8InCpp String overlayApkPath, int uid);
76      void removeIdmap(@utf8InCpp String overlayApkPath);
77      void rmPackageDir(@utf8InCpp String packageDir);
78      void markBootComplete(@utf8InCpp String instructionSet);
79      void freeCache(@nullable @utf8InCpp String uuid, long targetFreeBytes,
80              long cacheReservedBytes, int flags);
81      void linkNativeLibraryDirectory(@nullable @utf8InCpp String uuid,
82              @utf8InCpp String packageName, @utf8InCpp String nativeLibPath32, int userId);
83      void createOatDir(@utf8InCpp String oatDir, @utf8InCpp String instructionSet);
84      void linkFile(@utf8InCpp String relativePath, @utf8InCpp String fromBase,
85              @utf8InCpp String toBase);
86      void moveAb(@utf8InCpp String apkPath, @utf8InCpp String instructionSet,
87              @utf8InCpp String outputPath);
88      void deleteOdex(@utf8InCpp String apkPath, @utf8InCpp String instructionSet,
89              @nullable @utf8InCpp String outputPath);
90      void installApkVerity(@utf8InCpp String filePath, in FileDescriptor verityInput,
91              int contentSize);
92      void assertFsverityRootHashMatches(@utf8InCpp String filePath, in byte[] expectedHash);
93  
94      boolean reconcileSecondaryDexFile(@utf8InCpp String dexPath, @utf8InCpp String pkgName,
95          int uid, in @utf8InCpp String[] isas, @nullable @utf8InCpp String volume_uuid,
96          int storage_flag);
97  
98      byte[] hashSecondaryDexFile(@utf8InCpp String dexPath, @utf8InCpp String pkgName,
99          int uid, @nullable @utf8InCpp String volumeUuid, int storageFlag);
100  
101      void invalidateMounts();
102      boolean isQuotaSupported(@nullable @utf8InCpp String uuid);
103  
104      boolean prepareAppProfile(@utf8InCpp String packageName,
105          int userId, int appId, @utf8InCpp String profileName, @utf8InCpp String codePath,
106          @nullable @utf8InCpp String dexMetadata);
107  
108      long snapshotAppData(@nullable @utf8InCpp String uuid, in @utf8InCpp String packageName,
109              int userId, int snapshotId, int storageFlags);
110      void restoreAppDataSnapshot(@nullable @utf8InCpp String uuid, in @utf8InCpp String packageName,
111              int appId, @utf8InCpp String seInfo, int user, int snapshotId, int storageflags);
112      void destroyAppDataSnapshot(@nullable @utf8InCpp String uuid, @utf8InCpp String packageName,
113              int userId, long ceSnapshotInode, int snapshotId, int storageFlags);
114  
115      void migrateLegacyObbData();
116  
117      const int FLAG_STORAGE_DE = 0x1;
118      const int FLAG_STORAGE_CE = 0x2;
119      const int FLAG_STORAGE_EXTERNAL = 0x4;
120  
121      const int FLAG_CLEAR_CACHE_ONLY = 0x10;
122      const int FLAG_CLEAR_CODE_CACHE_ONLY = 0x20;
123  
124      const int FLAG_FREE_CACHE_V2 = 0x100;
125      const int FLAG_FREE_CACHE_V2_DEFY_QUOTA = 0x200;
126      const int FLAG_FREE_CACHE_NOOP = 0x400;
127  
128      const int FLAG_USE_QUOTA = 0x1000;
129      const int FLAG_FORCE = 0x2000;
130  }
131  
```

## 1.2 ActivityTaskManagerService

Android 10把 activities管理、启动相关的逻辑拆解到了 atm 中。

### 1.2.1 atm 构造方法

```
  public ActivityTaskManagerService(Context context) {
        mContext = context;
        mFactoryTest = FactoryTest.getMode();
        mSystemThread = ActivityThread.currentActivityThread(); //获取当前进程的ActivityThread对象
        mUiContext = mSystemThread.getSystemUiContext(); // systemUiContext，包含资源
        
        mLifecycleManager = new ClientLifecycleManager();// 重点：生命周期管理类。activity 就是在这里被管理的
        
        mInternal = new LocalService(); //继承ActivityTaskManagerInternal类，类似于本地服务。
        GL_ES_VERSION = SystemProperties.getInt("ro.opengles.version", GL_ES_VERSION_UNDEFINED);
    }
```

### 1.2.2 atm 的 onStart()

Lifecycle：

```
 @Override
        public void onStart() {
            // 往SM中注册 服务
            publishBinderService(Context.ACTIVITY_TASK_SERVICE, mService);
            // 往本地注册 服务
            mService.start();
        }
```

```
 protected final void publishBinderService(String name, IBinder service,
235              boolean allowIsolated, int dumpPriority) {
236          ServiceManager.addService(name, service, allowIsolated, dumpPriority);
237      }
238  
```

```
 private void start() {
        //class类型， mInternal： localService。
        LocalServices.addService(ActivityTaskManagerInternal.class, mInternal);
    }
```

LocalServices： 类似于SM的功能。不过这里注册的不是Binder对象，并且只能在同一个进程中使用。 一旦这个service强转为 systemServer接口对象，那么
LocalServices 就可以认为是类似于 SystemServerManager的功能了。

### 1.2.3 atm 的 initialize()

后续 AMS 的构造方法中会调用：

```
 public void initialize(IntentFirewall intentFirewall, PendingIntentController intentController,
            Looper looper) {
            // looper : DisplayThread的loop
        mH = new H(looper);
        mUiHandler = new UiHandler();
        mIntentFirewall = intentFirewall;
        final File systemDir = SystemServiceManager.ensureSystemDir();
        mAppWarnings = new AppWarnings(this, mUiContext, mH, mUiHandler, systemDir);
        mCompatModePackages = new CompatModePackages(this, systemDir, mH);
        mPendingIntentController = intentController;

        mTempConfig.setToDefaults();
        mTempConfig.setLocales(LocaleList.getDefault());
        mConfigurationSeq = mTempConfig.seq = 1;
        // 1 创建 ActivityStackSupervisor 对象 ,Activity栈的超级管理
        mStackSupervisor = createStackSupervisor();
        //暂时过渡的类，目的是与RootWindowContainer.java 关联起来
        mRootActivityContainer = new RootActivityContainer(this);
        mRootActivityContainer.onConfigurationChanged(mTempConfig);

        mTaskChangeNotificationController =
                new TaskChangeNotificationController(mGlobalLock, mStackSupervisor, mH);
        mLockTaskController = new LockTaskController(mContext, mStackSupervisor, mH);
        mActivityStartController = new ActivityStartController(this);
        mRecentTasks = createRecentTasks();
        //2 设置最近的任务栈 
        mStackSupervisor.setRecentTasks(mRecentTasks);
        mVrController = new VrController(mGlobalLock);
        mKeyguardController = mStackSupervisor.getKeyguardController();
    }
```

```
  protected ActivityStackSupervisor createStackSupervisor() {
        final ActivityStackSupervisor supervisor = new ActivityStackSupervisor(this, mH.getLooper());
        supervisor.initialize();
        return supervisor;
    }
```

## 1.3 SystemContext

## 1.3 SystemServerManager

## 1.4 ActivityManagerService

```
mActivityManagerService = ActivityManagerService.Lifecycle.startService(
        mSystemServiceManager, atm);
```

Lifecycle 是 ActivityManagerService的静态内部类。它继承了SystemServer类。

```
 public static final class Lifecycle extends SystemService {
        private final ActivityManagerService mService;
        private static ActivityTaskManagerService sAtm;

        public Lifecycle(Context context) {
            super(context);
            //2 startService() 过程中会反射调用这里
            mService = new ActivityManagerService(context, sAtm);
        }

        public static ActivityManagerService startService(
                SystemServiceManager ssm, ActivityTaskManagerService atm) {
            sAtm = atm;
             // 1 调用 SystemServiceManager startService来启动服务(会回调onStart方法)。返回当前Lifecycle对象
             // 在调用getService()，得到真正的ActivityManagerService 对象
            return ssm.startService(ActivityManagerService.Lifecycle.class).getService();
        }
        
        @Override
        public void onStart() {
            // 3 回调该方法 
            mService.start();
        }

        @Override
        public void onBootPhase(int phase) {
            mService.mBootPhase = phase;
            if (phase == PHASE_SYSTEM_SERVICES_READY) { 
                // 回调 systemServicesReady ？？
                mService.mBatteryStatsService.systemServicesReady();
                mService.mServices.systemServicesReady();
            } else if (phase == PHASE_ACTIVITY_MANAGER_READY) {
             // 回调 暂停Activity 消息 ？？
                mService.startBroadcastObservers();
            } else if (phase == PHASE_THIRD_PARTY_APPS_CAN_START) {
                 // 回调 第三方app 可以启动 ？？
                mService.mPackageWatchdog.onPackagesReady();
            }
        }

        @Override
        public void onCleanupUser(int userId) {
            mService.mBatteryStatsService.onCleanupUser(userId);
        }

        public ActivityManagerService getService() {
            // 4 返回一个 AMS 对象
            return mService;
        }
    }
```

1. 调用 SystemServiceManager startService()来启动服务 Lifecycle。返回当前Lifecycle对象。再调用getService()
   ，得到真正的ActivityManagerService 对象。

2. SystemServiceManager会通过反射得到 Lifecycle 对象，并把它加入到 mServices 列表中(用来管理所有服务的生命周期)， 同时创建AMS对象。

3. 各种生命周期回调。

### 1.4.1 SystemServiceManager.startService()

```
  public SystemService startService(String className) {
        final Class<SystemService> serviceClass;
        try {
            // 通过 服务名字 反射得到class 对象
            serviceClass = (Class<SystemService>)Class.forName(className);
        } catch (ClassNotFoundException ex) {
            Slog.i(TAG, "Starting " + className);
            throw new RuntimeException("Failed to create service " + className
                    + ": service class not found, usually indicates that the caller should "
                    + "have called PackageManager.hasSystemFeature() to check whether the "
                    + "feature is available on this device before trying to start the "
                    + "services that implement it", ex);
        }
        return startService(serviceClass);
    }
```

```
 public <T extends SystemService> T startService(Class<T> serviceClass) {
        try {
            final String name = serviceClass.getName();
            Slog.i(TAG, "Starting " + name);
            Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "StartService " + name);

            // Create the service.
            if (!SystemService.class.isAssignableFrom(serviceClass)) {
                throw new RuntimeException("Failed to create " + name
                        + ": service must extend " + SystemService.class.getName());
            }
            final T service;
            try {
                // 1 反射创建一个服务对象 
                Constructor<T> constructor = serviceClass.getConstructor(Context.class);
                service = constructor.newInstance(mContext);
            } catch (InstantiationException ex) {
                throw new RuntimeException("Failed to create service " + name
                        + ": service could not be instantiated", ex);
            } catch (IllegalAccessException ex) {
                throw new RuntimeException("Failed to create service " + name
                        + ": service must have a public constructor with a Context argument", ex);
            } catch (NoSuchMethodException ex) {
                throw new RuntimeException("Failed to create service " + name
                        + ": service must have a public constructor with a Context argument", ex);
            } catch (InvocationTargetException ex) {
                throw new RuntimeException("Failed to create service " + name
                        + ": service constructor threw an exception", ex);
            }
            2 启动新创建的服务 
            startService(service);
            return service;
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
        }
    }
```

```
   public void startService(@NonNull final SystemService service) {
        // Register it.
        // 1 注册 加入列表中 
        mServices.add(service);
        // Start it.
        long time = SystemClock.elapsedRealtime();
        try {
        // 2 回调onStart方法 ，表示服务已经启动。
            service.onStart();
        } catch (RuntimeException ex) {
            throw new RuntimeException("Failed to start service " + service.getClass().getName()
                    + ": onStart threw an exception", ex);
        }
        warnIfTooLong(SystemClock.elapsedRealtime() - time, service, "onStart");
    }
```

一套流程下来,我们知道： AMS 是通过内部类 Lifecycle 来管理生命周期的。

# 二、 AMS

## 2.1 AMS 继承关系

```
public class ActivityManagerService extends IActivityManager.Stub
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
```

- 继承了 IActivityManager.Stub ： 这个很合理，因为我们要提供IActivityManager接口能力
- 实现了 Watchdog.Monitor接口： Watchdog是一个线程，每一分钟会回调实现了Monitor接口的对象。如果这个进程没有返回，则杀掉这个进程。
- 实现BatteryStatsImpl.BatteryCallback接口： 电池生命周期事件监听

### 2.1.1 IActivityManager

提供了很多的方法： startActivity，startService，registerReceiver，getContentProvider。 具体参考：
> https://android.googlesource.com/platform/frameworks/base/+/33f5ddd/core/java/android/app/IActivityManager.java

### 2.1.2 Watchdog.Monitor

> Watchdog 是为了保护系统进程 SystemServer 能够正常运行而存在的。它覆盖了 AMS、PMS、DisplayThread、
> UiThread、AnimationThread、SurfaceAnimationThread、IoThread、SystemServer的主线程、BinderThreadMonitor
> 线程等监控。 一旦出现异常，则写入调用日志，同时重启 SystemServer 进程。

AMS的构造方法中有监听 本身对象和 mHandler。

```
 Watchdog.getInstance().addMonitor(this);
 Watchdog.getInstance().addThread(mHandler);
```

```
    /** In this method we try to acquire our lock to make sure that we have not deadlocked */
    public void monitor() {
        // 上市获取锁，如果拿不到锁，则认为AMS需要重启。
        synchronized (this) { }
    }
```

### 2.1.3 BatteryStatsImpl.BatteryCallback

```
 public interface BatteryCallback {
        public void batteryNeedsCpuUpdate();
        public void batteryPowerChanged(boolean onBattery);
        public void batterySendBroadcast(Intent intent);
        public void batteryStatsReset();
    }
```

## 2.2 AMS 构造方法

```
    // systemContext： systemServer的上下文。atm： ActivityTaskManagerService？？
 public ActivityManagerService(Context systemContext, ActivityTaskManagerService atm) {
        LockGuard.installLock(this, LockGuard.INDEX_ACTIVITY);
       
        mInjector = new Injector();
        mContext = systemContext; //上下文

        mFactoryTest = FactoryTest.getMode();
        mSystemThread = ActivityThread.currentActivityThread();
        //1 根据systemContext 创建一个用于UI的 context，包含资源。
        mUiContext = mSystemThread.getSystemUiContext();

        Slog.i(TAG, "Memory class: " + ActivityManager.staticGetMemoryClass());
        // 2 创建前台线程  名字：ActivityManagerService 
        mHandlerThread = new ServiceThread(TAG,
                THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
        mHandlerThread.start();
        // 3 创建线程 主线程的handler
        mHandler = new MainHandler(mHandlerThread.getLooper());
        mUiHandler = mInjector.getUiHandler(this);
        // 4 创建 ServiceThread 线程，名字：ActivityManagerService：procStart
        mProcStartHandlerThread = new ServiceThread(TAG + ":procStart",
                THREAD_PRIORITY_FOREGROUND, false /* allowIo */);
        mProcStartHandlerThread.start();
        mProcStartHandler = new Handler(mProcStartHandlerThread.getLooper());
        
        mConstants = new ActivityManagerConstants(mContext, this, mHandler);
        final ActiveUids activeUids = new ActiveUids(this, true /* postChangesToAtm */);
        mProcessList.init(this, activeUids);
        mLowMemDetector = new LowMemDetector(this);
        mOomAdjuster = new OomAdjuster(this, mProcessList, activeUids);

        // Broadcast policy parameters Broadcast广播策略
        final BroadcastConstants foreConstants = new BroadcastConstants(
                Settings.Global.BROADCAST_FG_CONSTANTS);
        foreConstants.TIMEOUT = BROADCAST_FG_TIMEOUT;

        final BroadcastConstants backConstants = new BroadcastConstants(
                Settings.Global.BROADCAST_BG_CONSTANTS);
        backConstants.TIMEOUT = BROADCAST_BG_TIMEOUT;

        final BroadcastConstants offloadConstants = new BroadcastConstants(
                Settings.Global.BROADCAST_OFFLOAD_CONSTANTS);
        offloadConstants.TIMEOUT = BROADCAST_BG_TIMEOUT;
        // by default, no "slow" policy in this queue
        offloadConstants.SLOW_TIME = Integer.MAX_VALUE;

        mEnableOffloadQueue = SystemProperties.getBoolean(
                "persist.device_config.activity_manager_native_boot.offload_queue_enabled", false);
        // 前台广播队列 处理超时时长是 10s
        mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "foreground", foreConstants, false);
        // 后台广播队列 处理超时时长是 60s
        mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "background", backConstants, true);
       // 分流广播队列，处理超时时长是 60s
        mOffloadBroadcastQueue = new BroadcastQueue(this, mHandler,
                "offload", offloadConstants, true);
        mBroadcastQueues[0] = mFgBroadcastQueue;
        mBroadcastQueues[1] = mBgBroadcastQueue;
        mBroadcastQueues[2] = mOffloadBroadcastQueue;
        // 四大组件的 service，管理 ServiceRecord
        mServices = new ActiveServices(this);
        // 四大组件 管理  ContentProviderRecord 对象 
        mProviderMap = new ProviderMap(this);
        mPackageWatchdog = PackageWatchdog.getInstance(mUiContext);
        mAppErrors = new AppErrors(mUiContext, this, mPackageWatchdog);

        final File systemDir = SystemServiceManager.ensureSystemDir();

        // TODO: Move creation of battery stats service outside of activity manager service.
        mBatteryStatsService = new BatteryStatsService(systemContext, systemDir,
                BackgroundThread.get().getHandler());
        mBatteryStatsService.getActiveStatistics().readLocked();
        mBatteryStatsService.scheduleWriteToDisk();
        mOnBattery = DEBUG_POWER ? true
                : mBatteryStatsService.getActiveStatistics().getIsOnBattery();
        mBatteryStatsService.getActiveStatistics().setCallback(this);
        mOomAdjProfiler.batteryPowerChanged(mOnBattery);

        mProcessStats = new ProcessStatsService(this, new File(systemDir, "procstats"));

        mAppOpsService = mInjector.getAppOpsService(new File(systemDir, "appops.xml"), mHandler);
        // uri授权service 
        mUgmInternal = LocalServices.getService(UriGrantsManagerInternal.class);

        mUserController = new UserController(this);
        // pendingIntent 控制器
        mPendingIntentController = new PendingIntentController(
                mHandlerThread.getLooper(), mUserController);

        if (SystemProperties.getInt("sys.use_fifo_ui", 0) != 0) {
            mUseFifoUiScheduling = true;
        }

        mTrackingAssociations = "1".equals(SystemProperties.get("debug.track-associations"));
        mIntentFirewall = new IntentFirewall(new IntentFirewallInterface(), mHandler);

        mActivityTaskManager = atm;
        // atm 初始化
        mActivityTaskManager.initialize(mIntentFirewall, mPendingIntentController,
                DisplayThread.get().getLooper());
                
                // 获取同进程的本地服务service 
        mAtmInternal = LocalServices.getService(ActivityTaskManagerInternal.class);
        // 创建线程 CpuTracker
        mProcessCpuThread = new Thread("CpuTracker") {
            @Override
            public void run() {
                synchronized (mProcessCpuTracker) {
                    mProcessCpuInitLatch.countDown();
                    mProcessCpuTracker.init();
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
                        updateCpuStatsNow();
                    } catch (Exception e) {
                        Slog.e(TAG, "Unexpected exception collecting process stats", e);
                    }
                }
            }
        };

        mHiddenApiBlacklist = new HiddenApiSettings(mHandler, mContext);
       // 添加 Watchdog 监听
        Watchdog.getInstance().addMonitor(this);
        Watchdog.getInstance().addThread(mHandler);

        // bind background threads to little cores
        // this is expected to fail inside of framework tests because apps can't touch cpusets directly
        // make sure we've already adjusted system_server's internal view of itself first
        updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_NONE);
        try {
            Process.setThreadGroupAndCpuset(BackgroundThread.get().getThreadId(),
                    Process.THREAD_GROUP_SYSTEM);
            Process.setThreadGroupAndCpuset(
                    mOomAdjuster.mAppCompact.mCompactionThread.getThreadId(),
                    Process.THREAD_GROUP_SYSTEM);
        } catch (Exception e) {
            Slog.w(TAG, "Setting background thread cpuset failed");
        }

    }
```

构造方法做了很多：

### 2.2.1 ActivityTaskManagerService

用来单独管理 activies以及他们的容器：task、stacks、displays、...等。

## 2.3 生命周期方法

### 2.3.1 start()

```
    private void start() {
        // 移除所有进程组 
        removeAllProcessGroups();
        // 开启CPU监控 线程
        mProcessCpuThread.start();

        mBatteryStatsService.publish();
        mAppOpsService.publish(mContext);
        Slog.d("AppOps", "AppOpsService published");
        LocalServices.addService(ActivityManagerInternal.class, new LocalService());
        mActivityTaskManager.onActivityManagerInternalAdded();
        mUgmInternal.onActivityManagerInternalAdded();
        mPendingIntentController.onActivityManagerInternalAdded();
        // Wait for the synchronized block started in mProcessCpuThread,
        // so that any other access to mProcessCpuTracker from main thread
        // will be blocked during mProcessCpuTracker initialization.
        try {
            mProcessCpuInitLatch.await();
        } catch (InterruptedException e) {
            Slog.wtf(TAG, "Interrupted wait during start", e);
            Thread.currentThread().interrupt();
            throw new IllegalStateException("Interrupted wait during start");
        }
    }
```

### 2.3.2 setSystemProcess()

向SM注册各种服务：

```
 public void setSystemProcess() {
        try {
            // 注册AMS
            ServiceManager.addService(Context.ACTIVITY_SERVICE, this, /* allowIsolated= */ true,
                    DUMP_FLAG_PRIORITY_CRITICAL | DUMP_FLAG_PRIORITY_NORMAL | DUMP_FLAG_PROTO);
            // 注册 ProcessStats 进程统计服务
            ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);
            // 注册 内存信息服务
            ServiceManager.addService("meminfo", new MemBinder(this), /* allowIsolated= */ false,
                    DUMP_FLAG_PRIORITY_HIGH);
            // 注册 图形相关服务
            ServiceManager.addService("gfxinfo", new GraphicsBinder(this));
            ServiceManager.addService("dbinfo", new DbBinder(this));
            if (MONITOR_CPU_USAGE) {
                ServiceManager.addService("cpuinfo", new CpuBinder(this),
                        /* allowIsolated= */ false, DUMP_FLAG_PRIORITY_CRITICAL);
            }
            // 注册 权限服务
            ServiceManager.addService("permission", new PermissionController(this));
            // 注册 processinfo 进程信息服务
            ServiceManager.addService("processinfo", new ProcessInfoService(this));
            // systemServer进程的 application
            ApplicationInfo info = mContext.getPackageManager().getApplicationInfo(
                    "android", STOCK_PM_FLAGS | MATCH_SYSTEM_ONLY);
            mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());

            synchronized (this) {
                ProcessRecord app = mProcessList.newProcessRecordLocked(info, info.processName,
                        false,
                        0,
                        new HostingRecord("system"));
                app.setPersistent(true);
                app.pid = MY_PID;
                app.getWindowProcessController().setPid(MY_PID);
                app.maxAdj = ProcessList.SYSTEM_ADJ;
                app.makeActive(mSystemThread.getApplicationThread(), mProcessStats);
                mPidsSelfLocked.put(app);
                mProcessList.updateLruProcessLocked(app, false, null);
                updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_NONE);
            }
        } catch (PackageManager.NameNotFoundException e) {
            throw new RuntimeException(
                    "Unable to find android system package", e);
        }

        // Start watching app ops after we and the package manager are up and running.
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

### 2.3.3  systemReady()

```
public void systemReady(final Runnable goingCallback, TimingsTraceLog traceLog) {
        traceLog.traceBegin("PhaseActivityManagerReady");
        synchronized(this) {
            //1 如果已经ready，则直接运行goingcallback
            if (mSystemReady) {
                // If we're done calling all the receivers, run the next "boot phase" passed in
                // by the SystemServer
                if (goingCallback != null) {
                    goingCallback.run();
                }
                return;
            }

            mLocalDeviceIdleController
                    = LocalServices.getService(DeviceIdleController.LocalService.class);
                    //2 调用一系列服务的 onSystemReady()
            mActivityTaskManager.onSystemReady();
            // Make sure we have the current profile info, since it is needed for security checks.
            mUserController.onSystemReady();
            mAppOpsService.systemReady();
            mSystemReady = true; // 修改ready状态
           
        }

        try {
            sTheRealBuildSerial = IDeviceIdentifiersPolicyService.Stub.asInterface(
                    ServiceManager.getService(Context.DEVICE_IDENTIFIERS_SERVICE))
                    .getSerial();
        } catch (RemoteException e) {}

        ArrayList<ProcessRecord> procsToKill = null;
        synchronized(mPidsSelfLocked) {
            //3  mPidsSelfLocked 保存了所以运行中的进程
            for (int i=mPidsSelfLocked.size()-1; i>=0; i--) {
                ProcessRecord proc = mPidsSelfLocked.valueAt(i);
                   // 在AMS启动完成前，且info.flag!=FLAG_PERSISTENT, 则加入到被杀的列表中
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
                    Slog.i(TAG, "Removing system update proc: " + proc);
                    // 开始杀掉列表中的进程
                    mProcessList.removeProcessLocked(proc, true, false, "system update done");
                }
            }

            // Now that we have cleaned up any update processes, we
            // are ready to start launching real processes and know that
            // we won't trample on them any more.
            mProcessesReady = true;
        }

        Slog.i(TAG, "System now ready");
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_AMS_READY, SystemClock.uptimeMillis());

        mAtmInternal.updateTopComponentForFactoryTest();
        mAtmInternal.getLaunchObserverRegistry().registerLaunchObserver(mActivityLaunchObserver);

        watchDeviceProvisioning(mContext);

        retrieveSettings();
        mUgmInternal.onSystemReady();

        final PowerManagerInternal pmi = LocalServices.getService(PowerManagerInternal.class);
        if (pmi != null) {
            pmi.registerLowPowerModeObserver(ServiceType.FORCE_BACKGROUND_CHECK,
                    state -> updateForceBackgroundCheck(state.batterySaverEnabled));
            updateForceBackgroundCheck(
                    pmi.getLowPowerState(ServiceType.FORCE_BACKGROUND_CHECK).batterySaverEnabled);
        } else {
            Slog.wtf(TAG, "PowerManagerInternal not found.");
        }
        // 运行 systemServer传递过来的 callback。用来通知外部服务systemReady 工作，并启动服务或者应用进程。
        if (goingCallback != null) goingCallback.run();
        
        // Check the current user here as a user can be started inside goingCallback.run() from
        // other system services.
        final int currentUserId = mUserController.getCurrentUserId();
        Slog.i(TAG, "Current user:" + currentUserId);
        if (currentUserId != UserHandle.USER_SYSTEM && !mUserController.isSystemUserStarted()) {
            // User other than system user has started. Make sure that system user is already
            // started before switching user.
            throw new RuntimeException("System user not started while current user is:"
                    + currentUserId);
        }
        traceLog.traceBegin("ActivityManagerStartApps");
        mBatteryStatsService.noteEvent(BatteryStats.HistoryItem.EVENT_USER_RUNNING_START,
                Integer.toString(currentUserId), currentUserId);
        mBatteryStatsService.noteEvent(BatteryStats.HistoryItem.EVENT_USER_FOREGROUND_START,
                Integer.toString(currentUserId), currentUserId);
          //调用所有系统服务的onStartUser接口 
        mSystemServiceManager.startUser(currentUserId);

        synchronized (this) {
            // Only start up encryption-aware persistent apps; once user is
            // unlocked we'll come back around and start unaware apps
            //启动persistent为1的application所在的进程
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
            
            //启动Home Activity
            mAtmInternal.startHomeOnAllDisplays(currentUserId, "systemReady");

            mAtmInternal.showSystemReadyErrorDialogsIfNeeded();

            final int callingUid = Binder.getCallingUid();
            final int callingPid = Binder.getCallingPid();
            long ident = Binder.clearCallingIdentity();
            try {
                // system发送广播 ACTION_USER_STARTED = "android.intent.action.USER_STARTED"; 
                Intent intent = new Intent(Intent.ACTION_USER_STARTED);
                intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY
                        | Intent.FLAG_RECEIVER_FOREGROUND);
                intent.putExtra(Intent.EXTRA_USER_HANDLE, currentUserId);
                broadcastIntentLocked(null, null, intent,
                        null, null, 0, null, null, null, OP_NONE,
                        null, false, false, MY_PID, SYSTEM_UID, callingUid, callingPid,
                        currentUserId);
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
                        null, true, false, MY_PID, SYSTEM_UID, callingUid, callingPid,
                        UserHandle.USER_ALL);
            } catch (Throwable t) {
                Slog.wtf(TAG, "Failed sending first user broadcasts", t);
            } finally {
                Binder.restoreCallingIdentity(ident);
            }
            mAtmInternal.resumeTopActivities(false /* scheduleIdle */);
            mUserController.sendUserSwitchBroadcasts(-1, currentUserId);

            BinderInternal.nSetBinderProxyCountWatermarks(BINDER_PROXY_HIGH_WATERMARK,
                    BINDER_PROXY_LOW_WATERMARK);
            BinderInternal.nSetBinderProxyCountEnabled(true);
            BinderInternal.setBinderProxyCountCallback(
                    new BinderInternal.BinderProxyLimitListener() {
                        @Override
                        public void onLimitReached(int uid) {
                            Slog.wtf(TAG, "Uid " + uid + " sent too many Binders to uid "
                                    + Process.myUid());
                            BinderProxy.dumpProxyDebugInfo();
                            if (uid == Process.SYSTEM_UID) {
                                Slog.i(TAG, "Skipping kill (uid is SYSTEM)");
                            } else {
                                killUid(UserHandle.getAppId(uid), UserHandle.getUserId(uid),
                                        "Too many Binders sent to SYSTEM");
                            }
                        }
                    }, mHandler);

            traceLog.traceEnd(); // ActivityManagerStartApps
            traceLog.traceEnd(); // PhaseActivityManagerReady
        }
    }
```

### 2.3.4  systemReady()

callback ：

```
mActivityManagerService.systemReady(() -> {
            Slog.i(TAG, "Making services ready");
            traceBeginAndSlog("StartActivityManagerReadyPhase");
            // startBootPhase
            mSystemServiceManager.startBootPhase(
                    SystemService.PHASE_ACTIVITY_MANAGER_READY);
            traceEnd();
            traceBeginAndSlog("StartObservingNativeCrashes");
            // native crash 监听
            try {
                mActivityManagerService.startObservingNativeCrashes();
            } catch (Throwable e) {
                reportWtf("observing native crashes", e);
            }
            traceEnd();

            // No dependency on Webview preparation in system server. But this should
            // be completed before allowing 3rd party
            final String WEBVIEW_PREPARATION = "WebViewFactoryPreparation";
            // 为把view 准备
            Future<?> webviewPrep = null;
            if (!mOnlyCore && mWebViewUpdateService != null) {
                webviewPrep = SystemServerInitThreadPool.get().submit(() -> {
                    Slog.i(TAG, WEBVIEW_PREPARATION);
                    TimingsTraceLog traceLog = new TimingsTraceLog(
                            SYSTEM_SERVER_TIMING_ASYNC_TAG, Trace.TRACE_TAG_SYSTEM_SERVER);
                    traceLog.traceBegin(WEBVIEW_PREPARATION);
                    ConcurrentUtils.waitForFutureNoInterrupt(mZygotePreload, "Zygote preload");
                    mZygotePreload = null;
                    mWebViewUpdateService.prepareWebViewInSystemServer();
                    traceLog.traceEnd();
                }, WEBVIEW_PREPARATION);
            }

            if (mPackageManager.hasSystemFeature(PackageManager.FEATURE_AUTOMOTIVE)) {
                traceBeginAndSlog("StartCarServiceHelperService");
                mSystemServiceManager.startService(CAR_SERVICE_HELPER_SERVICE_CLASS);
                traceEnd();
            }

            traceBeginAndSlog("StartSystemUI");
            try {
            // 系统ui启动
                startSystemUi(context, windowManagerF);
            } catch (Throwable e) {
                reportWtf("starting System UI", e);
            }
            traceEnd();
            // Enable airplane mode in safe mode. setAirplaneMode() cannot be called
            // earlier as it sends broadcasts to other services.
            // TODO: This may actually be too late if radio firmware already started leaking
            // RF before the respective services start. However, fixing this requires changes
            // to radio firmware and interfaces.
            if (safeMode) {
                traceBeginAndSlog("EnableAirplaneModeInSafeMode");
                try {
                    connectivityF.setAirplaneMode(true);
                } catch (Throwable e) {
                    reportWtf("enabling Airplane Mode during Safe Mode bootup", e);
                }
                traceEnd();
            }
            traceBeginAndSlog("MakeNetworkManagementServiceReady");
            try {
                if (networkManagementF != null) {
                    // 网络管理  systemReady
                    networkManagementF.systemReady();
                }
            } catch (Throwable e) {
                reportWtf("making Network Managment Service ready", e);
            }
            CountDownLatch networkPolicyInitReadySignal = null;
            if (networkPolicyF != null) {
                networkPolicyInitReadySignal = networkPolicyF
                        .networkScoreAndNetworkManagementServiceReady();
            }
            traceEnd();
            traceBeginAndSlog("MakeIpSecServiceReady");
            try {
                if (ipSecServiceF != null) {
                    ipSecServiceF.systemReady();
                }
            } catch (Throwable e) {
                reportWtf("making IpSec Service ready", e);
            }
            traceEnd();
            traceBeginAndSlog("MakeNetworkStatsServiceReady");
            try {
                if (networkStatsF != null) {
                    networkStatsF.systemReady();
                }
            } catch (Throwable e) {
                reportWtf("making Network Stats Service ready", e);
            }
            traceEnd();
            traceBeginAndSlog("MakeConnectivityServiceReady");
            try {
                if (connectivityF != null) {
                    connectivityF.systemReady();
                }
            } catch (Throwable e) {
                reportWtf("making Connectivity Service ready", e);
            }
            traceEnd();
            traceBeginAndSlog("MakeNetworkPolicyServiceReady");
            try {
                if (networkPolicyF != null) {
                    networkPolicyF.systemReady(networkPolicyInitReadySignal);
                }
            } catch (Throwable e) {
                reportWtf("making Network Policy Service ready", e);
            }
            traceEnd();

            // Wait for all packages to be prepared
            mPackageManagerService.waitForAppDataPrepared();

            // It is now okay to let the various system services start their
            // third party code...
            traceBeginAndSlog("PhaseThirdPartyAppsCanStart");
            // confirm webview completion before starting 3rd party
            // 再次确保为把view
            if (webviewPrep != null) {
                ConcurrentUtils.waitForFutureNoInterrupt(webviewPrep, WEBVIEW_PREPARATION);
            }
            mSystemServiceManager.startBootPhase(
                    SystemService.PHASE_THIRD_PARTY_APPS_CAN_START);
            traceEnd();

            traceBeginAndSlog("StartNetworkStack");
            try {
                // Note : the network stack is creating on-demand objects that need to send
                // broadcasts, which means it currently depends on being started after
                // ActivityManagerService.mSystemReady and ActivityManagerService.mProcessesReady
                // are set to true. Be careful if moving this to a different place in the
                // startup sequence.
                NetworkStackClient.getInstance().start(context);
            } catch (Throwable e) {
                reportWtf("starting Network Stack", e);
            }
            traceEnd();

            traceBeginAndSlog("MakeLocationServiceReady");
            try {
                if (locationF != null) {
                    locationF.systemRunning();
                }
            } catch (Throwable e) {
                reportWtf("Notifying Location Service running", e);
            }
            traceEnd();
            traceBeginAndSlog("MakeCountryDetectionServiceReady");
            try {
                if (countryDetectorF != null) {
                    countryDetectorF.systemRunning();
                }
            } catch (Throwable e) {
                reportWtf("Notifying CountryDetectorService running", e);
            }
            traceEnd();
            traceBeginAndSlog("MakeNetworkTimeUpdateReady");
            try {
                if (networkTimeUpdaterF != null) {
                    networkTimeUpdaterF.systemRunning();
                }
            } catch (Throwable e) {
                reportWtf("Notifying NetworkTimeService running", e);
            }
            traceEnd();
            traceBeginAndSlog("MakeInputManagerServiceReady");
            try {
                // TODO(BT) Pass parameter to input manager
                if (inputManagerF != null) {
                // inputmanager 的 systemRunning()
                    inputManagerF.systemRunning();
                }
            } catch (Throwable e) {
                reportWtf("Notifying InputManagerService running", e);
            }
            traceEnd();
            traceBeginAndSlog("MakeTelephonyRegistryReady");
            try {
                if (telephonyRegistryF != null) {
                    telephonyRegistryF.systemRunning();
                }
            } catch (Throwable e) {
                reportWtf("Notifying TelephonyRegistry running", e);
            }
            traceEnd();
            traceBeginAndSlog("MakeMediaRouterServiceReady");
            try {
                if (mediaRouterF != null) {
                //
                    mediaRouterF.systemRunning();
                }
            } catch (Throwable e) {
                reportWtf("Notifying MediaRouterService running", e);
            }
            traceEnd();
            traceBeginAndSlog("MakeMmsServiceReady");
            try {
                if (mmsServiceF != null) {
                    mmsServiceF.systemRunning();
                }
            } catch (Throwable e) {
                reportWtf("Notifying MmsService running", e);
            }
            traceEnd();

            traceBeginAndSlog("IncidentDaemonReady");
            try {
                // TODO: Switch from checkService to getService once it's always
                // in the build and should reliably be there.
                final IIncidentManager incident = IIncidentManager.Stub.asInterface(
                        ServiceManager.getService(Context.INCIDENT_SERVICE));
                if (incident != null) {
                    incident.systemRunning();
                }
            } catch (Throwable e) {
                reportWtf("Notifying incident daemon running", e);
            }
            traceEnd();
        }, BOOT_TIMINGS_TRACE_LOG);
    }

```

# 参考

gityuan 设计思想 视频































