# PKMS

是对apk的扫描、安装、卸载、解析等操作。

疑问： apk安装过程是怎么样的？？

apk 卸载过程是怎么样的？

# 一、启动时机

> SystemServer.java:

```
private void startBootstrapServices() {
    ...
    
    traceBeginAndSlog("StartPackageManagerService");
        try {
            Watchdog.getInstance().pauseWatchingCurrentThread("packagemanagermain");
            //mSystemContext: 系统的上下文, install:安装apk时的服务  mOnlyCore=false
            mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                    mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
        } finally {
            Watchdog.getInstance().resumeWatchingCurrentThread("packagemanagermain");
        }
        mFirstBoot = mPackageManagerService.isFirstBoot();
        //得到 pm对象 
        mPackageManager = mSystemContext.getPackageManager();
        traceEnd();
        ...
}

```

mFactoryTestMode有三种模式，一般都是非工厂模式，即关闭状态。 疑问： 有什么用？

```
FACTORY_TEST_OFF, FACTORY_TEST_LOW_LEVEL, or FACTORY_TEST_HIGH_LEVEL.
```

# 二、 main()

> PackageManagerService.java

```

 public static PackageManagerService main(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
        // Self-check for initial settings. 自检
        PackageManagerServiceCompilerMapping.checkProperties();
        // 创建 PMS对象。
        PackageManagerService m = new PackageManagerService(context, installer,
                factoryTest, onlyCore);
        m.enableSystemUserPackages();
        // 往SM注册服务，名字为：package，服务：m
        ServiceManager.addService("package", m);
        //  创建 PackageManagerNative 服务，注册到SM 
        final PackageManagerNative pmn = m.new PackageManagerNative();
       // 往SM注册服务，名字为：package_native，服务：pmn
        ServiceManager.addService("package_native", pmn);
        return m;
    }
```

1. 创建PKMS对象，构造方法中传入了installer对象。 onlycore表示是否启动核心应用。如，在由密码的情况下我们只启动核心服务。
2. 往SM中注册 package服务，服务对象为 PKMS。
3. 往SM中注册 package_native 服务。对象为 PackageManagerNative

疑问： PackageManagerNative 是PMS的内部类。IPackageManagerNative是啥？

答： Parallel implementation of certain {@link PackageManager} APIs that need to be exposed to native
code. 翻译： 跟PM类的接似的实现。这些方法是提供给native调用的。 虽然和PM的api很像，但是还是保持在两个不同的地方把。

```
PackageManagerNative extends IPackageManagerNative.Stub{
```

PMS 还有 一个内部类 PackageManagerInternalImpl 。PackageManagerInternal是啥？

```
PackageManagerInternalImpl extends PackageManagerInternal 
```

猜测，应该是PMS在本进程对外暴露的接口类？

# 三、PMS的 构造方法

构造方法很长，根据log打印可以分为5 个阶段：

EventLogTags.BOOT_PROGRESS_PMS_START： 启动阶段 EventLogTags.BOOT_PROGRESS_PMS_SYSTEM_SCAN_START：系统扫描阶段
EventLogTags.BOOT_PROGRESS_PMS_DATA_SCAN_START： data扫描阶段 EventLogTags.BOOT_PROGRESS_PMS_SCAN_END：
扫描结束阶段 EventLogTags.BOOT_PROGRESS_PMS_READY： 准备阶段

## 3.1 启动阶段

DisplayMetrics： 保存屏幕信息。

```
public PackageManagerService(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
        LockGuard.installLock(mPackages, LockGuard.INDEX_PACKAGES);
        // 
        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "create package manager");
        // 日志追踪
        EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_START,
                SystemClock.uptimeMillis());

        if (mSdkVersion <= 0) {
            Slog.w(TAG, "**** ro.build.version.sdk not set!");
        }

        mContext = context;

        mFactoryTest = factoryTest;
        // 
        mOnlyCore = onlyCore;
        // 1 存储屏幕相关信息
        mMetrics = new DisplayMetrics();
        // 安装服务。本身是 installd 进程的封装。真正是通过binder通信来实现
        mInstaller = installer;

        // Create sub-components that provide services / data. Order here is important.
        // 2 进入同步锁 mInstallLock，安装apk时需要的锁
        synchronized (mInstallLock) { 
        // 3 进入同步锁 mPackages  // 更新apk时候需要的锁 
        synchronized (mPackages) {
            // Expose private service for system components to use.
            // 4，注册 PMInternalImpl 到LocalServices中，对外暴露PkMS的接口，方便本进程系统组件使用
            LocalServices.addService(
                    PackageManagerInternal.class, new PackageManagerInternalImpl());
                    // 5. 用户管理服务
            sUserManager = new UserManagerService(context, this,
                    new UserDataPreparer(mInstaller, mInstallLock, mContext, mOnlyCore), mPackages);
                    // 6. 组件解析器。 传入了 userManager、PKMS对外暴露的内部类对象、mPackages。
            mComponentResolver = new ComponentResolver(sUserManager,
                    LocalServices.getService(PackageManagerInternal.class),
                    mPackages);
                    // 7, 权限管理服务
            mPermissionManager = PermissionManagerService.create(context,
                    mPackages /*externalLock*/);
                    // 8，默认的权限的策略
            mDefaultPermissionPolicy = mPermissionManager.getDefaultPermissionGrantPolicy();
                    // 9，Settings 保存了所有包的信息结构 
            mSettings = new Settings(Environment.getDataDirectory(),
                    mPermissionManager.getPermissionSettings(), mPackages);
        }
        } // 退出同步锁 mInstallLock  mPackages
        
        // 10, 把system、phone、log、nfc、bluetooth、shell、se、networkstack 8种 shareUserId 作为默认的 
        sharedUid 加入到 mSettings，
        mSettings.addSharedUserLPw("android.uid.system", Process.SYSTEM_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.phone", RADIO_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.log", LOG_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.nfc", NFC_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.bluetooth", BLUETOOTH_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.shell", SHELL_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.se", SE_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);
        mSettings.addSharedUserLPw("android.uid.networkstack", NETWORKSTACK_UID,
                ApplicationInfo.FLAG_SYSTEM, ApplicationInfo.PRIVATE_FLAG_PRIVILEGED);

        String separateProcesses = SystemProperties.get("debug.separate_processes");
        if (separateProcesses != null && separateProcesses.length() > 0) {
            if ("*".equals(separateProcesses)) {
                mDefParseFlags = PackageParser.PARSE_IGNORE_PROCESSES;
                mSeparateProcesses = null;
                Slog.w(TAG, "Running with debug.separate_processes: * (ALL)");
            } else {
                mDefParseFlags = 0;
                mSeparateProcesses = separateProcesses.split(",");
                Slog.w(TAG, "Running with debug.separate_processes: "
                        + separateProcesses);
            }
        } else {
            mDefParseFlags = 0;
            mSeparateProcesses = null;
        }
         // 11 包优化器 
        mPackageDexOptimizer = new PackageDexOptimizer(installer, mInstallLock, context,
                "*dexopt*");
         // 12 dex管理者,追踪dex文件的过程
        mDexManager = new DexManager(mContext, this, mPackageDexOptimizer, installer, mInstallLock);
        // 13 art管理服务 
        mArtManagerService = new ArtManagerService(mContext, this, installer, mInstallLock);
        // 14 android.fg线程 的handler
        mMoveCallbacks = new MoveCallbacks(FgThread.get().getLooper());

        mViewCompiler = new ViewCompiler(mInstallLock, mInstaller);

        mOnPermissionChangeListeners = new OnPermissionChangeListeners(
                FgThread.get().getLooper());

        // 获取屏幕信息 
        getDefaultDisplayMetrics(context, mMetrics);

        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "get system config");
        // 开始获取系统配置
        SystemConfig systemConfig = SystemConfig.getInstance();
        mAvailableFeatures = systemConfig.getAvailableFeatures();
        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        // 受保护的包？
        mProtectedPackages = new ProtectedPackages(mContext);
        // apex 管理者 
        mApexManager = new ApexManager(context);
        // 进入同步锁 mInstallLock 、mPackages
        synchronized (mInstallLock) {
        // writer
        synchronized (mPackages) {
            // 创建并启动后台线程
            mHandlerThread = new ServiceThread(TAG,
                    Process.THREAD_PRIORITY_BACKGROUND, true /*allowIo*/);
            mHandlerThread.start();
             // 创建 mHandler.
            mHandler = new PackageHandler(mHandlerThread.getLooper());
            mProcessLoggingHandler = new ProcessLoggingHandler();
            // Watchdog监测这个 packageHandler 线程
            Watchdog.getInstance().addThread(mHandler, WATCHDOG_TIMEOUT);
            // InstantAppRegistry
            mInstantAppRegistry = new InstantAppRegistry(this);

            // 公共库 
            ArrayMap<String, SystemConfig.SharedLibraryEntry> libConfig
                    = systemConfig.getSharedLibraries();
            final int builtInLibCount = libConfig.size();
            for (int i = 0; i < builtInLibCount; i++) {
                String name = libConfig.keyAt(i);
                SystemConfig.SharedLibraryEntry entry = libConfig.valueAt(i);
                // 遍历共享库，加入到内嵌的lib中 ？？
                addBuiltInSharedLibraryLocked(entry.filename, name);
            }

            // Now that we have added all the libraries, iterate again to add dependency
            // information IFF their dependencies are added.
            long undefinedVersion = SharedLibraryInfo.VERSION_UNDEFINED;
            // 再次遍历内嵌的公共库，添加依赖
            for (int i = 0; i < builtInLibCount; i++) {
                String name = libConfig.keyAt(i);
                SystemConfig.SharedLibraryEntry entry = libConfig.valueAt(i);
                final int dependencyCount = entry.dependencies.length;
                for (int j = 0; j < dependencyCount; j++) {
                    final SharedLibraryInfo dependency =
                        getSharedLibraryInfoLPr(entry.dependencies[j], undefinedVersion);
                    if (dependency != null) {
                        getSharedLibraryInfoLPr(name, undefinedVersion).addDependency(dependency);
                    }
                }
            }

            SELinuxMMAC.readInstallPolicy();

            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "loadFallbacks");
            FallbackCategoryProvider.loadFallbacks();
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);

            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "read user settings");
            // mSettings 解析 data/system/ 目录中的 packages.xml 等文件
            mFirstBoot = !mSettings.readLPw(sUserManager.getUsers(false));
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);

            // Clean up orphaned packages for which the code path doesn't exist
            // and they are an update to a system app - caused by bug/32321269
            // 遍历所有安装的包 mPackages，移除无效包
            final int packageSettingCount = mSettings.mPackages.size();
            for (int i = packageSettingCount - 1; i >= 0; i--) {
                PackageSetting ps = mSettings.mPackages.valueAt(i);
                if (!isExternal(ps) && (ps.codePath == null || !ps.codePath.exists())
                        && mSettings.getDisabledSystemPkgLPr(ps.name) != null) {
                    mSettings.mPackages.removeAt(i);
                    mSettings.enableSystemPackageLPw(ps.name);
                }
            }
            // 第一次启动且没有加密，则把system部分拷贝到data中去。实际init进程已经完成了拷贝，这里只是确认下。
            if (!mOnlyCore && mFirstBoot) {
                requestCopyPreoptedFiles();
            }

            String customResolverActivityName = Resources.getSystem().getString(
                    R.string.config_customResolverActivity);
            if (!TextUtils.isEmpty(customResolverActivityName)) {
                mCustomResolverComponentName = ComponentName.unflattenFromString(
                        customResolverActivityName);
            }
            
        } // 退出锁 
            
```

总结：

1. 初始化了 多用户管理服务、权限管理服务、art管理服务、Settings设置。 Settings中包含了所有包的信息。
2. 把system、phone、log、nfc、bluetooth、shell、se、networkstack 8种 shareUserId加入到 mSettings。因为
   两个apk如果设置了相同的SharedUid则可以可以共享数据。
3. 初始化 PackageDexOptimizer和DexManager 用来做dex优化类、dex管理者、
4. 获取屏幕信息 DisplayMetrics
5. 获取系统全局配置 SystemConfig，完成权限的读取。添加共享的公共库
6. 创建 PackageManager 线程，同时绑定PackageHandler。Watchdog 检测该线程，防止由于工作太多导致无响应
7. mSettings 解析 data/system/ 目录中的 packages.xml 等文件。packages.xml
   保存了所有安装的包的信息，包含fingerprint指纹、签名、权限等基本信息。 如果 readLPw()方法返回true，mFirsBoot则为false，表示已经初始化过了。
8. 移除无效的包、继续初始化

### 3.1.1 创建 Settings

```
  Settings(File dataDir, PermissionSettings permission,
            Object lock) {
        mLock = lock;
        mPermissions = permission;
        mRuntimePermissionsPersistence = new RuntimePermissionPersistence(mLock);

        mSystemDir = new File(dataDir, "system");
        //创建 /data/system/ 目录 
        mSystemDir.mkdirs();
        FileUtils.setPermissions(mSystemDir.toString(),
                FileUtils.S_IRWXU|FileUtils.S_IRWXG
                |FileUtils.S_IROTH|FileUtils.S_IXOTH,
                -1, -1);
        mSettingsFilename = new File(mSystemDir, "packages.xml");
        mBackupSettingsFilename = new File(mSystemDir, "packages-backup.xml");
        mPackageListFilename = new File(mSystemDir, "packages.list");
        FileUtils.setPermissions(mPackageListFilename, 0640, SYSTEM_UID, PACKAGE_INFO_GID);

        final File kernelDir = new File("/config/sdcardfs");
        mKernelMappingFilename = kernelDir.exists() ? kernelDir : null;

        // Deprecated: Needed for migration
        mStoppedPackagesFilename = new File(mSystemDir, "packages-stopped.xml");
        mBackupStoppedPackagesFilename = new File(mSystemDir, "packages-stopped-backup.xml");
    }
```

### 3.1.2 SystemConfig.getInstance()

```
 SystemConfig() {
        // Read configuration from system
        readPermissions(Environment.buildPath(
                Environment.getRootDirectory(), "etc", "sysconfig"), ALLOW_ALL);

        // Read configuration from the old permissions dir
        readPermissions(Environment.buildPath(
                Environment.getRootDirectory(), "etc", "permissions"), ALLOW_ALL);

        // Vendors are only allowed to customize these
        int vendorPermissionFlag = ALLOW_LIBS | ALLOW_FEATURES | ALLOW_PRIVAPP_PERMISSIONS
                | ALLOW_ASSOCIATIONS;
        if (Build.VERSION.FIRST_SDK_INT <= Build.VERSION_CODES.O_MR1) {
            // For backward compatibility
            vendorPermissionFlag |= (ALLOW_PERMISSIONS | ALLOW_APP_CONFIGS);
        }
        readPermissions(Environment.buildPath(
                Environment.getVendorDirectory(), "etc", "sysconfig"), vendorPermissionFlag);
        readPermissions(Environment.buildPath(
                Environment.getVendorDirectory(), "etc", "permissions"), vendorPermissionFlag);

        // Allow ODM to customize system configs as much as Vendor, because /odm is another
        // vendor partition other than /vendor.
        int odmPermissionFlag = vendorPermissionFlag;
        readPermissions(Environment.buildPath(
                Environment.getOdmDirectory(), "etc", "sysconfig"), odmPermissionFlag);
        readPermissions(Environment.buildPath(
                Environment.getOdmDirectory(), "etc", "permissions"), odmPermissionFlag);

        String skuProperty = SystemProperties.get(SKU_PROPERTY, "");
        if (!skuProperty.isEmpty()) {
            String skuDir = "sku_" + skuProperty;

            readPermissions(Environment.buildPath(
                    Environment.getOdmDirectory(), "etc", "sysconfig", skuDir), odmPermissionFlag);
            readPermissions(Environment.buildPath(
                    Environment.getOdmDirectory(), "etc", "permissions", skuDir),
                    odmPermissionFlag);
        }

        // Allow OEM to customize these
        int oemPermissionFlag = ALLOW_FEATURES | ALLOW_OEM_PERMISSIONS | ALLOW_ASSOCIATIONS;
        readPermissions(Environment.buildPath(
                Environment.getOemDirectory(), "etc", "sysconfig"), oemPermissionFlag);
        readPermissions(Environment.buildPath(
                Environment.getOemDirectory(), "etc", "permissions"), oemPermissionFlag);

        // Allow Product to customize all system configs
        readPermissions(Environment.buildPath(
                Environment.getProductDirectory(), "etc", "sysconfig"), ALLOW_ALL);
        readPermissions(Environment.buildPath(
                Environment.getProductDirectory(), "etc", "permissions"), ALLOW_ALL);

        // Allow /product_services to customize all system configs
        readPermissions(Environment.buildPath(
                Environment.getProductServicesDirectory(), "etc", "sysconfig"), ALLOW_ALL);
        readPermissions(Environment.buildPath(
                Environment.getProductServicesDirectory(), "etc", "permissions"), ALLOW_ALL);
    }
```

读取指定目录的权限。

在PKMS 做了初始化后，那么就开始真正的扫描和安装阶段的工作了。 首先扫描的位置 system/ 目录。

## 3.2 system扫描阶段 SYSTEM_SCAN_START

```
 long startTime = SystemClock.uptimeMillis();

            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SYSTEM_SCAN_START,
                    startTime);
            // 获取 bootClass 的路径  如 system/framework 目录下的 content.jar、framework.jar、input.jar等。
            final String bootClassPath = System.getenv("BOOTCLASSPATH");
            // 主要包括 system/framework/ 目录下的 services.jar、ethernet-service.jar、wifi-services.jar三个文件
            final String systemServerClassPath = System.getenv("SYSTEMSERVERCLASSPATH");
            ...
            //  /system/framework 目录
            File frameworkDir = new File(Environment.getRootDirectory(), "framework");

            final VersionInfo ver = mSettings.getInternalVersion();
            // 版本指纹不一样则更新
            mIsUpgrade = !Build.FINGERPRINT.equals(ver.fingerprint);
            if (mIsUpgrade) {
                logCriticalInfo(Log.INFO,
                        "Upgrading from " + ver.fingerprint + " to " + Build.FINGERPRINT);
            }

            // when upgrading from pre-M, promote system app permissions from install to runtime
            // 在 Android6.0 之前升级的版本，则调整动态权限
            mPromoteSystemApps =
                    mIsUpgrade && ver.sdkVersion <= Build.VERSION_CODES.LOLLIPOP_MR1;

            // When upgrading from pre-N, we need to handle package extraction like first boot,
            // as there is no profiling data available.
            mIsPreNUpgrade = mIsUpgrade && ver.sdkVersion < Build.VERSION_CODES.N;

            mIsPreNMR1Upgrade = mIsUpgrade && ver.sdkVersion < Build.VERSION_CODES.N_MR1;
            mIsPreQUpgrade = mIsUpgrade && ver.sdkVersion < Build.VERSION_CODES.Q;

            int preUpgradeSdkVersion = ver.sdkVersion;

            // save off the names of pre-existing system packages prior to scanning; we don't
            // want to automatically grant runtime permissions for new system apps
            if (mPromoteSystemApps) {
                Iterator<PackageSetting> pkgSettingIter = mSettings.mPackages.values().iterator();
                while (pkgSettingIter.hasNext()) {
                    PackageSetting ps = pkgSettingIter.next();
                    if (isSystemApp(ps)) {
                        mExistingSystemPackages.add(ps.name);
                    }
                }
            }

            ...
            
            // Collect vendor/product/product_services overlay packages. (Do this before scanning
            // any apps.)
            // For security and version matching reason, only consider overlay packages if they
            // reside in the right directory.
            // 扫描 /vendor/overlay 目录下的文件
            scanDirTracedLI(new File(VENDOR_OVERLAY_DIR),
                    mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM_DIR,
                    scanFlags
                    | SCAN_AS_SYSTEM
                    | SCAN_AS_VENDOR,
                    0);
             // 扫描 /product/overlay 目录下的文件                 
            scanDirTracedLI(new File(PRODUCT_OVERLAY_DIR),
                    mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM_DIR,
                    scanFlags
                    | SCAN_AS_SYSTEM
                    | SCAN_AS_PRODUCT,
                    0);
             // 扫描 /product_services/overlay 目录下的文件                       
            scanDirTracedLI(new File(PRODUCT_SERVICES_OVERLAY_DIR),
                    mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM_DIR,
                    scanFlags
                    | SCAN_AS_SYSTEM
                    | SCAN_AS_PRODUCT_SERVICES,
                    0);
             // 扫描 /odm/overlay 目录下的文件                       
            scanDirTracedLI(new File(ODM_OVERLAY_DIR),
                    mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM_DIR,
                    scanFlags
                    | SCAN_AS_SYSTEM
                    | SCAN_AS_ODM,
                    0);
             // 扫描 /oem/overlay 目录下的文件                       
            scanDirTracedLI(new File(OEM_OVERLAY_DIR),
                    mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM_DIR,
                    scanFlags
                    | SCAN_AS_SYSTEM
                    | SCAN_AS_OEM,
                    0);

            mParallelPackageParserCallback.findStaticOverlayPackages();

            // Find base frameworks (resource packages without code).
            // 在 system/framework 目录下查找资源包，不包含代码
            scanDirTracedLI(frameworkDir,
                    mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM_DIR,
                    scanFlags
                    | SCAN_NO_DEX
                    | SCAN_AS_SYSTEM
                    | SCAN_AS_PRIVILEGED,
                    0);
            // 其实为了找framework.res.apk         
            if (!mPackages.containsKey("android")) {
                throw new IllegalStateException(
                        "Failed to load frameworks package; check log for warnings");
            }

            // Collect privileged system packages.  
            // 扫描收集私有特权的系统包： system/priv_app 目录的包文件
            final File privilegedAppDir = new File(Environment.getRootDirectory(), "priv-app");
            scanDirTracedLI(privilegedAppDir,
                    mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM_DIR,
                    scanFlags
                    | SCAN_AS_SYSTEM
                    | SCAN_AS_PRIVILEGED,
                    0);

            // Collect ordinary system packages.
            // 扫描收集普通系统包： system/app 目录的包文件            
            final File systemAppDir = new File(Environment.getRootDirectory(), "app");
            scanDirTracedLI(systemAppDir,
                    mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM_DIR,
                    scanFlags
                    | SCAN_AS_SYSTEM,
                    0);

            // Collect privileged vendor packages.
          
            File privilegedVendorAppDir = new File(Environment.getVendorDirectory(), "priv-app");
            try {
                privilegedVendorAppDir = privilegedVendorAppDir.getCanonicalFile();
            } catch (IOException e) {
                // failed to look up canonical path, continue with original one
            }
              // vender/priv-app
            scanDirTracedLI(privilegedVendorAppDir,
                    mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM_DIR,
                    scanFlags
                    | SCAN_AS_SYSTEM
                    | SCAN_AS_VENDOR
                    | SCAN_AS_PRIVILEGED,
                    0);

            // Collect ordinary vendor packages.
         
            File vendorAppDir = new File(Environment.getVendorDirectory(), "app");
            try {
                vendorAppDir = vendorAppDir.getCanonicalFile();
            } catch (IOException e) {
                // failed to look up canonical path, continue with original one
            }
                 // vender/app
            scanDirTracedLI(vendorAppDir,
                    mDefParseFlags
                    | PackageParser.PARSE_IS_SYSTEM_DIR,
                    scanFlags
                    | SCAN_AS_SYSTEM
                    | SCAN_AS_VENDOR,
                    0);

            // Collect privileged odm packages. /odm is another vendor partition
            // other than /vendor.
            File privilegedOdmAppDir = new File(Environment.getOdmDirectory(),
                        "priv-app");
            //除了 vender/app和vender/priv-app，还有 odm/app和odm/priv-app、oem/app和oem/priv-app
            product/app和product/priv-app、product_services/app和product_services/priv-app目录，也会扫描。
            ...

            // Prune any system packages that no longer exist.
            //  可能需要删除的系统 app
            final List<String> possiblyDeletedUpdatedSystemApps = new ArrayList<>();
            // Stub packages must either be replaced with full versions in the /data
            // partition or be disabled.
            final List<String> stubSystemApps = new ArrayList<>();
            // 收集有可能需要删除的系统app
            if (!mOnlyCore) {
                // do this first before mucking with mPackages for the "expecting better" case
                final Iterator<PackageParser.Package> pkgIterator = mPackages.values().iterator();
                while (pkgIterator.hasNext()) {
                    final PackageParser.Package pkg = pkgIterator.next();
                    if (pkg.isStub) {
                        stubSystemApps.add(pkg.packageName);
                    }
                }

                final Iterator<PackageSetting> psit = mSettings.mPackages.values().iterator();
                while (psit.hasNext()) {
                    PackageSetting ps = psit.next();

                    /*
                     * If this is not a system app, it can't be a
                     * disable system app.
                     */
                    if ((ps.pkgFlags & ApplicationInfo.FLAG_SYSTEM) == 0) {
                        continue;
                    }

                    /*
                     * If the package is scanned, it's not erased.
                     */
                    final PackageParser.Package scannedPkg = mPackages.get(ps.name);
                    if (scannedPkg != null) {
                        /*
                         * If the system app is both scanned and in the
                         * disabled packages list, then it must have been
                         * added via OTA. Remove it from the currently
                         * scanned package so the previously user-installed
                         * application can be scanned.
                         */
                        if (mSettings.isDisabledSystemPackageLPr(ps.name)) {
                        // 如果存在，且同时被标记了disable状态
                            logCriticalInfo(Log.WARN,
                                    "Expecting better updated system app for " + ps.name
                                    + "; removing system app.  Last known"
                                    + " codePath=" + ps.codePathString
                                    + ", versionCode=" + ps.versionCode
                                    + "; scanned versionCode=" + scannedPkg.getLongVersionCode());
                            removePackageLI(scannedPkg, true);
                            //  表示已经升级过了，是OAT升级包，作为更好的选择
                            mExpectingBetter.put(ps.name, ps.codePath);
                        }

                        continue;
                    }

                    if (!mSettings.isDisabledSystemPackageLPr(ps.name)) {
                        // 如果没有被标记，则移除。
                        psit.remove(); 
                        logCriticalInfo(Log.WARN, "System package " + ps.name
                                + " no longer exists; it's data will be wiped");
                        // Actual deletion of code and data will be handled by later
                        // reconciliation step
                    } else {
                        // 如果被标记为 disabled
                        // we still have a disabled system package, but, it still might have
                        // been removed. check the code path still exists and check there's
                        // still a package. the latter can happen if an OTA keeps the same
                        // code path, but, changes the package name.
                        final PackageSetting disabledPs =
                                mSettings.getDisabledSystemPkgLPr(ps.name);
                        if (disabledPs.codePath == null || !disabledPs.codePath.exists()
                                || disabledPs.pkg == null) {
                             //  system的代码文件不存在， 则加入到可能被移除的列表。由于此时没法确定，只能等到扫描完data
                        分区后才能确定！！
                            possiblyDeletedUpdatedSystemApps.add(ps.name);
                        } else {
                            // We're expecting that the system app should remain disabled, but add
                            // it to expecting better to recover in case the data version cannot
                            // be scanned.
                            // 如果system的文件存在，则认为无更新。统一加入到已升级的列表中
                            mExpectingBetter.put(disabledPs.name, disabledPs.codePath);
                        }
                    }
                }
            }

            //delete tmp files
            deleteTempPackageFiles();

            final int cachedSystemApps = PackageParser.sCachedPackageReadCount.get();

            // Remove any shared userIDs that have no associated packages
            mSettings.pruneSharedUsersLPw();
            final long systemScanTime = SystemClock.uptimeMillis() - startTime;
            final int systemPackagesCount = mPackages.size();
            Slog.i(TAG, "Finished scanning system apps. Time: " + systemScanTime
                    + " ms, packageCount: " + systemPackagesCount
                    + " , timePerPackage: "
                    + (systemPackagesCount == 0 ? 0 : systemScanTime / systemPackagesCount)
                    + " , cached: " + cachedSystemApps);
            if (mIsUpgrade && systemPackagesCount > 0) {
                MetricsLogger.histogram(null, "ota_package_manager_system_app_avg_scan_time",
                        ((int) systemScanTime) / systemPackagesCount);
            }
```

总结：

1. 从 init.rc 文件中得到环境变量：bootClassPath和systemClassPath。即 system/framework下的系统共享库。如 xxx.jar。
2. 对于Android6.0之前升级上来的版本，进行权限调整。从安装时候授权改为动态申请
3. 优先扫描 vendor/product/product_services、system/framewrok等目录。
4. 遍历 mPackages所有包，区分出有可能被移除的系统app，加入possiblyDeletedUpdatedSystemApps，以及已经更新完成的系统app
   ，加入mExpectingBetter。

> 可以通过adb shell env 来查看系统的环境变量，如 bootClassPath 。

### 系统app的三种更新逻辑

- 无更新
- 有更新 在新的OAT版本中，这个系统app有更新
- 被移除 在新的OAT版本中，这个系统app被移除

1. 当 PKMS的mSettings中的系统app被标记了disable且pkg存在mPackage中，则认为已经更新了系统app，则加入到mExpectingBetter。
2. 没有被标记，则移除。
3. 没有被标记，是在system/目录下却找不到：最认为可能会被移除 possiblyDeletedUpdatedSystemApps；如果找到了则认为
   是无更新，统一加入mExpectingBetter。

### system 目录下的子目录

- system/app  : 系统的应用。Android系统内置app或者厂商的app
- system/framework ： 存放framework框架层的jar包
- system/priv-app ： 存放特权app
- system/lib ： 存放系统公共库 so
- system/fonts ： 存放系统的字体
- system/media ： 存放闹钟的声音、通知的声音等文件

## 3.3 data扫描阶段 DATA_SCAN_START

```
    // data/app 
 private static final File sAppInstallDir =
            new File(Environment.getDataDirectory(), "app");

// false 表示没有加密。开始扫描data区 
if (!mOnlyCore) { 
                EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_DATA_SCAN_START,
                        SystemClock.uptimeMillis());
                 // 扫描  /data/app/ 目录
                // 扫描apk，解析得到 application、activity、service、broadcast、providers等信息。
                scanDirTracedLI(sAppInstallDir, 0, scanFlags | SCAN_REQUIRE_KNOWN, 0);

                // Remove disable package settings for updated system apps that were
                // removed via an OTA. If the update is no longer present, remove the
                // app completely. Otherwise, revoke their system privileges.
                // 删除可能需要删除的包
                for (int i = possiblyDeletedUpdatedSystemApps.size() - 1; i >= 0; --i) {
                    final String packageName = possiblyDeletedUpdatedSystemApps.get(i);
                    // 从mPackages中获取 pkg
                    final PackageParser.Package pkg = mPackages.get(packageName);
                    final String msg;

                    // remove from the disabled system list; do this first so any future
                    // scans of this package are performed without this state
                    mSettings.removeDisabledSystemPackageLPw(packageName);

                    if (pkg == null) {
                        // should have found an update, but, we didn't; remove everything
                        // data分区也找不到。那就删除这些数据
                        msg = "Updated system package " + packageName
                                + " no longer exists; removing its data";
                        // Actual deletion of code and data will be handled by later
                        // reconciliation step
                    } else {
                        // found an update; revoke system privileges
                        //找到了，需要去除系统特权标识
                        msg = "Updated system package " + packageName
                                + " no longer exists; rescanning package on data";

                        // NOTE: We don't do anything special if a stub is removed from the
                        // system image. But, if we were [like removing the uncompressed
                        // version from the /data partition], this is where it'd be done.

                        // remove the package from the system and re-scan it without any
                        // special privileges
                        //移除特权
                        removePackageLI(pkg, true);
                        try {
                            final File codePath = new File(pkg.applicationInfo.getCodePath());
                            //当做普通app，重新扫描这个包
                            scanPackageTracedLI(codePath, 0, scanFlags, 0, null);
                        } catch (PackageManagerException e) {
                            Slog.e(TAG, "Failed to parse updated, ex-system package: "
                                    + e.getMessage());
                        }
                    }

                    // one final check. if we still have a package setting [ie. it was
                    // previously scanned and known to the system], but, we don't have
                    // a package [ie. there was an error scanning it from the /data
                    // partition], completely remove the package data.
                    final PackageSetting ps = mSettings.mPackages.get(packageName);
                    if (ps != null && mPackages.get(packageName) == null) {
                        removePackageDataLIF(ps, null, null, 0, false);

                    }
                    logCriticalInfo(Log.WARN, msg);
                }

                /*
                 * Make sure all system apps that we expected to appear on
                 * the userdata partition actually showed up. If they never
                 * appeared, crawl back and revive the system version.
                 */
                 // 遍历mExpectingBetter列表，表示最终要展示的系统app
                for (int i = 0; i < mExpectingBetter.size(); i++) {
                    final String packageName = mExpectingBetter.keyAt(i);
                    //如果在data分区中也没有找到，则可能在OAT升级过程中被清除数据。那么需要重新扫描system/app、system/priv-app等系统目录
                    if (!mPackages.containsKey(packageName)) {
                        final File scanFile = mExpectingBetter.valueAt(i);

                        logCriticalInfo(Log.WARN, "Expected better " + packageName
                                + " but never showed up; reverting to system");

                        final @ParseFlags int reparseFlags;
                        final @ScanFlags int rescanFlags;
                        if (FileUtils.contains(privilegedAppDir, scanFile)) {
                            reparseFlags =
                                    mDefParseFlags |
                                    PackageParser.PARSE_IS_SYSTEM_DIR;
                            rescanFlags =
                                    scanFlags
                                    | SCAN_AS_SYSTEM
                                    | SCAN_AS_PRIVILEGED;
                        } else if (FileUtils.contains(systemAppDir, scanFile)) {
                            reparseFlags =
                                    mDefParseFlags |
                                    PackageParser.PARSE_IS_SYSTEM_DIR;
                            rescanFlags =
                                    scanFlags
                                    | SCAN_AS_SYSTEM;
                        } else if (FileUtils.contains(privilegedVendorAppDir, scanFile)
                                || FileUtils.contains(privilegedOdmAppDir, scanFile)) {
                            reparseFlags =
                                    mDefParseFlags |
                                    PackageParser.PARSE_IS_SYSTEM_DIR;
                            rescanFlags =
                                    scanFlags
                                    | SCAN_AS_SYSTEM
                                    | SCAN_AS_VENDOR
                                    | SCAN_AS_PRIVILEGED;
                        } else if (FileUtils.contains(vendorAppDir, scanFile)
                                || FileUtils.contains(odmAppDir, scanFile)) {
                            reparseFlags =
                                    mDefParseFlags |
                                    PackageParser.PARSE_IS_SYSTEM_DIR;
                            rescanFlags =
                                    scanFlags
                                    | SCAN_AS_SYSTEM
                                    | SCAN_AS_VENDOR;
                        } else if (FileUtils.contains(oemAppDir, scanFile)) {
                            reparseFlags =
                                    mDefParseFlags |
                                    PackageParser.PARSE_IS_SYSTEM_DIR;
                            rescanFlags =
                                    scanFlags
                                    | SCAN_AS_SYSTEM
                                    | SCAN_AS_OEM;
                        } else if (FileUtils.contains(privilegedProductAppDir, scanFile)) {
                            reparseFlags =
                                    mDefParseFlags |
                                    PackageParser.PARSE_IS_SYSTEM_DIR;
                            rescanFlags =
                                    scanFlags
                                    | SCAN_AS_SYSTEM
                                    | SCAN_AS_PRODUCT
                                    | SCAN_AS_PRIVILEGED;
                        } else if (FileUtils.contains(productAppDir, scanFile)) {
                            reparseFlags =
                                    mDefParseFlags |
                                    PackageParser.PARSE_IS_SYSTEM_DIR;
                            rescanFlags =
                                    scanFlags
                                    | SCAN_AS_SYSTEM
                                    | SCAN_AS_PRODUCT;
                        } else if (FileUtils.contains(privilegedProductServicesAppDir, scanFile)) {
                            reparseFlags =
                                    mDefParseFlags |
                                    PackageParser.PARSE_IS_SYSTEM_DIR;
                            rescanFlags =
                                    scanFlags
                                    | SCAN_AS_SYSTEM
                                    | SCAN_AS_PRODUCT_SERVICES
                                    | SCAN_AS_PRIVILEGED;
                        } else if (FileUtils.contains(productServicesAppDir, scanFile)) {
                            reparseFlags =
                                    mDefParseFlags |
                                    PackageParser.PARSE_IS_SYSTEM_DIR;
                            rescanFlags =
                                    scanFlags
                                    | SCAN_AS_SYSTEM
                                    | SCAN_AS_PRODUCT_SERVICES;
                        } else {
                            Slog.e(TAG, "Ignoring unexpected fallback path " + scanFile);
                            continue;
                        }
                        //更新settings中的 packageSetting参数为true
                        mSettings.enableSystemPackageLPw(packageName);
                        
                        try {
                         // 扫描系统App的升级包
                            scanPackageTracedLI(scanFile, reparseFlags, rescanFlags, 0, null);
                        } catch (PackageManagerException e) {
                            Slog.e(TAG, "Failed to parse original system package: "
                                    + e.getMessage());
                        }
                    }
                }

                // Uncompress and install any stubbed system applications.
                // 解压和安装 系统应用 app
                // This must be done last to ensure all stubs are replaced or disabled.
                installSystemStubPackages(stubSystemApps, scanFlags);

                final int cachedNonSystemApps = PackageParser.sCachedPackageReadCount.get()
                                - cachedSystemApps;

                final long dataScanTime = SystemClock.uptimeMillis() - systemScanTime - startTime;
                final int dataPackagesCount = mPackages.size() - systemPackagesCount;
                Slog.i(TAG, "Finished scanning non-system apps. Time: " + dataScanTime
                        + " ms, packageCount: " + dataPackagesCount
                        + " , timePerPackage: "
                        + (dataPackagesCount == 0 ? 0 : dataScanTime / dataPackagesCount)
                        + " , cached: " + cachedNonSystemApps);
                if (mIsUpgrade && dataPackagesCount > 0) {
                    MetricsLogger.histogram(null, "ota_package_manager_data_app_avg_scan_time",
                            ((int) dataScanTime) / dataPackagesCount);
                }
            }
            //安装完了，可以清空
            mExpectingBetter.clear();

            // Resolve the storage manager.
            // storage 解析类
            mStorageManagerPackage = getStorageManagerPackageName();

            // Resolve protected action filters. Only the setup wizard is allowed to
            // have a high priority filter for these actions.
            // 解析intent-filters，
            mSetupWizardPackage = getSetupWizardPackageName();
            mComponentResolver.fixProtectedFilterPriorities();

            mSystemTextClassifierPackage = getSystemTextClassifierPackageName();

            mWellbeingPackage = getWellbeingPackageName();
            mDocumenterPackage = getDocumenterPackageName();
            mConfiguratorPackage =
                    mContext.getString(R.string.config_deviceConfiguratorPackageName);
            mAppPredictionServicePackage = getAppPredictionServicePackageName();
            mIncidentReportApproverPackage = getIncidentReportApproverPackageName();

            // Now that we know all of the shared libraries, update all clients to have
            // the correct library paths.
            updateAllSharedLibrariesLocked(null, Collections.unmodifiableMap(mPackages));
              // 更新共享库路径 
            for (SharedUserSetting setting : mSettings.getAllSharedUsersLPw()) {
                // NOTE: We ignore potential failures here during a system scan (like
                // the rest of the commands above) because there's precious little we
                // can do about it. A settings error is reported, though.
                final List<String> changedAbiCodePath =
                        adjustCpuAbisForSharedUserLPw(setting.packages, null /*scannedPackage*/);
                if (changedAbiCodePath != null && changedAbiCodePath.size() > 0) {
                    for (int i = changedAbiCodePath.size() - 1; i >= 0; --i) {
                        final String codePathString = changedAbiCodePath.get(i);
                        try {
                            mInstaller.rmdex(codePathString,
                                    getDexCodeInstructionSet(getPreferredInstructionSet()));
                        } catch (InstallerException ignored) {
                        }
                    }
                }
                // Adjust seInfo to ensure apps which share a sharedUserId are placed in the same
                // SELinux domain.
                setting.fixSeInfoLocked();
            }

            // Now that we know all the packages we are keeping,
            // read and update their last usage times.
            mPackageUsage.read(mPackages);
            mCompilerStats.read();

```

总结：

1. 扫描 /data/app/ 目录。解析apk，解析得到 application、activity、service、broadcast、providers等信息。保存到mPackages中。

2. 遍历 possiblyDeletedUpdatedSystemApps 可能要删除的系统app列表，检查升级文件名是否存在mPacakages中，如果不存在则说明在data分区也没找到，是
   残留信息，确实要删除；如果在data分区存在，则说明是system目录下文件可能被移除了。那么就移除系统特权，以普通app的权限再次扫描。
3. 遍历mExpectingBetter列表中的系统app，理论上应该在data分区能够找对应的目录，此时不作额外处理。但是如果在data分区找不到，那么就认为可能在OAT
   升级过程中被删除了，所以需要再次扫描system分区。
4. 开始安装系统app
5. 更新共享库的路径

### data目录的子目录

- /data/app  :存储用户自己安装的app。包含apk、优化后的lat文件、so库。
- /data/data :  存放所有已安装的App数据，根据包名划分子目录
- /data/app-private： app的私有存储空间
- /data/app-lib ：存储所有app的jni库
- /data/system ：存放系统的配置文件
- /data/anr ： 存放anr发生时候的trace文件

## 3.4 扫描结束阶段

```
EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_SCAN_END,
                    SystemClock.uptimeMillis());
            Slog.i(TAG, "Time to scan packages: "
                    + ((SystemClock.uptimeMillis()-startTime)/1000f)
                    + " seconds");

            // If the platform SDK has changed since the last time we booted,
            // we need to re-grant app permission to catch any new ones that
            // appear.  This is really a hack, and means that apps can in some
            // cases get permissions that the user didn't initially explicitly
            // allow...  it would be nice to have some better way to handle
            // this situation.
            // 如果此次 sdk 升级的版本不一致
            final boolean sdkUpdated = (ver.sdkVersion != mSdkVersion);
            if (sdkUpdated) {
                Slog.i(TAG, "Platform changed from " + ver.sdkVersion + " to "
                        + mSdkVersion + "; regranting permissions for internal storage");
            }
              // 如果此次 sdk 升级的版本不一致，则更新所以packages的权限 
            mPermissionManager.updateAllPermissions(
                    StorageManager.UUID_PRIVATE_INTERNAL, sdkUpdated, mPackages.values(),
                    mPermissionCallback);
            ver.sdkVersion = mSdkVersion;

            // If this is the first boot or an update from pre-M, and it is a normal
            // boot, then we need to initialize the default preferred apps across
            // all defined users.
            // 如果是第一次启动，或者是更新完Android6.0之后的第一次启动，则初始化默认首选app
            if (!onlyCore && (mPromoteSystemApps || mFirstBoot)) {
                for (UserInfo user : sUserManager.getUsers(true)) {
                    mSettings.applyDefaultPreferredAppsLPw(user.id);
                    primeDomainVerificationsLPw(user.id);
                }
            }

            // Prepare storage for system user really early during boot,
            // since core system apps like SettingsProvider and SystemUI
            // can't wait for user to start
            final int storageFlags;
            if (StorageManager.isFileEncryptedNativeOrEmulated()) {
                storageFlags = StorageManager.FLAG_STORAGE_DE;
            } else {
                storageFlags = StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE;
            }
            List<String> deferPackages = reconcileAppsDataLI(StorageManager.UUID_PRIVATE_INTERNAL,
                    UserHandle.USER_SYSTEM, storageFlags, true /* migrateAppData */,
                    true /* onlyCoreApps */);
            mPrepareAppDataFuture = SystemServerInitThreadPool.get().submit(() -> {
                TimingsTraceLog traceLog = new TimingsTraceLog("SystemServerTimingAsync",
                        Trace.TRACE_TAG_PACKAGE_MANAGER);
                traceLog.traceBegin("AppDataFixup");
                try {
                    mInstaller.fixupAppData(StorageManager.UUID_PRIVATE_INTERNAL,
                            StorageManager.FLAG_STORAGE_DE | StorageManager.FLAG_STORAGE_CE);
                } catch (InstallerException e) {
                    Slog.w(TAG, "Trouble fixing GIDs", e);
                }
                traceLog.traceEnd();

                traceLog.traceBegin("AppDataPrepare");
                if (deferPackages == null || deferPackages.isEmpty()) {
                    return;
                }
                int count = 0;
                for (String pkgName : deferPackages) {
                    PackageParser.Package pkg = null;
                    synchronized (mPackages) {
                        PackageSetting ps = mSettings.getPackageLPr(pkgName);
                        if (ps != null && ps.getInstalled(UserHandle.USER_SYSTEM)) {
                            pkg = ps.pkg;
                        }
                    }
                    if (pkg != null) {
                        synchronized (mInstallLock) {
                            prepareAppDataAndMigrateLIF(pkg, UserHandle.USER_SYSTEM, storageFlags,
                                    true /* maybeMigrateAppData */);
                        }
                        count++;
                    }
                }
                traceLog.traceEnd();
                Slog.i(TAG, "Deferred reconcileAppsData finished " + count + " packages");
            }, "prepareAppData");

            // If this is first boot after an OTA, and a normal boot, then
            // we need to clear code cache directories.
            // Note that we do *not* clear the application profiles. These remain valid
            // across OTAs and are used to drive profile verification (post OTA) and
            // profile compilation (without waiting to collect a fresh set of profiles).
            if (mIsUpgrade && !onlyCore) {
                Slog.i(TAG, "Build fingerprint changed; clearing code caches");
                // 如果升级了版本，且第一次启动OAT，清楚appdata目录下的代码缓存 
                for (int i = 0; i < mSettings.mPackages.size(); i++) {
                    final PackageSetting ps = mSettings.mPackages.valueAt(i);
                    if (Objects.equals(StorageManager.UUID_PRIVATE_INTERNAL, ps.volumeUuid)) {
                        // No apps are running this early, so no need to freeze
                        
                        clearAppDataLIF(ps.pkg, UserHandle.USER_ALL,
                                FLAG_STORAGE_DE | FLAG_STORAGE_CE | FLAG_STORAGE_EXTERNAL
                                        | Installer.FLAG_CLEAR_CODE_CACHE_ONLY);
                    }
                }
                ver.fingerprint = Build.FINGERPRINT;
            }

            // Grandfather existing (installed before Q) non-system apps to hide
            // their icons in launcher.
            if (!onlyCore && mIsPreQUpgrade) {
                Slog.i(TAG, "Whitelisting all existing apps to hide their icons");
                int size = mSettings.mPackages.size();
                for (int i = 0; i < size; i++) {
                    final PackageSetting ps = mSettings.mPackages.valueAt(i);
                    if ((ps.pkgFlags & ApplicationInfo.FLAG_SYSTEM) != 0) {
                        continue;
                    }
                    ps.disableComponentLPw(PackageManager.APP_DETAILS_ACTIVITY_CLASS_NAME,
                            UserHandle.USER_SYSTEM);
                }
            }

            // clear only after permissions and other defaults have been updated
            mExistingSystemPackages.clear();
            mPromoteSystemApps = false;

            // All the changes are done during package scanning.
            ver.databaseVersion = Settings.CURRENT_DATABASE_VERSION;

            // can downgrade to reader
            Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "write settings");
            //更新 settings
            mSettings.writeLPr();
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
```

总结：

1. 如果OAT版本不一致，则更新所有权限
2. 如果是第一次启动，或者是更新完Android6.0之后的第一次启动，则初始化默认首选app
3. 如果升级OAT后第一次启动，清除代码缓存目录
4. 更新settings内容到 package.xml

## 3.5 准备阶段

```
 EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_PMS_READY,
                    SystemClock.uptimeMillis());

            if (!mOnlyCore) {
                mRequiredVerifierPackage = getRequiredButNotReallyRequiredVerifierLPr();
                mRequiredInstallerPackage = getRequiredInstallerLPr();
                mRequiredUninstallerPackage = getRequiredUninstallerLPr();
                mIntentFilterVerifierComponent = getIntentFilterVerifierComponentNameLPr();
                if (mIntentFilterVerifierComponent != null) {
                    mIntentFilterVerifier = new IntentVerifierProxy(mContext,
                            mIntentFilterVerifierComponent);
                } else {
                    mIntentFilterVerifier = null;
                }
                mServicesSystemSharedLibraryPackageName = getRequiredSharedLibraryLPr(
                        PackageManager.SYSTEM_SHARED_LIBRARY_SERVICES,
                        SharedLibraryInfo.VERSION_UNDEFINED);
                mSharedSystemSharedLibraryPackageName = getRequiredSharedLibraryLPr(
                        PackageManager.SYSTEM_SHARED_LIBRARY_SHARED,
                        SharedLibraryInfo.VERSION_UNDEFINED);
            } else {
                mRequiredVerifierPackage = null;
                mRequiredInstallerPackage = null;
                mRequiredUninstallerPackage = null;
                mIntentFilterVerifierComponent = null;
                mIntentFilterVerifier = null;
                mServicesSystemSharedLibraryPackageName = null;
                mSharedSystemSharedLibraryPackageName = null;
            }
            // PermissionController hosts default permission granting and role management, so it's a
            // critical part of the core system.
            mRequiredPermissionControllerPackage = getRequiredPermissionControllerLPr();

            // Initialize InstantAppRegistry's Instant App list for all users.
            final int[] userIds = UserManagerService.getInstance().getUserIds();
            for (PackageParser.Package pkg : mPackages.values()) {
                if (pkg.isSystem()) {
                    continue;
                }
                for (int userId : userIds) {
                    final PackageSetting ps = (PackageSetting) pkg.mExtras;
                    if (ps == null || !ps.getInstantApp(userId) || !ps.getInstalled(userId)) {
                        continue;
                    }
                    mInstantAppRegistry.addInstantAppLPw(userId, ps.appId);
                }
            }

            // 安装服务
            mInstallerService = new PackageInstallerService(context, this, mApexManager);
            final Pair<ComponentName, String> instantAppResolverComponent =
                    getInstantAppResolverLPr();
            if (instantAppResolverComponent != null) {
                if (DEBUG_INSTANT) {
                    Slog.d(TAG, "Set ephemeral resolver: " + instantAppResolverComponent);
                }
                mInstantAppResolverConnection = new InstantAppResolverConnection(
                        mContext, instantAppResolverComponent.first,
                        instantAppResolverComponent.second);
                mInstantAppResolverSettingsComponent =
                        getInstantAppResolverSettingsLPr(instantAppResolverComponent.first);
            } else {
                mInstantAppResolverConnection = null;
                mInstantAppResolverSettingsComponent = null;
            }
            updateInstantAppInstallerLocked(null);

            // Read and update the usage of dex files.
            // Do this at the end of PM init so that all the packages have their
            // data directory reconciled.
            // At this point we know the code paths of the packages, so we can validate
            // the disk file and build the internal cache.
            // The usage file is expected to be small so loading and verifying it
            // should take a fairly small time compare to the other activities (e.g. package
            // scanning).
            final Map<Integer, List<PackageInfo>> userPackages = new HashMap<>();
            for (int userId : userIds) {
                userPackages.put(userId, getInstalledPackages(/*flags*/ 0, userId).getList());
            }
            // 加载 dex文件
            mDexManager.load(userPackages);
            if (mIsUpgrade) {
                MetricsLogger.histogram(null, "ota_package_manager_init_time",
                        (int) (SystemClock.uptimeMillis() - startTime));
            }
        } // synchronized (mPackages)
        } // synchronized (mInstallLock)

        mModuleInfoProvider = new ModuleInfoProvider(mContext, this);

        // Now after opening every single application zip, make sure they
        // are all flushed.  Not really needed, but keeps things nice and
        // tidy.
        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "GC");
        // 调用gc，回收垃圾
        Runtime.getRuntime().gc();
        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);

        // The initial scanning above does many calls into installd while
        // holding the mPackages lock, but we're mostly interested in yelling
        // once we have a booted system.
        mInstaller.setWarnIfHeld(mPackages);

        PackageParser.readConfigUseRoundIcon(mContext.getResources());

        mServiceStartWithDelay = SystemClock.uptimeMillis() + (60 * 1000L);

        Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
    }
```

总结：

1. 创建 PackageInstallerService 对象，用来安装apk时使用
2. 主动gc

# 四、dex 优化

在 systemServer的 startOtherServices()方法中会调用 PKMS的方法：

```
 private void startCoreServices() {
     
     ...
     
     if (!mOnlyCore) {
                traceBeginAndSlog("UpdatePackagesIfNeeded");
                try {
                    Watchdog.getInstance().pauseWatchingCurrentThread("dexopt");
                    // 1 dex优化
                    mPackageManagerService.updatePackagesIfNeeded();
                } catch (Throwable e) {
                    reportWtf("update packages", e);
                } finally {
                    Watchdog.getInstance().resumeWatchingCurrentThread("dexopt");
                }
                traceEnd();
            }
    
            traceBeginAndSlog("PerformFstrimIfNeeded");
            try {
                // 执行ftrim
                mPackageManagerService.performFstrimIfNeeded();
            } catch (Throwable e) {
                reportWtf("performing fstrim", e);
            }
            traceEnd();
            ...
            traceBeginAndSlog("MakePackageManagerServiceReady");
            // 3 调用 systemReady() 方法
            mPackageManagerService.systemReady();
            
            ...
             // Wait for all packages to be prepared
             4. 等待所以的包安装完成
            mPackageManagerService.waitForAppDataPrepared();
    }            ...   
}
```

1. updatePackagesIfNeeded(),内部会调用 performDexOptUpgrade() 进行dex优化
2. 磁盘清理 ，清理空间
3. systemReady(),默认授权和更新package的信息,通知 PKMS相关对象已经初始化完成
4. 等待所有的包安装完成

## 4.1  updatePackagesIfNeeded()

```
 public void updatePackagesIfNeeded() {
        // 必须是system 或者root 才可以访问
        enforceSystemOrRoot("Only the system can request package update");

        // We need to re-extract after an OTA.
        在OAT更新后，重新抽取
        boolean causeUpgrade = isDeviceUpgrading();

        // First boot or factory reset.
        // Note: we also handle devices that are upgrading to N right now as if it is their
        //       first boot, as they do not have profile data.
        // 第一次启动
        boolean causeFirstBoot = isFirstBoot() || mIsPreNUpgrade;

        // We need to re-extract after a pruned cache, as AoT-ed files will be out of date.
        // 是否清理过虚拟机的缓存
        boolean causePrunedCache = VMRuntime.didPruneDalvikCache();
        // 如果 没有OAT升级，也不是第一次启动，也没有清理过虚拟机的缓存，那么就无需dex优化。
        if (!causeUpgrade && !causeFirstBoot && !causePrunedCache) {
            return;
        }

        List<PackageParser.Package> pkgs;
        synchronized (mPackages) {
            // 获取需要dex优化的包。排序规则： core apps > system apps > other apps
            pkgs = PackageManagerServiceUtils.getPackagesForDexopt(mPackages.values(), this);
        }

        final long startTime = System.nanoTime();
           //按照排序后，执行 dex优化。内部最终通过 mInstaller.dexopt 来执行真正的dex优化
        final int[] stats = performDexOptUpgrade(pkgs, mIsPreNUpgrade /* showDialog */,
                    causeFirstBoot ? REASON_FIRST_BOOT : REASON_BOOT,
                    false /* bootComplete */);

        final int elapsedTimeSeconds =
                (int) TimeUnit.NANOSECONDS.toSeconds(System.nanoTime() - startTime);

        MetricsLogger.histogram(mContext, "opt_dialog_num_dexopted", stats[0]);
        MetricsLogger.histogram(mContext, "opt_dialog_num_skipped", stats[1]);
        MetricsLogger.histogram(mContext, "opt_dialog_num_failed", stats[2]);
        MetricsLogger.histogram(mContext, "opt_dialog_num_total", getOptimizablePackages().size());
        MetricsLogger.histogram(mContext, "opt_dialog_time_s", elapsedTimeSeconds);
    }
```

总结：

1. 如果 没有OAT升级，也不是第一次启动，也没有清理过虚拟机的缓存，那么就无需dex优化。
2. 根据规则排序： core apps > system apps > other apps
3. 按照排序规则，最终调用 mInstaller.dexopt() 来完成 dex优化。

# installd 守护进程启动

> /frameworks/native/cmds/installd/installd.rc

```
2 service installd /system/bin/installd
3     class main
```

installd是一个进程？？？是的！ 专门用来干活的。 具有root权限，pms的SystemServer只有 system 权限。

PMS中安装、移除apk都是通过 installer服务来完成的。而 installer服务只是在java层，真正干活是通过 installd进程 的binder服务
InstalldNativeService 来完成的。

开启了binder线程来通信。没有启动socket了。android 6.0是通过socket来通信的。

init进程解析installd.rc文件，启动service，fork出installd进程。执行入口函数main()。

# installd.cpp

> /frameworks/native/cmds/installd/installd.cpp

```
int main(const int argc, char *argv[]) {
239      return android::installd::installd_main(argc, argv);
240  }
```

installd_main():

```
static int installd_main(const int argc ATTRIBUTE_UNUSED, char *argv[]) {
196      int ret;
197      int selinux_enabled = (is_selinux_enabled() > 0);
198  
199      setenv("ANDROID_LOG_TAGS", "*:v", 1);
200      android::base::InitLogging(argv);
201  
202      SLOGI("installd firing up");
203  
204      union selinux_callback cb;
205      cb.func_log = log_callback;
206      selinux_set_callback(SELINUX_CB_LOG, cb);
207  
208      if (!initialize_globals()) {
209          SLOGE("Could not initialize globals; exiting.\n");
210          exit(1);
211      }
212  
213      if (initialize_directories() < 0) {
214          SLOGE("Could not create directories; exiting.\n");
215          exit(1);
216      }
217  
218      if (selinux_enabled && selinux_status_open(true) < 0) {
219          SLOGE("Could not open selinux status; exiting.\n");
220          exit(1);
221      }
222      // 内部通过native代码注册 installd服务到SM中。同时开启binder线程
223      if ((ret = InstalldNativeService::start()) != android::OK) {
224          SLOGE("Unable to start InstalldNativeService: %d", ret);
225          exit(1);
226      }
227      // 在IPCThreadState中通过 talKWitchDriver()不断的与binder驱动通信，读写命令
228      IPCThreadState::self()->joinThreadPool();
229  
230      LOG(INFO) << "installd shutting down";
231  
232      return 0;
233  }
```

## initialize_globals()

```
static bool initialize_globals() {
69      return init_globals_from_data_and_root();
70  }
```

```
 bool init_globals_from_data_and_root() {
        // 获取目录： /data/
64      const char* data_path = getenv("ANDROID_DATA");
65      if (data_path == nullptr) {
66          LOG(ERROR) << "Could not find ANDROID_DATA";
67          return false;
68      }
        // 获取目录： data/system/
69      const char* root_path = getenv("ANDROID_ROOT");
70      if (root_path == nullptr) {
71          LOG(ERROR) << "Could not find ANDROID_ROOT";
72          return false;
73      }
74      return init_globals_from_data_and_root(data_path, root_path);
75  }
```

传入两个目录： /data/、/system/。部分常量如下：

```

31  static constexpr const char* APP_SUBDIR = "app/"; // sub-directory under ANDROID_DATA
32  
33  static constexpr const char* PRIV_APP_SUBDIR = "priv-app/"; // sub-directory under ANDROID_DATA
34  
35  static constexpr const char* EPHEMERAL_APP_SUBDIR = "app-ephemeral/"; // sub-directory under
36                                                                        // ANDROID_DATA
37  
38  static constexpr const char* APP_LIB_SUBDIR = "app-lib/"; // sub-directory under ANDROID_DATA
39  
40  static constexpr const char* MEDIA_SUBDIR = "media/"; // sub-directory under ANDROID_DATA
41  
42  static constexpr const char* PROFILES_SUBDIR = "misc/profiles"; // sub-directory under ANDROID_DATA
43  
44  static constexpr const char* PRIVATE_APP_SUBDIR = "app-private/"; // sub-directory under
45                                                                    // ANDROID_DATA
46  
47  static constexpr const char* STAGING_SUBDIR = "app-staging/"; // sub-directory under ANDROID_DATA
```

```
bool init_globals_from_data_and_root(const char* data, const char* root) {
86      // Get the android data directory. 
        // data/ 目录
87      android_data_dir = ensure_trailing_slash(data);
88  
89      // Get the android root directory.
        // system/   目录
90      android_root_dir = ensure_trailing_slash(root);
91  
92      // Get the android app directory.
        //  /data/app/    app 目录
93      android_app_dir = android_data_dir + APP_SUBDIR;
94  
95      // Get the android protected app directory.
        //  /data/app-private/   受保护的私有目录目录
96      android_app_private_dir = android_data_dir + PRIVATE_APP_SUBDIR;
97  
98      // Get the android ephemeral app directory.
        //  /data/app-ephemeral//   临时目录
99      android_app_ephemeral_dir = android_data_dir + EPHEMERAL_APP_SUBDIR;
100  
101      // Get the android app native library directory.
        //  /data/app-lib/   native库的目录
102      android_app_lib_dir = android_data_dir + APP_LIB_SUBDIR;
103  
104      // Get the sd-card ASEC mount point.
        //  /mnt/aesc/   sdcard挂载点目录
105      android_asec_dir = ensure_trailing_slash(getenv(ASEC_MOUNTPOINT_ENV_NAME));
106  
107      // Get the android media directory.
        //  /data/media//   目录
108      android_media_dir = android_data_dir + MEDIA_SUBDIR;
109  
110      // Get the android external app directory.
        //  /mnt/expand/   外部目录
111      android_mnt_expand_dir = "/mnt/expand/";
112  
113      // Get the android profiles directory.
         //  data/misc/profiles   
114      android_profiles_dir = android_data_dir + PROFILES_SUBDIR;
115  
116      // Get the android session staging directory.
            //  data/app-staging/  目录
117      android_staging_dir = android_data_dir + STAGING_SUBDIR;
118  
119      // Take note of the system and vendor directories.
120      android_system_dirs.clear();
          //  system/app/ 
121      android_system_dirs.push_back(android_root_dir + APP_SUBDIR);
          //  system/priv-app/
122      android_system_dirs.push_back(android_root_dir + PRIV_APP_SUBDIR);
            //   /vendor/app/
123      android_system_dirs.push_back("/vendor/app/");
        //   /oem/app/
124      android_system_dirs.push_back("/oem/app/");
125  
126      return true;
127  }
```

职责： 初始化 /data 、/system 下的相关目录。初始化厂商

## initialize_directories()

```
static int initialize_directories() {
73      int res = -1;
74  
75      // Read current filesystem layout version to handle upgrade paths
        // 读取文件系统 
76      char version_path[PATH_MAX];
77      snprintf(version_path, PATH_MAX, "%s.layout_version", android_data_dir.c_str());
78  
79      int oldVersion;
80      if (fs_read_atomic_int(version_path, &oldVersion) == -1) {
81          oldVersion = 0;
82      }
83      int version = oldVersion;
84  
85      if (version < 2) {
86          SLOGD("Assuming that device has multi-user storage layout; upgrade no longer supported");
87          version = 2;
88      }
89      // 创建 /data/misc/user 目录
90      if (ensure_config_user_dirs(0) == -1) {
91          SLOGE("Failed to setup misc for user 0");
92          goto fail;
93      }
94  
95      if (version == 2) {
96          SLOGD("Upgrading to /data/misc/user directories");
97         //  misc_dir= /data/misc/
98          char misc_dir[PATH_MAX];
99          snprintf(misc_dir, PATH_MAX, "%smisc", android_data_dir.c_str());
100             //  /data/misc/keychain/cacerts-added
101          char keychain_added_dir[PATH_MAX];
102          snprintf(keychain_added_dir, PATH_MAX, "%s/keychain/cacerts-added", misc_dir);
103             
104          char keychain_removed_dir[PATH_MAX];
                 //  /data/misc/keychain/cacerts-removed
105          snprintf(keychain_removed_dir, PATH_MAX, "%s/keychain/cacerts-removed", misc_dir);
106  
107          DIR *dir;
108          struct dirent *dirent;
                //  打开目录  /data/user 
109          dir = opendir("/data/user");
110          if (dir != nullptr) {
111              while ((dirent = readdir(dir))) {
112                  const char *name = dirent->d_name;
113  
114                  // skip "." and ".."
115                  if (name[0] == '.') {
116                      if (name[1] == 0) continue;
117                      if ((name[1] == '.') && (name[2] == 0)) continue;
118                  }
119                  // 根据名字获取uid
120                  uint32_t user_id = std::stoi(name);
121  
122                  // /data/misc/user/<user_id>
123                  if (ensure_config_user_dirs(user_id) == -1) {
124                      goto fail;
125                  }
126  
127                  char misc_added_dir[PATH_MAX];
                      // /data/misc/user/uid/keychain/cacerts-added
128                  snprintf(misc_added_dir, PATH_MAX, "%s/user/%s/cacerts-added", misc_dir, name);
129  
130                  char misc_removed_dir[PATH_MAX];
                      // /data/misc/user/uid/keychain/cacerts-removed
131                  snprintf(misc_removed_dir, PATH_MAX, "%s/user/%s/cacerts-removed", misc_dir, name);
132  
133                  uid_t uid = multiuser_get_uid(user_id, AID_SYSTEM);
134                  gid_t gid = uid;
                      // 拷贝 /data/misc/keychain/cacerts-added--> 
                      //  /data/misc/user/uid/keychain/cacerts-added
135                  if (access(keychain_added_dir, F_OK) == 0) {
136                      if (copy_dir_files(keychain_added_dir, misc_added_dir, uid, gid) != 0) {
137                          SLOGE("Some files failed to copy");
138                      }
139                  }
140                  if (access(keychain_removed_dir, F_OK) == 0) {
141                      if (copy_dir_files(keychain_removed_dir, misc_removed_dir, uid, gid) != 0) {
142                          SLOGE("Some files failed to copy");
143                      }
144                  }
145              }
146              closedir(dir);
147  
148              if (access(keychain_added_dir, F_OK) == 0) {
149                  delete_dir_contents(keychain_added_dir, 1, nullptr);
150              }
151              if (access(keychain_removed_dir, F_OK) == 0) {
152                  delete_dir_contents(keychain_removed_dir, 1, nullptr);
153              }
154          }
155  
156          version = 3;
157      }
158  
159      // Persist layout version if changed
160      if (version != oldVersion) {
            // 更新版本号
161          if (fs_write_atomic_int(version_path, version) == -1) {
162              SLOGE("Failed to save version to %s: %s", version_path, strerror(errno));
163              goto fail;
164          }
165      }
166  
167      // Success!
168      res = 0;
169  
170  fail:
171      return res;
172  }

```

1. 在 /data/user 目录下，遍历每个item，根据item获取uid。
2. 把 /data/misc/keychain/ 目录里面的内容拷贝到 /data/misc/user/uid/keychain/ 目录。
3. 删除 /data/misc/keychain/ 目录内容
4. 更新版本号

## 注册服务 InstalldNativeService

> /frameworks/native/cmds/installd/InstalldNativeService.cpp

```
 status_t InstalldNativeService::start() {
256      IPCThreadState::self()->disableBackgroundScheduling(true);
         // 通过模板类BinderService来 发布服务
257      status_t ret = BinderService<InstalldNativeService>::publish();
258      if (ret != android::OK) {
259          return ret;
260      }
261      sp<ProcessState> ps(ProcessState::self());
         // 内部通过 new PoolThread()来创建binder线程
262      ps->startThreadPool();
263      ps->giveThreadPoolName();
264      return android::OK;
265  }
```

模板类 ：

```
 // ---------------------------------------------------------------------------
31  namespace android {
32  
33  template<typename SERVICE>
34  class BinderService
35  {
36  public:// 提供了两个方法
37      static status_t publish(bool allowIsolated = false,
38                              int dumpFlags = IServiceManager::DUMP_FLAG_PRIORITY_DEFAULT) {
39          sp<IServiceManager> sm(defaultServiceManager());
40          return sm->addService(String16(SERVICE::getServiceName()), new SERVICE(), allowIsolated,
41                                dumpFlags);
42      }
43  // 发布的同时启动binder线程
44      static void publishAndJoinThreadPool(
45              bool allowIsolated = false,
46              int dumpFlags = IServiceManager::DUMP_FLAG_PRIORITY_DEFAULT) {
47          publish(allowIsolated, dumpFlags);
48          joinThreadPool();
49      }
50  
51      static void instantiate() { publish(); }
52  
53      static status_t shutdown() { return NO_ERROR; }
54  
55  private:
56      static void joinThreadPool() {
57          sp<ProcessState> ps(ProcessState::self());
58          ps->startThreadPool();
59          ps->giveThreadPoolName();
60          IPCThreadState::self()->joinThreadPool();
61      }
62  };
63  
64  
65  }; // namespace android
66  // ---------------------------------------------------------------------------
67  #endif // ANDROID_BINDER_SERVICE_H
68  
```

回过头来看 InstalldNativeService的 getServiceName()方法：

> /frameworks/native/cmds/installd/InstalldNativeService.h

```
 static char const* getServiceName() { return "installd"; }
```

确实是注册了名字为 installd 的服务。

## 其他功能方法

# start？？

mnt/sdcard/ sdcard/ storage/emulated/0

以上三个目录的内容都是一样的。为什么？

http://gityuan.com/2016/11/06/packagemanager/

http://gityuan.com/2016/11/13/android-installd/

https://blog.csdn.net/yiranfeng/article/details/103941371






