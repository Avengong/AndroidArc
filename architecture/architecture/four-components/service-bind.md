# 概述 bindService 启动流程

服务既可以通过启动方式又可以通过绑定，startService()分析完毕，接着分析bindService()。

# IServiceConnection 是一个aidl文件

那么肯定是binder服务接口。 服务端在哪里？ 客户端在哪里？ 客户端就在调用bindService的client端的，connection接口

```
 /** @hide */
23  oneway interface IServiceConnection {
24      @UnsupportedAppUsage
25      void connected(in ComponentName name, IBinder service, boolean dead);
26  }
27  
```

# 一、Activity.bindService()

> Activity.java

 ```
@Override
    public boolean bindService(Intent service, ServiceConnection conn,
            int flags) {
        return mBase.bindService(service, conn, flags);
    }
```

经过之前的分析，直接看 ContextImpl:

## 1.1 ContextImpl.bindService()

```
  @Override
    public boolean bindService(Intent service, ServiceConnection conn, int flags) {
        warnIfCallingFromSystemProcess();
        return bindServiceCommon(service, conn, flags, null, mMainThread.getHandler(), null,
                getUser());
    }

    @Override
    public boolean bindService(
            Intent service, int flags, Executor executor, ServiceConnection conn) {
        warnIfCallingFromSystemProcess();
        return bindServiceCommon(service, conn, flags, null, null, executor, getUser());
    }

```

## 1.2 bindServiceCommon()

> ContextImpl.java

```
private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags,
            String instanceName, Handler handler, Executor executor, UserHandle user) {
        // Keep this in sync with DevicePolicyManager.bindDeviceAdminServiceAsUser.
        IServiceConnection sd;
        // conn 不能为空
        if (conn == null) {
            throw new IllegalArgumentException("connection is null");
        }
        // handler 和  executor只能赋值一个
        if (handler != null && executor != null) {
            throw new IllegalArgumentException("Handler and Executor both supplied");
        }
        if (mPackageInfo != null) {
            if (executor != null) {
               // 这里有个关键点 getServiceDispatcher 
                sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), executor, flags);
            } else {
                sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(), handler, flags);
            }
        } else {
            throw new RuntimeException("Not supported in system context");
        }
        validateServiceIntent(service);
        try {
            // activity在系统端的 token
            IBinder token = getActivityToken();
            if (token == null && (flags&BIND_AUTO_CREATE) == 0 && mPackageInfo != null
                    && mPackageInfo.getApplicationInfo().targetSdkVersion
                    < android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
                flags |= BIND_WAIVE_PRIORITY;
            }
            service.prepareToLeaveProcess(this);
            // 调用 AMS 
            int res = ActivityManager.getService().bindIsolatedService(
                mMainThread.getApplicationThread(), getActivityToken(), service,
                service.resolveTypeIfNeeded(getContentResolver()),
                sd, flags, instanceName, getOpPackageName(), user.getIdentifier());
            if (res < 0) {
                throw new SecurityException(
                        "Not allowed to bind to service " + service);
            }
            return res != 0;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }

```

getServiceDispatcher 是啥？？ todo

### 1.2.1 getServiceDispatcher()

> LoadApk.java

```
 @UnsupportedAppUsage
  public final IServiceConnection getServiceDispatcher(ServiceConnection c,
          Context context, Handler handler, int flags) {
      return getServiceDispatcherCommon(c, context, handler, null, flags);
  }

  public final IServiceConnection getServiceDispatcher(ServiceConnection c,
          Context context, Executor executor, int flags) {
      return getServiceDispatcherCommon(c, context, null, executor, flags);
  }
```

### 1.2.2 getServiceDispatcherCommon()

> LoadApk.java

```
private IServiceConnection getServiceDispatcherCommon(ServiceConnection c,
            Context context, Handler handler, Executor executor, int flags) {
        synchronized (mServices) {
            LoadedApk.ServiceDispatcher sd = null;
            // mServices 是一个map。context为key，ArrayMap<ServiceConnection, LoadedApk.
            ServiceDispatcher>为value 。 因此，每个context都有自己的 ServiceConnection和ServiceDispatcher
            ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher> map = mServices.get(context);
            if (map != null) {
                if (DEBUG) Slog.d(TAG, "Returning existing dispatcher " + sd + " for conn " + c);
                sd = map.get(c);
            }
            if (sd == null) {
               // ServiceDispatcher的构造方法中会创建 InnerConnection(是ServiceDispatcher的静态内部类)，
                if (executor != null) {
                    sd = new ServiceDispatcher(c, context, executor, flags);
                } else {
                    sd = new ServiceDispatcher(c, context, handler, flags);
                }
                if (DEBUG) Slog.d(TAG, "Creating new dispatcher " + sd + " for conn " + c);
                if (map == null) {
                    map = new ArrayMap<>();
                    mServices.put(context, map);
                }
                map.put(c, sd);
            } else {
                sd.validate(context, handler, executor);
            }
            // 返回的是 InnerConnection 对象
            return sd.getIServiceConnection();
        }
    }
```

最终返回一个InnerConnection， 继承了 Stub 的Binder对象。

### ServiceDispatcher的构造方法

```
  ServiceDispatcher(ServiceConnection conn,
                Context context, Handler activityThread, int flags) {
            mIServiceConnection = new InnerConnection(this);
            mConnection = conn;
            mContext = context;
            mActivityThread = activityThread;
            mActivityExecutor = null;
            mLocation = new ServiceConnectionLeaked(null);
            mLocation.fillInStackTrace();
            mFlags = flags;
        }
```

### InnerConnection 静态内部类

>

```
private static class InnerConnection extends IServiceConnection.Stub {
            @UnsupportedAppUsage
            final WeakReference<LoadedApk.ServiceDispatcher> mDispatcher;

            InnerConnection(LoadedApk.ServiceDispatcher sd) {
                mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);
            }
            // 这个方法肯定是 服务进程调用的过来的。是的 就是AMS调过来的
            public void connected(ComponentName name, IBinder service, boolean dead)
                    throws RemoteException {
                LoadedApk.ServiceDispatcher sd = mDispatcher.get();
                if (sd != null) {
                    // 回调 connected()方法 
                    sd.connected(name, service, dead);
                }
            }
        }
```

可以看到

conn是app端的普通的对象。sd则是一个 IServiceConnection aidl定义的接口，包含提供的接口方法。
经过编译后，肯定会生成IServiceConnection接口，继承IInterface接口。 同时生成抽象的静态内部类Stub 继承Binder(本身实现IBinder)
，实现IServiceConnection接口。 stub是作为服务端使用。
在Stub类里面，还会有一个警惕内部Proxy，实现了IServiceConnection接口，内部有一个mRemote对象，指向了BinderProxy，最终指向c++层的BpBinder。

那么，在bindservice的过程中，stub在哪个端？ 在客户端。 服务 端什么调用connected()方法呢？

# 二、 AMS 端

## 2.1 bindIsolatedService()

> ActivityManagerService.java

```
public int bindIsolatedService(IApplicationThread caller, IBinder token, Intent service,
            String resolvedType, IServiceConnection connection, int flags, String instanceName,
            String callingPackage, int userId) throws TransactionTooLargeException {
            
        enforceNotIsolatedCaller("bindService");
        //... 一些安全检测

        synchronized(this) {
            return mServices.bindServiceLocked(caller, token, service,
                    resolvedType, connection, flags, instanceName, callingPackage, userId);
        }
    }
```

IBinder token: Activity在系统中Token对象。 疑问： service 必须要跟 activity 吗？？不是的，可选。 IServiceConnection
connection：客户端提供的 Binder 对象。

## 2.2 ActiveServices.bindServiceLocked()

> ActiveServices.java

```
int bindServiceLocked(IApplicationThread caller, IBinder token, Intent service,
            String resolvedType, final IServiceConnection connection, int flags,
            String instanceName, String callingPackage, final int userId)
            throws TransactionTooLargeException {
        // 获取调用者进程对象 ProcessRecord 
        final ProcessRecord callerApp = mAm.getRecordForAppLocked(caller);
        if (callerApp == null) {
            throw new SecurityException(
                    "Unable to find app for caller " + caller
                    + " (pid=" + Binder.getCallingPid()
                    + ") when binding service " + service);
        }

        ActivityServiceConnectionsHolder<ConnectionRecord> activity = null;
        // 如果token不为null，则表示是activity中bindservice的
        if (token != null) {
           // 根据传递过来的token，获取activity对象，为空则表示在系统端没有注册，是非法的，直接结束。
            activity = mAm.mAtmInternal.getServiceConnectionsHolder(token);
            if (activity == null) {
                Slog.w(TAG, "Binding with unknown activity: " + token);
                return 0;
            }
        }

        int clientLabel = 0;
        PendingIntent clientIntent = null;
        final boolean isCallerSystem = callerApp.info.uid == Process.SYSTEM_UID;

        if (isCallerSystem) {
           //... 不是系统进程，不走这里
        }
         // 下面是一系列的 flags 校验是否设置了这些flag？？： 
        if ((flags&Context.BIND_TREAT_LIKE_ACTIVITY) != 0) {
            mAm.enforceCallingPermission(android.Manifest.permission.MANAGE_ACTIVITY_STACKS,
                    "BIND_TREAT_LIKE_ACTIVITY");
        }

        if ((flags & Context.BIND_SCHEDULE_LIKE_TOP_APP) != 0 && !isCallerSystem) {
            throw new SecurityException("Non-system caller (pid=" + Binder.getCallingPid()
                    + ") set BIND_SCHEDULE_LIKE_TOP_APP when binding service " + service);
        }

        if ((flags & Context.BIND_ALLOW_WHITELIST_MANAGEMENT) != 0 && !isCallerSystem) {
            throw new SecurityException(
                    "Non-system caller " + caller + " (pid=" + Binder.getCallingPid()
                    + ") set BIND_ALLOW_WHITELIST_MANAGEMENT when binding service " + service);
        }

        if ((flags & Context.BIND_ALLOW_INSTANT) != 0 && !isCallerSystem) {
            throw new SecurityException(
                    "Non-system caller " + caller + " (pid=" + Binder.getCallingPid()
                            + ") set BIND_ALLOW_INSTANT when binding service " + service);
        }

        if ((flags & Context.BIND_ALLOW_BACKGROUND_ACTIVITY_STARTS) != 0) {
            mAm.enforceCallingPermission(
                    android.Manifest.permission.START_ACTIVITIES_FROM_BACKGROUND,
                    "BIND_ALLOW_BACKGROUND_ACTIVITY_STARTS");
        }
        // callerFg 是否前台
        final boolean callerFg = callerApp.setSchedGroup != ProcessList.SCHED_GROUP_BACKGROUND;
        final boolean isBindExternal = (flags & Context.BIND_EXTERNAL_SERVICE) != 0;
        final boolean allowInstant = (flags & Context.BIND_ALLOW_INSTANT) != 0;
         // 内部通过PMS解析获取 ServiceRecord 信息
        ServiceLookupResult res =
            retrieveServiceLocked(service, instanceName, resolvedType, callingPackage,
                    Binder.getCallingPid(), Binder.getCallingUid(), userId, true,
                    callerFg, isBindExternal, allowInstant);
        if (res == null) {
            return 0;
        }
        if (res.record == null) {
            return -1;
        }
        // 得到 ServiceRecord 对象 
        ServiceRecord s = res.record;

        boolean permissionsReviewRequired = false;

        // If permissions need a review before any of the app components can run,
        // we schedule binding to the service but do not start its process, then
        // we launch a review activity to which is passed a callback to invoke
        // when done to start the bound service's process to completing the binding.
        if (mAm.getPackageManagerInternalLocked().isPermissionsReviewRequired(
                s.packageName, s.userId)) {

            permissionsReviewRequired = true;

            // Show a permission review UI only for binding from a foreground app
            if (!callerFg) {
                return 0;
            }

            final ServiceRecord serviceRecord = s;
            final Intent serviceIntent = service;

            RemoteCallback callback = new RemoteCallback(
                    new RemoteCallback.OnResultListener() {
                @Override
                public void onResult(Bundle result) {
                    synchronized(mAm) {
                        final long identity = Binder.clearCallingIdentity();
                        try {
                            if (!mPendingServices.contains(serviceRecord)) {
                                return;
                            }
                            // If there is still a pending record, then the service
                            // binding request is still valid, so hook them up. We
                            // proceed only if the caller cleared the review requirement
                            // otherwise we unbind because the user didn't approve.
                            if (!mAm.getPackageManagerInternalLocked()
                                    .isPermissionsReviewRequired(
                                            serviceRecord.packageName,
                                            serviceRecord.userId)) {
                                try {
                                    bringUpServiceLocked(serviceRecord,
                                            serviceIntent.getFlags(),
                                            callerFg, false, false);
                                } catch (RemoteException e) {
                                    /* ignore - local call */
                                }
                            } else {
                                unbindServiceLocked(connection);
                            }
                        } finally {
                            Binder.restoreCallingIdentity(identity);
                        }
                    }
                }
            });

            final Intent intent = new Intent(Intent.ACTION_REVIEW_PERMISSIONS);
            intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK
                    | Intent.FLAG_ACTIVITY_MULTIPLE_TASK
                    | Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS);
            intent.putExtra(Intent.EXTRA_PACKAGE_NAME, s.packageName);
            intent.putExtra(Intent.EXTRA_REMOTE_CALLBACK, callback);
           
            // mAm 是AMS对象 
            mAm.mHandler.post(new Runnable() {
                @Override
                public void run() {
                    mAm.mContext.startActivityAsUser(intent, new UserHandle(userId));
                }
            });
        }

        final long origId = Binder.clearCallingIdentity();

        try {
            if (unscheduleServiceRestartLocked(s, callerApp.info.uid, false)) {

            if ((flags&Context.BIND_AUTO_CREATE) != 0) {
                s.lastActivity = SystemClock.uptimeMillis();
                if (!s.hasAutoCreateConnections()) {
                    // This is the first binding, let the tracker know.
                    ServiceState stracker = s.getTracker();
                    if (stracker != null) {
                        stracker.setBound(true, mAm.mProcessStats.getMemFactorLocked(),
                                s.lastActivity);
                    }
                }
            }

            if ((flags & Context.BIND_RESTRICT_ASSOCIATIONS) != 0) {
                mAm.requireAllowedAssociationsLocked(s.appInfo.packageName);
            }

            mAm.startAssociationLocked(callerApp.uid, callerApp.processName,
                    callerApp.getCurProcState(), s.appInfo.uid, s.appInfo.longVersionCode,
                    s.instanceName, s.processName);
            // Once the apps have become associated, if one of them is caller is ephemeral
            // the target app should now be able to see the calling app
            mAm.grantEphemeralAccessLocked(callerApp.userId, service,
                    UserHandle.getAppId(s.appInfo.uid), UserHandle.getAppId(callerApp.uid));
            // 得到  AppBindRecord 对象，表示该服务对应的其中一个客户端(一个服务可以被多个客户端绑定)
            AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);
            // 创建 ConnectionRecord 对象，封装从绑定服务的发起端传递过来的conn
            ConnectionRecord c = new ConnectionRecord(b, activity,
                    connection, flags, clientLabel, clientIntent,
                    callerApp.uid, callerApp.processName, callingPackage);
            // 获取发起端的 Binder连接对象
            IBinder binder = connection.asBinder();
            // 添加到 serviceRecord中的 connections。 connections 
            是个map。binder为key，list<ConnectionRecord>为value。 
            s.addConnection(binder, c);
            添加到 AppBindRecord中的 connections
            b.connections.add(c);
            if (activity != null) {
                activity.addConnection(c);
            }
            b.client.connections.add(c);
            c.startAssociationIfNeeded();
            if ((c.flags&Context.BIND_ABOVE_CLIENT) != 0) {
                b.client.hasAboveClient = true;
            }
            if ((c.flags&Context.BIND_ALLOW_WHITELIST_MANAGEMENT) != 0) {
                s.whitelistManager = true;
            }
            if ((flags & Context.BIND_ALLOW_BACKGROUND_ACTIVITY_STARTS) != 0) {
                s.setHasBindingWhitelistingBgActivityStarts(true);
            }
            if (s.app != null) {
                updateServiceClientActivitiesLocked(s.app, c, true);
            }
            ArrayList<ConnectionRecord> clist = mServiceConnections.get(binder);
            if (clist == null) {
                clist = new ArrayList<>();
                mServiceConnections.put(binder, clist);
            }
            clist.add(c);
             // 如果设置了BIND_AUTO_CREATE 标志
            if ((flags&Context.BIND_AUTO_CREATE) != 0) {
                s.lastActivity = SystemClock.uptimeMillis();
                // 开始启动 service
                if (bringUpServiceLocked(s, service.getFlags(), callerFg, false,
                        permissionsReviewRequired) != null) {
                    return 0;
                }
            }

            if (s.app != null) {
                if ((flags&Context.BIND_TREAT_LIKE_ACTIVITY) != 0) {
                    s.app.treatLikeActivity = true;
                }
                if (s.whitelistManager) {
                    s.app.whitelistManager = true;
                }
                // This could have made the service more important.
                mAm.updateLruProcessLocked(s.app,
                        (callerApp.hasActivitiesOrRecentTasks() && s.app.hasClientActivities())
                                || (callerApp.getCurProcState() <= ActivityManager.PROCESS_STATE_TOP
                                        && (flags & Context.BIND_TREAT_LIKE_ACTIVITY) != 0),
                        b.client);
                mAm.updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_BIND_SERVICE);
            }

         

            if (s.app != null && b.intent.received) {
                // Service is already running, so we can immediately
                // publish the connection.
                try {
                    // 服务已经发布，通过binder调用发起端的connected()方法
                    c.conn.connected(s.name, b.intent.binder, false);
                } catch (Exception e) {
                    Slog.w(TAG, "Failure sending service " + s.shortInstanceName
                            + " to connection " + c.conn.asBinder()
                            + " (in " + c.binding.client.processName + ")", e);
                }

                // If this is the first app connected back to this binding,
                // and the service had previously asked to be told when
                // rebound, then do so.
                if (b.intent.apps.size() == 1 && b.intent.doRebind) {
                    // 如果已经bind过，则回调onReBind
                    requestServiceBindingLocked(s, b.intent, callerFg, true);
                }
            } else if (!b.intent.requested) {
               // 最终回调 onBind() 方法
                requestServiceBindingLocked(s, b.intent, callerFg, false);
            }

            getServiceMapLocked(s.userId).ensureNotStartingBackgroundLocked(s);

        } finally {
            Binder.restoreCallingIdentity(origId);
        }
        return 1;
    }
``` 

- 内部通过PMS解析获取 ServiceRecord 信息
- 根据callerApp来获取发起端信息--AppBindRecord： intent、ServiceRecord、ProcessRecord
- 如果设置了标记：BIND_AUTO_CREATE，则调用 bringUpServiceLocked() 拉起服务，一般bindservice()方法中flag设置
- 如果服务已经发布，通过binder调用发起端的connected()方法
- 回调 onReBind() 方法

## 2.3 ActiveServices.bringUpServiceLocked()

```
 private String bringUpServiceLocked(ServiceRecord r, int intentFlags, boolean execInFg,
            boolean whileRestarting, boolean permissionsReviewRequired)
            throws TransactionTooLargeException {
        if (r.app != null && r.app.thread != null) {
            // 如果service已经启动，则直接回调onStartCommand()方法，传入的是false
            sendServiceArgsLocked(r, execInFg, false);
            return null;
        }

        if (!whileRestarting && mRestartingServices.contains(r)) {
            // If waiting for a restart, then do nothing.
            return null;
        }

        // We are now bringing the service up, so no longer in the
        // restarting state.
        if (mRestartingServices.remove(r)) {
            clearRestartingIfNeededLocked(r);
        }

        // Make sure this service is no longer considered delayed, we are starting it now.
        if (r.delayed) {
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "REM FR DELAY LIST (bring up): " + r);
            getServiceMapLocked(r.userId).mDelayedStartList.remove(r);
            r.delayed = false;
        }

        // Make sure that the user who owns this service is started.  If not,
        // we don't want to allow it to run.
        if (!mAm.mUserController.hasStartedUserState(r.userId)) {
            String msg = "Unable to launch app "
                    + r.appInfo.packageName + "/"
                    + r.appInfo.uid + " for service "
                    + r.intent.getIntent() + ": user " + r.userId + " is stopped";
            Slog.w(TAG, msg);
            bringDownServiceLocked(r);
            return msg;
        }

        // Service is now being launched, its package can't be stopped.
        try {
            AppGlobals.getPackageManager().setPackageStoppedState(
                    r.packageName, false, r.userId);
        } catch (RemoteException e) {
        } catch (IllegalArgumentException e) {
            Slog.w(TAG, "Failed trying to unstop package "
                    + r.packageName + ": " + e);
        }

        final boolean isolated = (r.serviceInfo.flags&ServiceInfo.FLAG_ISOLATED_PROCESS) != 0;
        final String procName = r.processName;
        HostingRecord hostingRecord = new HostingRecord("service", r.instanceName);
        ProcessRecord app;

        if (!isolated) {
            app = mAm.getProcessRecordLocked(procName, r.appInfo.uid, false);
            if (DEBUG_MU) Slog.v(TAG_MU, "bringUpServiceLocked: appInfo.uid=" + r.appInfo.uid
                        + " app=" + app);
            if (app != null && app.thread != null) {
                try {
                    app.addPackage(r.appInfo.packageName, r.appInfo.longVersionCode, mAm.mProcessStats);
                    // 
                    realStartServiceLocked(r, app, execInFg);
                    return null;
                } catch (TransactionTooLargeException e) {
                    throw e;
                } catch (RemoteException e) {
                    Slog.w(TAG, "Exception when starting service " + r.shortInstanceName, e);
                }

                // If a dead object exception was thrown -- fall through to
                // restart the application.
            }
        } else {
            // If this service runs in an isolated process, then each time
            // we call startProcessLocked() we will get a new isolated
            // process, starting another process if we are currently waiting
            // for a previous process to come up.  To deal with this, we store
            // in the service any current isolated process it is running in or
            // waiting to have come up.
            app = r.isolatedProc;
            if (WebViewZygote.isMultiprocessEnabled()
                    && r.serviceInfo.packageName.equals(WebViewZygote.getPackageName())) {
                hostingRecord = HostingRecord.byWebviewZygote(r.instanceName);
            }
            if ((r.serviceInfo.flags & ServiceInfo.FLAG_USE_APP_ZYGOTE) != 0) {
                hostingRecord = HostingRecord.byAppZygote(r.instanceName, r.definingPackageName,
                        r.definingUid);
            }
        }

        // Not running -- get it started, and enqueue this service record
        // to be executed when the app comes up.
        //如果进程没有拉起，则先拉起进程
        if (app == null && !permissionsReviewRequired) {
        
            if ((app=mAm.startProcessLocked(procName, r.appInfo, true, intentFlags,
                    hostingRecord, false, isolated, false)) == null) {
                String msg = "Unable to launch app "
                        + r.appInfo.packageName + "/"
                        + r.appInfo.uid + " for service "
                        + r.intent.getIntent() + ": process is bad";
                Slog.w(TAG, msg);
                bringDownServiceLocked(r);
                return msg;
            }
            if (isolated) {
                r.isolatedProc = app;
            }
        }

        if (r.fgRequired) {
            mAm.tempWhitelistUidLocked(r.appInfo.uid,
                    SERVICE_START_FOREGROUND_TIMEOUT, "fg-service-launch");
        }

        if (!mPendingServices.contains(r)) {
            mPendingServices.add(r);
        }

        if (r.delayedStop) {
            // Oh and hey we've already been asked to stop!
            r.delayedStop = false;
            if (r.startRequested) {
                if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE,
                        "Applying delayed stop (in bring up): " + r);
                stopServiceLocked(r);
            }
        }

        return null;
    }
```

调用 realStartServiceLocked() .

### 2.3.1 realStartServiceLocked()

```
 private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
        if (app.thread == null) {
            throw new RemoteException();
        }
        r.setProcess(app);
        r.restartTime = r.lastActivity = SystemClock.uptimeMillis();

        final boolean newService = app.services.add(r);
        // 发送延迟消息
        bumpServiceExecutingLocked(r, execInFg, "create");
        mAm.updateLruProcessLocked(app, false, null);
        updateServiceForegroundLocked(r.app, /* oomAdj= */ false);
        mAm.updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_START_SERVICE);

        boolean created = false;
        try {
            if (LOG_SERVICE_START_STOP) {
                String nameTerm;
                int lastPeriod = r.shortInstanceName.lastIndexOf('.');
                nameTerm = lastPeriod >= 0 ? r.shortInstanceName.substring(lastPeriod)
                        : r.shortInstanceName;
                EventLogTags.writeAmCreateService(
                        r.userId, System.identityHashCode(r), nameTerm, r.app.uid, r.app.pid);
            }
            StatsLog.write(StatsLog.SERVICE_LAUNCH_REPORTED, r.appInfo.uid, r.name.getPackageName(),
                    r.name.getClassName());
            synchronized (r.stats.getBatteryStats()) {
                r.stats.startLaunchedLocked();
            }
            mAm.notifyPackageUse(r.serviceInfo.packageName,
                                 PackageManager.NOTIFY_PACKAGE_USE_SERVICE);
            app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
            // 执行onCreate()流程
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackage(r.serviceInfo.applicationInfo),
                    app.getReportedProcState());
            r.postNotification();
            created = true;
        } catch (DeadObjectException e) {
            Slog.w(TAG, "Application dead when creating service " + r);
            mAm.appDiedLocked(app);
            throw e;
        } finally {
            if (!created) {
                // Keep the executeNesting count accurate.
                final boolean inDestroying = mDestroyingServices.contains(r);
                serviceDoneExecutingLocked(r, inDestroying, inDestroying);

                // Cleanup.
                if (newService) {
                    app.services.remove(r);
                    r.setProcess(null);
                }

                // Retry.
                if (!inDestroying) {
                    scheduleServiceRestartLocked(r, false);
                }
            }
        }

        if (r.whitelistManager) {
            app.whitelistManager = true;
        }
        // 开始执行 bind流程
        requestServiceBindingsLocked(r, execInFg);

        updateServiceClientActivitiesLocked(app, null, true);

        if (newService && created) {
            app.addBoundClientUidsOfNewService(r);
        }

        // If the service is in the started state, and there are no
        // pending arguments, then fake up one so its onStartCommand() will
        // be called.
        
        // 这里重要，因为r.callStart=false。
        if (r.startRequested && r.callStart && r.pendingStarts.size() == 0) {
            r.pendingStarts.add(new ServiceRecord.StartItem(r, false, r.makeNextStartId(),
                    null, null, 0));
        }
        // 回调onStartCommand()
        sendServiceArgsLocked(r, execInFg, true);

        if (r.delayed) {
            if (DEBUG_DELAYED_STARTS) Slog.v(TAG_SERVICE, "REM FR DELAY LIST (new proc): " + r);
            getServiceMapLocked(r.userId).mDelayedStartList.remove(r);
            r.delayed = false;
        }

        if (r.delayedStop) {
            // Oh and hey we've already been asked to stop!
            r.delayedStop = false;
            if (r.startRequested) {
                stopServiceLocked(r);
            }
        }
    }
```

- 发送超时消息。 前台服务为20s，后天服务为200s
- scheduleCreateService() 回调app端创建service对象，执行onCreate()
- 回调onBind()方法
- 回调onStartCommand() 方法

小结一下： 绑定的过程中，发起端的intent、conn 作为服务端传递到AMS AMS 开始根据inten，靠PMS解析得到serviceinfo，最终得到ServiceRecord。
开始启动service，执行创建流程，回调onCreate() 方法 app端 onCreate方法完毕后，告诉AMS服务创建完毕。 AMS端继续执行bind流程，回调onBind()
方法，根据onBind()返回的IBinder服务对象，publish到AMS端 AMS端，在回调conn，把服务端的 IBinder对象发送到发起端，回调onServiceConnected()
方法， 至此，客户端也就是发起端，有了服务提供的IBinder对象，强转成IHelloService接口，调用方法IBinder服务。

这种通过service提供IBinder给客户端的方式，比注册到ServiceManager，

onCreate流程跟startService一样。 那么onStartCommand()方法会被回调吗？ 答：不会。因为

### 2.3。2 scheduleCreateService()

该过程与startService一样。

### 2.3.3 requestServiceBindingsLocked() 绑定流程

```
private final void requestServiceBindingsLocked(ServiceRecord r, boolean execInFg)
            throws TransactionTooLargeException {
            // 遍历 bindings 集合 
        for (int i=r.bindings.size()-1; i>=0; i--) {
            IntentBindRecord ibr = r.bindings.valueAt(i);
            if (!requestServiceBindingLocked(r, ibr, execInFg, false)) {
                break;
            }
        }
    }

```

bindings 的作用？ 所有绑定到该服务的

### 2.3.4 requestServiceBindingLocked()

```
  private final boolean requestServiceBindingLocked(ServiceRecord r, IntentBindRecord i,
            boolean execInFg, boolean rebind) throws TransactionTooLargeException {
        if (r.app == null || r.app.thread == null) {
            // If service is not currently running, can't yet bind.
            return false;
        }
      
        if ((!i.requested || rebind) && i.apps.size() > 0) {
            try {
               // 开始设置绑定超时
                bumpServiceExecutingLocked(r, execInFg, "bind");
                r.app.forceProcessStateUpTo(ActivityManager.PROCESS_STATE_SERVICE);
                // 开始调用scheduleBindService()方法 
                r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind,
                        r.app.getReportedProcState());
                if (!rebind) {
                    i.requested = true;
                }
                i.hasBound = true;
                i.doRebind = false;
            }
            ...
        }
        return true;
    }
```

回调到 app 端的 onBind() 方法：

# app端 scheduleBindService()

```
  public final void scheduleBindService(IBinder token, Intent intent,
                boolean rebind, int processState) {
            updateProcessState(processState, false);
            BindServiceData s = new BindServiceData();
            s.token = token;
            s.intent = intent;
            s.rebind = rebind;

            if (DEBUG_SERVICE)
                Slog.v(TAG, "scheduleBindService token=" + token + " intent=" + intent + " uid="
                        + Binder.getCallingUid() + " pid=" + Binder.getCallingPid());
            sendMessage(H.BIND_SERVICE, s);
        }
```

## H.BIND_SERVICE消息

```
 case BIND_SERVICE:
            Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "serviceBind");
            handleBindService((BindServiceData)msg.obj);
            Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            break;
```

## handleBindService()

```
  private void handleBindService(BindServiceData data) {
        Service s = mServices.get(data.token);
        if (DEBUG_SERVICE)
            Slog.v(TAG, "handleBindService s=" + s + " rebind=" + data.rebind);
        if (s != null) {
            try {
                data.intent.setExtrasClassLoader(s.getClassLoader());
                data.intent.prepareToEnterProcess();
                try {
                    if (!data.rebind) {
                       // 没有bind过，则调用onBind()方法；同时告知AMS去发布服务
                        IBinder binder = s.onBind(data.intent);
                        ActivityManager.getService().publishService(
                                data.token, data.intent, binder);
                    } else {
                       // bind过，则调用onReBind()方法，同时告知AMS 执行完毕
                        s.onRebind(data.intent);
                        ActivityManager.getService().serviceDoneExecuting(
                                data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
                    }
                } ...
            } ...
        }
    }
```

- 没有bind过，则调用onBind()方法；同时告知AMS去发布服务
- bind过，则调用onReBind()方法，同时告知AMS 执行完毕

# AMS. publishService()

```
 public void publishService(IBinder token, Intent intent, IBinder service) {
        // Refuse possible leaked file descriptors

        synchronized(this) {
            if (!(token instanceof ServiceRecord)) {
                throw new IllegalArgumentException("Invalid service token");
            }
            mServices.publishServiceLocked((ServiceRecord)token, intent, service);
        }
    }

```

## ActiveServices.publishServiceLocked()

> ActiveServices.java

```
void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
        final long origId = Binder.clearCallingIdentity();
        try {
            if (r != null) {
                Intent.FilterComparison filter
                        = new Intent.FilterComparison(intent);
                IntentBindRecord b = r.bindings.get(filter);
                if (b != null && !b.received) {
                    b.binder = service; // 服务service 传递过来的Binder对象
                    b.requested = true;
                    b.received = true;
                    ArrayMap<IBinder, ArrayList<ConnectionRecord>> connections = r.getConnections();
                    for (int conni = connections.size() - 1; conni >= 0; conni--) {
                        ArrayList<ConnectionRecord> clist = connections.valueAt(conni);
                        for (int i=0; i<clist.size(); i++) {
                            ConnectionRecord c = clist.get(i);
                            // 过滤掉不匹配的intent
                            if (!filter.equals(c.binding.intent.intent)) {
                                if (DEBUG_SERVICE) Slog.v(
                                        TAG_SERVICE, "Not publishing to: " + c);
                                if (DEBUG_SERVICE) Slog.v(
                                        TAG_SERVICE, "Bound intent: " + c.binding.intent.intent);
                                if (DEBUG_SERVICE) Slog.v(
                                        TAG_SERVICE, "Published intent: " + intent);
                                continue;
                            }
                            if (DEBUG_SERVICE) Slog.v(TAG_SERVICE, "Publishing to: " + c);
                            try {
                            // 回调app端的 IServiceConnection的 connected()方法
                                c.conn.connected(r.name, service, false);
                            } catch (Exception e) {
                                Slog.w(TAG, "Failure sending service " + r.shortInstanceName
                                      + " to connection " + c.conn.asBinder()
                                      + " (in " + c.binding.client.processName + ")", e);
                            }
                        }
                    }
                }

                serviceDoneExecutingLocked(r, mDestroyingServices.contains(r), false);
            }
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }

```

# APP端 IServiceConnection

是 LoadApk中的静态内部类 InnerConnection

```
private static class InnerConnection extends IServiceConnection.Stub {
            @UnsupportedAppUsage
            final WeakReference<LoadedApk.ServiceDispatcher> mDispatcher;

            InnerConnection(LoadedApk.ServiceDispatcher sd) {
                mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);
            }
            // 这个方法肯定是 服务进程调用的过来的。是的 就是AMS调过来的
            public void connected(ComponentName name, IBinder service, boolean dead)
                    throws RemoteException {
                LoadedApk.ServiceDispatcher sd = mDispatcher.get();
                if (sd != null) {
                    // 回调 connected()方法 
                    sd.connected(name, service, dead);
                }
            }
        }
```

## ServiceDispatcher.connected()

```

 public void connected(ComponentName name, IBinder service, boolean dead) {
            if (mActivityExecutor != null) {
                mActivityExecutor.execute(new RunConnection(name, service, 0, dead));
            } else if (mActivityThread != null) {
                mActivityThread.post(new RunConnection(name, service, 0, dead));
            } else {
                doConnected(name, service, dead);
            }
        }
```

## doConnected()

```
 public void doConnected(ComponentName name, IBinder service, boolean dead) {
            ServiceDispatcher.ConnectionInfo old;
            ServiceDispatcher.ConnectionInfo info;

            synchronized (this) {
                if (mForgotten) {
                    // We unbound before receiving the connection; ignore
                    // any connection received.
                    return;
                }
                old = mActiveConnections.get(name);
                if (old != null && old.binder == service) {
                    // Huh, already have this one.  Oh well!
                    return;
                }

                if (service != null) {
                    // A new service is being connected... set it all up.
                    info = new ConnectionInfo();
                    info.binder = service;
                    info.deathMonitor = new DeathMonitor(name, service);
                    try {
                        service.linkToDeath(info.deathMonitor, 0);
                        mActiveConnections.put(name, info);
                    } catch (RemoteException e) {
                        // This service was dead before we got it...  just
                        // don't do anything with it.
                        mActiveConnections.remove(name);
                        return;
                    }

                } else {
                    // The named service is being disconnected... clean up.
                    mActiveConnections.remove(name);
                }

                if (old != null) {
                    old.binder.unlinkToDeath(old.deathMonitor, 0);
                }
            }

            // If there was an old service, it is now disconnected.
            if (old != null) {
                mConnection.onServiceDisconnected(name);
            }
            if (dead) {
                mConnection.onBindingDied(name);
            }
            // If there is a new viable service, it is now connected.
            if (service != null) {
               // 通知 onServiceConnected
                mConnection.onServiceConnected(name, service);
            } else {
                // The binding machinery worked, but the remote returned null from onBind().
                mConnection.onNullBinding(name);
            }
        }

```









