system server进程启动后，那launch进程又是什么时候启动的呢？

谁来启动的？

启动过程是怎样的？

想理解launch进程的启动，因为涉及到了AMS PMS 等一系列服务，还有binder通信，因此先搞清楚这三者的原理才能继续往下走。

# Launch 启动流程

# 两个阶段

## AMS 向 Zygote发起socket通信过程

## Zygote 进程处理 socket请求过程

## 执行 Launch 进程逻辑过程

在 AMS 中的 systemReady() 方法中 会调用 startHomeOnAllDisplays() 方法来启动 launch进程。

```
mAtmInternal.startHomeOnAllDisplays(currentUserId, "systemReady");
```

mAtmInternal 是 ActivityTaskManagerService的 静态内部类 LocalService 对象。

```
 @Override
        public boolean startHomeOnAllDisplays(int userId, String reason) {
            synchronized (mGlobalLock) {
                return mRootActivityContainer.startHomeOnAllDisplays(userId, reason);
            }
        }
```

RootActivityContainer activity容器的根节点，用来分担ActivityStackSuperVisor的一部分工作。

> RootActivityContainer.java

```
 boolean startHomeOnAllDisplays(int userId, String reason) {
        boolean homeStarted = false;
        for (int i = mActivityDisplays.size() - 1; i >= 0; i--) {
            // 
            final int displayId = mActivityDisplays.get(i).mDisplayId;
            //reason: "systemReady";  屏幕id: 屏幕id
            homeStarted |= startHomeOnDisplay(userId, reason, displayId);
        }
        return homeStarted;
    }
```

ActivityDisplay 表示屏幕信息。 主屏幕、外接屏幕、虚拟屏幕。可暂时理解为只有一个屏幕。

# startHomeOnDisplay：

默认采用 DEFAULT_DISPLAY=0 ，作为主屏幕。 如果launch界面想要展示在 Secondary 屏幕上， 则需要设置category SECONDARY_HOME。

```
boolean startHomeOnDisplay(int userId, String reason, int displayId, boolean allowInstrumenting,
            boolean fromHomeKey) {
        // Fallback to top focused display if the displayId is invalid.
        if (displayId == INVALID_DISPLAY) {
            displayId = getTopDisplayFocusedStack().mDisplayId;
        }

        Intent homeIntent = null;
        ActivityInfo aInfo = null;
        //DEFAULT_DISPLAY=0 默认屏幕
        if (displayId == DEFAULT_DISPLAY) { 
            // 获取home intent
            homeIntent = mService.getHomeIntent();
            // 通过pms获取 ActivityInfo 
            aInfo = resolveHomeActivity(userId, homeIntent);
        } else if (shouldPlaceSecondaryHomeOnDisplay(displayId)) {
            Pair<ActivityInfo, Intent> info = resolveSecondaryHomeActivity(userId, displayId);
            aInfo = info.first;
            homeIntent = info.second;
        }
        if (aInfo == null || homeIntent == null) {
            return false;
        }

        if (!canStartHomeOnDisplay(aInfo, displayId, allowInstrumenting)) {
            return false;
        }

        // Updates the home component of the intent.
        
        // 更新 home component 信息
        homeIntent.setComponent(new ComponentName(aInfo.applicationInfo.packageName, aInfo.name));
        homeIntent.setFlags(homeIntent.getFlags() | FLAG_ACTIVITY_NEW_TASK);
        
        // Updates the extra information of the intent.
        if (fromHomeKey) {
            homeIntent.putExtra(WindowManagerPolicy.EXTRA_FROM_HOME_KEY, true);
        }
        // Update the reason for ANR debugging to verify if the user activity is the one that
        // actually launched.
        final String myReason = reason + ":" + userId + ":" + UserHandle.getUserId(
                aInfo.applicationInfo.uid) + ":" + displayId;
                
        // 开始启动        
        mService.getActivityStartController().startHomeActivity(homeIntent, aInfo, myReason,
                displayId);
        return true;
    }

```

## getHomeIntent()

```
 Intent getHomeIntent() {
        //mTopAction = Intent.ACTION_MAIN;
        Intent intent = new Intent(mTopAction, mTopData != null ? Uri.parse(mTopData) : null);
        intent.setComponent(mTopComponent);
        intent.addFlags(Intent.FLAG_DEBUG_TRIAGED_MISSING);
        if (mFactoryTest != FactoryTest.FACTORY_TEST_LOW_LEVEL) {
            // 表示是开机后的第一个activity
            intent.addCategory(Intent.CATEGORY_HOME);
        }
        return intent;
    }
```

## resolveHomeActivity()

```
ActivityInfo resolveHomeActivity(int userId, Intent homeIntent) {
        final int flags = ActivityManagerService.STOCK_PM_FLAGS;
        final ComponentName comp = homeIntent.getComponent();
        ActivityInfo aInfo = null;
        try {
            if (comp != null) {
                // Factory test.
                aInfo = AppGlobals.getPackageManager().getActivityInfo(comp, flags, userId);
            } else {
            // 走这个分支？？
                final String resolvedType =
                        homeIntent.resolveTypeIfNeeded(mService.mContext.getContentResolver());
                        
                final ResolveInfo info = AppGlobals.getPackageManager()
                        .resolveIntent(homeIntent, resolvedType, flags, userId);
                if (info != null) {
                    aInfo = info.activityInfo;
                }
            }
        } catch (RemoteException e) {
            // ignore
        }

        if (aInfo == null) {
            Slog.wtf(TAG, "No home screen found for " + homeIntent, new Throwable());
            return null;
        }

        aInfo = new ActivityInfo(aInfo); // 创建
        aInfo.applicationInfo = mService.getAppInfoForUser(aInfo.applicationInfo, userId);
        return aInfo;
    }
```

> ActivityStartController.java

```
 void startHomeActivity(Intent intent, ActivityInfo aInfo, String reason, int displayId) {
        final ActivityOptions options = ActivityOptions.makeBasic();
        options.setLaunchWindowingMode(WINDOWING_MODE_FULLSCREEN);
        if (!ActivityRecord.isResolverActivity(aInfo.name)) {
            // The resolver activity shouldn't be put in home stack because when the foreground is
            // standard type activity, the resolver activity should be put on the top of current
            // foreground instead of bring home stack to front.
            options.setLaunchActivityType(ACTIVITY_TYPE_HOME);
        }
        options.setLaunchDisplayId(displayId);
        mLastHomeActivityStartResult = obtainStarter(intent, "startHomeActivity: " + reason)
                .setOutActivity(tmpOutRecord)
                .setCallingUid(0)
                .setActivityInfo(aInfo)
                .setActivityOptions(options.toBundle())
                .execute();
        mLastHomeActivityStartRecord = tmpOutRecord[0];
        final ActivityDisplay display =
                mService.mRootActivityContainer.getActivityDisplay(displayId);
        final ActivityStack homeStack = display != null ? display.getHomeStack() : null;
        if (homeStack != null && homeStack.mInResumeTopActivity) {
            // If we are in resume section already, home activity will be initialized, but not
            // resumed (to avoid recursive resume) and will stay that way until something pokes it
            // again. We need to schedule another resume.
            mSupervisor: ActivityStackSupervisor类型
            mSupervisor.scheduleResumeTopActivities();
        }
    }
```

最终调用 ActivityStackSupervisor 的 scheduleResumeTopActivities()

> ActivityStackSupervisor.java

```
 final void scheduleResumeTopActivities() {
        if (!mHandler.hasMessages(RESUME_TOP_ACTIVITY_MSG)) {
            mHandler.sendEmptyMessage(RESUME_TOP_ACTIVITY_MSG);
        }
    }
```

往 mHandler 发送了一个消息。 那么 mHandler 是什么呢？

在AMS的构造方法中，会调用 atm的 initialize()方法： 传入的是 DisplayThread 线程的looper。

> AMS

```
mActivityTaskManager.initialize(mIntentFirewall, mPendingIntentController,
                DisplayThread.get().getLooper())
```

在 atm 中会创建 mH、stackSuperVisor：
> atm

```
 public void initialize(IntentFirewall intentFirewall, PendingIntentController intentController,
            Looper looper) {
             mH = new H(looper);
             ...
             mStackSupervisor = createStackSupervisor();
             ...
 }
```

```
 protected ActivityStackSupervisor createStackSupervisor() {
        final ActivityStackSupervisor supervisor = new ActivityStackSupervisor(this, mH.getLooper());
        supervisor.initialize();
        return supervisor;
    }
    
```

这里传入了 mH的looper，其实就是 DisplayThread 线程的looper。

```
 public ActivityStackSupervisor(ActivityTaskManagerService service, Looper looper) {
        mService = service;
        mLooper = looper;
        mHandler = new ActivityStackSupervisorHandler(looper);
    }
```

因此 ActivityStackSupervisor 的 mHandler 发送的消息最终都是 DisplayThread 线程来处理。

# ActivityStackSupervisorHandler 的 handleMessage()

```
 @Override
        public void handleMessage(Message msg) {
                 ...
                case RESUME_TOP_ACTIVITY_MSG: {
                    synchronized (mService.mGlobalLock) {
                        mRootActivityContainer.resumeFocusedStacksTopActivities();
                    }
                } break;
                ...
```

> RootActivityContainer.java

```
 boolean resumeFocusedStacksTopActivities() {
        // 全部传入null
        return resumeFocusedStacksTopActivities(null, null, null);
    }

    boolean resumeFocusedStacksTopActivities(
            ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {

        if (!mStackSupervisor.readyToResume()) {
            return false;
        }

        boolean result = false;
        if (targetStack != null && (targetStack.isTopStackOnDisplay()
                || getTopDisplayFocusedStack() == targetStack)) {
            result = targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
        }

        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            boolean resumedOnDisplay = false;
            final ActivityDisplay display = mActivityDisplays.get(displayNdx);
            for (int stackNdx = display.getChildCount() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = display.getChildAt(stackNdx);
                final ActivityRecord topRunningActivity = stack.topRunningActivityLocked();
                if (!stack.isFocusableAndVisible() || topRunningActivity == null) {
                    continue;
                }
                if (stack == targetStack) {
                    // Simply update the result for targetStack because the targetStack had
                    // already resumed in above. We don't want to resume it again, especially in
                    // some cases, it would cause a second launch failure if app process was dead.
                    resumedOnDisplay |= result;
                    continue;
                }
                if (display.isTopStack(stack) && topRunningActivity.isState(RESUMED)) {
                    // Kick off any lingering app transitions form the MoveTaskToFront operation,
                    // but only consider the top task and stack on that display.
                    stack.executeAppTransition(targetOptions);
                } else {
                    resumedOnDisplay |= topRunningActivity.makeActiveIfNeeded(target);
                }
            }
            if (!resumedOnDisplay) {
                // In cases when there are no valid activities (e.g. device just booted or launcher
                // crashed) it's possible that nothing was resumed on a display. Requesting resume
                // of top activity in focused stack explicitly will make sure that at least home
                // activity is started and resumed, and no recursion occurs.
                final ActivityStack focusedStack = display.getFocusedStack();
                if (focusedStack != null) {
                    // 刚启动或者launch奔溃后，走这里
                    focusedStack.resumeTopActivityUncheckedLocked(target, targetOptions);
                }
            }
        }

        return result;
    }
```

> ActivityStack.java

```
// 没有检测和锁定？ 因此，只能调用一次！
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        if (mInResumeTopActivity) {
            // Don't even start recursing.
            return false;
        }

        boolean result = false;
        try {
            // Protect against recursion.// 防止多次调用
            mInResumeTopActivity = true; 
            // 开始锁定topActivity，两个参数都是null
            result = resumeTopActivityInnerLocked(prev, options);

            // When resuming the top activity, it may be necessary to pause the top activity (for
            // example, returning to the lock screen. We suppress the normal pause logic in
            // {@link #resumeTopActivityUncheckedLocked}, since the top activity is resumed at the
            // end. We call the {@link ActivityStackSupervisor#checkReadyForSleepLocked} again here
            // to ensure any necessary pause logic occurs. In the case where the Activity will be
            // shown regardless of the lock screen, the call to
            // {@link ActivityStackSupervisor#checkReadyForSleepLocked} is skipped.
            
            final ActivityRecord next = topRunningActivityLocked(true /* focusableOnly */);
            if (next == null || !next.canTurnScreenOn()) {
                checkReadyForSleep();
            }
        } finally {
            mInResumeTopActivity = false;
        }

        return result;
    }
    
```

# 几个概念

ActivityDisplay ActivityStack TaskRecord ActivityRecord

ActivityStackSupervisor






























