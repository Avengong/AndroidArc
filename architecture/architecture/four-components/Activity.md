# AMS中的一些概念

## Activity

一个屏幕对应一个activityDisplay。里面包含多个activityStack。

一个ActivityStack 对应一个进程，里面可能有多个TaskRecord。一个TaskRecord对应一个栈。

TaskRecord里面有很多activities。一个activity对应一个activityRecord。

## ProcessRecord

ProcessRecord 包含了一个进程中所以的activityRecord，不同的TaskRecord中的activityRecord可能属于同一个ProcessRecord。
为甚？因为一个进程可以开辟多个activity栈，也就是多个TaskRecord。如 利用singleInstance启动模式。那么就有两个栈。

## 可以验证以上的关系吗？ 特别是activityStack??

```
adb shell dumpsys activity activities
```

### 利用singleInstance模式启动。

第一个启动activity(standard)： TaskInfo{userId=0 stackId=144 taskId=2344 displayId=0 isRunning=true
baseIntent=Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER]
flg=0x10000000 cmp=com.zygote.insight/.MainActivity }
baseActivity=ComponentInfo{com.zygote.insight/com.zygote.insight.MainActivity}
topActivity=ComponentInfo{com.zygote.insight/com.zygote.insight.MainActivity} origActivity=null
realActivity=ComponentInfo{com.zygote.insight/com.zygote.insight.MainActivity} numActivities=1
lastActiveTime=856901021 supportsSplitScreenMultiWindow=true resizeMode=1

第二个 activity(singleInstance)： TaskInfo{userId=0 stackId=145 taskId=2345 displayId=0 isRunning=true
baseIntent=Intent { flg=0x10000000 cmp=com.zygote.insight/.SecondActivity }
baseActivity=ComponentInfo{com.zygote.insight/com.zygote.insight.SecondActivity}
topActivity=ComponentInfo{com.zygote.insight/com.zygote.insight.SecondActivity} origActivity=null
realActivity=ComponentInfo{com.zygote.insight/com.zygote.insight.SecondActivity} numActivities=1
lastActiveTime=856909378 supportsSplitScreenMultiWindow=true resizeMode=1

dumpsys 信息： 确实是两个 ActivityStack,两个TaskRecord。

```
    // 最上面的 ActivityStack 155.
 Stack #155: type=standard mode=fullscreen
  isSleeping=false
  mBounds=Rect(0, 0 - 0, 0)
    Task id #2355
    mBounds=Rect(0, 0 - 0, 0)
    mMinWidth=-1
    mMinHeight=-1
    mLastNonFullscreenBounds=null
    * TaskRecord{77fa9e9 #2355 A=com.zygote.insight U=0 StackId=155 sz=1}
      userId=0 effectiveUid=u0a1398 mCallingUid=u0a1398 mUserSetupComplete=true mCallingPackage=com.zygote.insight
      affinity=com.zygote.insight
      intent={flg=0x10000000 cmp=com.zygote.insight/.SecondActivity}
      mActivityComponent=com.zygote.insight/.SecondActivity
      autoRemoveRecents=false isPersistable=true numFullscreen=1 activityType=1
      rootWasReset=false mNeverRelinquishIdentity=true mReuseTask=false mLockTaskAuth=LOCK_TASK_AUTH_PINNABLE
      Activities=[ActivityRecord{ce7f0e3 u0 com.zygote.insight/.SecondActivity t2355}]
      askedCompatMode=false inRecents=true isAvailable=true
      mRootProcess=ProcessRecord{dfef6f2 8961:com.zygote.insight/u0a1398}
      stackId=155
      hasBeenVisible=true mResizeMode=RESIZE_MODE_RESIZEABLE_VIA_SDK_VERSION mSupportsPictureInPicture=false isResizeable=true lastActiveTime=881512926 (inactive for 17s)
      mAppSceneMode == 0
      * Hist #0: ActivityRecord{ce7f0e3 u0 com.zygote.insight/.SecondActivity t2355}
          packageName=com.zygote.insight processName=com.zygote.insight
          launchedFromUid=11398 launchedFromPackage=com.zygote.insight userId=0
          app=ProcessRecord{dfef6f2 8961:com.zygote.insight/u0a1398}
          Intent { flg=0x10000000 cmp=com.zygote.insight/.SecondActivity }
          frontOfTask=true task=TaskRecord{77fa9e9 #2355 A=com.zygote.insight U=0 StackId=155 sz=1}
          taskAffinity=com.zygote.insight
          mActivityComponent=com.zygote.insight/.SecondActivity
          baseDir=/data/app/com.zygote.insight-p3qWjdGW1AgjwfX51lQN-A==/base.apk
          dataDir=/data/user/0/com.zygote.insight
          stateNotNeeded=false componentSpecified=true mActivityType=standard
          compat={420dpi} labelRes=0x7f0c001e icon=0x7f0b0001 theme=0x7f0d0006
          mLastReportedConfigurations:
           mGlobalConfig={1.0 0  ?mcc?mnc [zh_CN] ldltr sw411dp w411dp h750dp 420dpi nrml long port force dark=0 finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2160) mAppBounds=Rect(0, 0 - 1080, 2034) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=undefined mAlwaysOnTop=undefined mRotation=ROTATION_0} s.21FontSeq = 1, FontUserId = 0, FontID = 1}
           mOverrideConfig={1.0 0  ?mcc?mnc [zh_CN] ldltr sw411dp w411dp h750dp 420dpi nrml long port force dark=0 finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2160) mAppBounds=Rect(0, 0 - 1080, 2034) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} s.1FontSeq = 1, FontUserId = 0, FontID = 1}
          CurrentConfiguration={1.0 0  ?mcc?mnc [zh_CN] ldltr sw411dp w411dp h750dp 420dpi nrml long port force dark=0 finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2160) mAppBounds=Rect(0, 0 - 1080, 2034) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} s.1FontSeq = 1, FontUserId = 0, FontID = 1}
          taskDescription: label="null" icon=null iconResource=0 iconFilename=null primaryColor=ff6200ee
           backgroundColor=fffafafa
           statusBarColor=ff3700b3
           navigationBarColor=ff000000
          launchFailed=false launchCount=1 lastLaunchTime=-17s645ms
          haveState=false icicle=null
          state=RESUMED stopped=false delayedResume=false finishing=false
          keysPaused=false inHistory=true visible=true sleeping=false idle=true mStartingWindowState=STARTING_WINDOW_SHOWN
          fullscreen=true noDisplay=false immersive=false launchMode=3
          frozenBeforeDestroy=false forceNewConfig=false
          mActivityType=standard
           nowVisible=true lastVisibleTime=-16s994ms
          resizeMode=RESIZE_MODE_RESIZEABLE_VIA_SDK_VERSION
          mLastReportedMultiWindowMode=false mLastReportedPictureInPictureMode=false

    Running activities (most recent first):
      TaskRecord{77fa9e9 #2355 A=com.zygote.insight U=0 StackId=155 sz=1}
        Run #0: ActivityRecord{ce7f0e3 u0 com.zygote.insight/.SecondActivity t2355}

    mResumedActivity: ActivityRecord{ce7f0e3 u0 com.zygote.insight/.SecondActivity t2355}

    // 从上往下，第二个 ActivityStack 154
  Stack #154: type=standard mode=fullscreen
  isSleeping=false
  mBounds=Rect(0, 0 - 0, 0)

    Task id #2354
    mBounds=Rect(0, 0 - 0, 0)
    mMinWidth=-1
    mMinHeight=-1
    mLastNonFullscreenBounds=null
    * TaskRecord{a56866e #2354 A=com.zygote.insight U=0 StackId=154 sz=1}
      userId=0 effectiveUid=u0a1398 mCallingUid=2000 mUserSetupComplete=true mCallingPackage=null
      affinity=com.zygote.insight
      intent={act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.zygote.insight/.MainActivity}
      mActivityComponent=com.zygote.insight/.MainActivity
      autoRemoveRecents=false isPersistable=true numFullscreen=1 activityType=1
      rootWasReset=false mNeverRelinquishIdentity=true mReuseTask=false mLockTaskAuth=LOCK_TASK_AUTH_PINNABLE
      Activities=[ActivityRecord{576278e u0 com.zygote.insight/.MainActivity t2354}]
      askedCompatMode=false inRecents=false isAvailable=true
      stackId=154
      hasBeenVisible=true mResizeMode=RESIZE_MODE_RESIZEABLE_VIA_SDK_VERSION mSupportsPictureInPicture=false isResizeable=true lastActiveTime=881512917 (inactive for 17s)
      mAppSceneMode == 0
      * Hist #0: ActivityRecord{576278e u0 com.zygote.insight/.MainActivity t2354}
          packageName=com.zygote.insight processName=com.zygote.insight
          launchedFromUid=2000 launchedFromPackage=null userId=0
          app=ProcessRecord{dfef6f2 8961:com.zygote.insight/u0a1398}
          Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.zygote.insight/.MainActivity }
          frontOfTask=true task=TaskRecord{a56866e #2354 A=com.zygote.insight U=0 StackId=154 sz=1}
          taskAffinity=com.zygote.insight
          mActivityComponent=com.zygote.insight/.MainActivity
          baseDir=/data/app/com.zygote.insight-p3qWjdGW1AgjwfX51lQN-A==/base.apk
          dataDir=/data/user/0/com.zygote.insight
          stateNotNeeded=false componentSpecified=true mActivityType=standard
          compat={420dpi} labelRes=0x7f0c001e icon=0x7f0b0001 theme=0x7f0d0006
          mLastReportedConfigurations:
           mGlobalConfig={1.0 0  ?mcc?mnc [zh_CN] ldltr sw411dp w411dp h750dp 420dpi nrml long port force dark=0 finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2160) mAppBounds=Rect(0, 0 - 1080, 2034) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=undefined mAlwaysOnTop=undefined mRotation=ROTATION_0} s.21FontSeq = 1, FontUserId = 0, FontID = 1}
           mOverrideConfig={1.0 0  ?mcc?mnc [zh_CN] ldltr sw411dp w411dp h750dp 420dpi nrml long port force dark=0 finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2160) mAppBounds=Rect(0, 0 - 1080, 2034) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} s.1FontSeq = 1, FontUserId = 0, FontID = 1}
          CurrentConfiguration={1.0 0  ?mcc?mnc [zh_CN] ldltr sw411dp w411dp h750dp 420dpi nrml long port force dark=0 finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2160) mAppBounds=Rect(0, 0 - 1080, 2034) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} s.1FontSeq = 1, FontUserId = 0, FontID = 1}
          taskDescription: label="null" icon=null iconResource=0 iconFilename=null primaryColor=ff6200ee
           backgroundColor=fffafafa
           statusBarColor=ff3700b3
           navigationBarColor=ff000000
          launchFailed=false launchCount=0 lastLaunchTime=-21m9s54ms
          haveState=true icicle=Bundle[mParcelledData.dataSize=1588]
          state=STOPPED stopped=true delayedResume=false finishing=false
          keysPaused=false inHistory=true visible=false sleeping=false idle=true mStartingWindowState=STARTING_WINDOW_SHOWN
          fullscreen=true noDisplay=false immersive=false launchMode=0
          frozenBeforeDestroy=false forceNewConfig=false
          mActivityType=standard
           nowVisible=false lastVisibleTime=-21m8s53ms
          resizeMode=RESIZE_MODE_RESIZEABLE_VIA_SDK_VERSION
          mLastReportedMultiWindowMode=false mLastReportedPictureInPictureMode=false

    Running activities (most recent first):
      TaskRecord{a56866e #2354 A=com.zygote.insight U=0 StackId=154 sz=1}
        Run #0: ActivityRecord{576278e u0 com.zygote.insight/.MainActivity t2354}

    mLastPausedActivity: ActivityRecord{576278e u0 com.zygote.insight/.MainActivity t2354}

    // 第三个 ActivityStack 0 桌面
  Stack #0: type=home mode=fullscreen
  isSleeping=false
  mBounds=Rect(0, 0 - 0, 0)
    ......
```

总结： 第二个activity设置了singleInstance模式，ams里面会创建两个ActivityStack、每个ActivityStack里面会创建各自的TaskRecord
对象。每一个TaskRecord里面各有一个activityRecord。 提问：为什么查看任务栏只显示一个图标？
因为虽然分开了两个ActivityStack。但是由于TaskAffinity一样，所以会共用一个显示。点击返回，那么MainActivity会显示出来。
例外：如果你按了任务键，然后在按返回键。那么会回到home界面。为什么？为啥任务键ui默认是home的一部分。相当于 SecondActivity 的ActivityStack下面已经是home了。

提问： 想要分开显示两个图标怎么办？ 简单。设置一个TaskAffinity即可。

### 利用singleTask模式启动。

第一个： TaskInfo{userId=0 stackId=146 taskId=2346 displayId=0 isRunning=true baseIntent=Intent {
act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000
cmp=com.zygote.insight/.MainActivity }
baseActivity=ComponentInfo{com.zygote.insight/com.zygote.insight.MainActivity}
topActivity=ComponentInfo{com.zygote.insight/com.zygote.insight.MainActivity} origActivity=null
realActivity=ComponentInfo{com.zygote.insight/com.zygote.insight.MainActivity} numActivities=1
lastActiveTime=857253418 supportsSplitScreenMultiWindow=true resizeMode=1

第二个： TaskInfo{userId=0 stackId=146 taskId=2346 displayId=0 isRunning=true baseIntent=Intent {
act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000
cmp=com.zygote.insight/.MainActivity }
baseActivity=ComponentInfo{com.zygote.insight/com.zygote.insight.MainActivity}
topActivity=ComponentInfo{com.zygote.insight/com.zygote.insight.SecondActivity} origActivity=null
realActivity=ComponentInfo{com.zygote.insight/com.zygote.insight.MainActivity} numActivities=2
lastActiveTime=857259889 supportsSplitScreenMultiWindow=true resizeMode=1

总结： singleTask没有singleInstance强制性。 它可以在当前的TaskRecord。跟普通的没有什么区别。

1. 但是如果设置了TaskAffinity，则会另外起一个ActivityStack。
2. 如果重新启动，则不会重新onCreate，只会回调onNewIntent。
3. ABC,AC正常模式。B设置了singleTask+TaskAfinity。A启动B，此时会切换栈。B在启动C，C会被加入到B的任务栈中，而不是加入
   到A的任务栈中。这就是singleInstance与singleTask的区别。
4.

### standard模式跟之前task的一样。

```
OnePlus5T:/ $ dumpsys activity activities
ACTIVITY MANAGER ACTIVITIES (dumpsys activity activities)

Display #0 (activities from top to bottom): //display 0号屏幕，从上到下显示
   // 第一个AcitiityStack stacID=148 
  Stack #148: type=standard mode=fullscreen  
  isSleeping=false
  mBounds=Rect(0, 0 - 0, 0)
    Task id #2348 // TaskRecord 的 id= 2348 
    mBounds=Rect(0, 0 - 0, 0)
    mMinWidth=-1
    mMinHeight=-1
    mLastNonFullscreenBounds=null
    // 一个 TaskRecord
    * TaskRecord{8fd0eb7 #2348 A=com.zygote.insight U=0 StackId=148 sz=1}
      userId=0 effectiveUid=u0a1398 mCallingUid=2000 mUserSetupComplete=true mCallingPackage=null
      affinity=com.zygote.insight
      intent={act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.zygote.insight/.MainActivity}
      mActivityComponent=com.zygote.insight/.MainActivity
      autoRemoveRecents=false isPersistable=true numFullscreen=1 activityType=1
      rootWasReset=false mNeverRelinquishIdentity=true mReuseTask=false mLockTaskAuth=LOCK_TASK_AUTH_PINNABLE
      // 这个 TaskRecord 里面有多少个activities 
      Activities=[ActivityRecord{bc6313c u0 com.zygote.insight/.MainActivity t2348}]
      askedCompatMode=false inRecents=true isAvailable=true
      mRootProcess=ProcessRecord{4793241 4916:com.zygote.insight/u0a1398}
      stackId=148
      hasBeenVisible=true mResizeMode=RESIZE_MODE_RESIZEABLE_VIA_SDK_VERSION mSupportsPictureInPicture=false isResizeable=true lastActiveTime=861969273 (inactive for 48s)
      mAppSceneMode == 0
      // 历史栈
      * Hist #0: ActivityRecord{bc6313c u0 com.zygote.insight/.MainActivity t2348}
          packageName=com.zygote.insight processName=com.zygote.insight
          launchedFromUid=2000 launchedFromPackage=null userId=0
          app=ProcessRecord{4793241 4916:com.zygote.insight/u0a1398}
          Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.zygote.insight/.MainActivity }
          frontOfTask=true task=TaskRecord{8fd0eb7 #2348 A=com.zygote.insight U=0 StackId=148 sz=1}
          taskAffinity=com.zygote.insight
          mActivityComponent=com.zygote.insight/.MainActivity
          baseDir=/data/app/com.zygote.insight-GQw7Ovt1SRUEWk9rY6XxkQ==/base.apk
          dataDir=/data/user/0/com.zygote.insight
          stateNotNeeded=false componentSpecified=true mActivityType=standard
          compat={420dpi} labelRes=0x7f0c001e icon=0x7f0b0001 theme=0x7f0d0006
          mLastReportedConfigurations:
           mGlobalConfig={1.0 0  ?mcc?mnc [zh_CN] ldltr sw411dp w411dp h750dp 420dpi nrml long port force dark=0 finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2160) mAppBounds=Rect(0, 0 - 1080, 2034) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=undefined mAlwaysOnTop=undefined mRotation=ROTATION_0} s.21FontSeq = 1, FontUserId = 0, FontID = 1}
           mOverrideConfig={1.0 0  ?mcc?mnc [zh_CN] ldltr sw411dp w411dp h750dp 420dpi nrml long port force dark=0 finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2160) mAppBounds=Rect(0, 0 - 1080, 2034) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} s.1FontSeq = 1, FontUserId = 0, FontID = 1}
          CurrentConfiguration={1.0 0  ?mcc?mnc [zh_CN] ldltr sw411dp w411dp h750dp 420dpi nrml long port force dark=0 finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2160) mAppBounds=Rect(0, 0 - 1080, 2034) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} s.1FontSeq = 1, FontUserId = 0, FontID = 1}
          taskDescription: label="null" icon=null iconResource=0 iconFilename=null primaryColor=ff6200ee
           backgroundColor=fffafafa
           statusBarColor=ff3700b3
           navigationBarColor=ff000000
          launchFailed=false launchCount=1 lastLaunchTime=-48s303ms
          haveState=false icicle=null
          state=RESUMED stopped=false delayedResume=false finishing=false
          keysPaused=false inHistory=true visible=true sleeping=false idle=true mStartingWindowState=STARTING_WINDOW_SHOWN
          fullscreen=true noDisplay=false immersive=false launchMode=0
          frozenBeforeDestroy=false forceNewConfig=false
          mActivityType=standard
           nowVisible=true lastVisibleTime=-47s304ms
          resizeMode=RESIZE_MODE_RESIZEABLE_VIA_SDK_VERSION
          mLastReportedMultiWindowMode=false mLastReportedPictureInPictureMode=false
    // 这个ActivityStack 里面处于running 的activities
    Running activities (most recent first):
      TaskRecord{8fd0eb7 #2348 A=com.zygote.insight U=0 StackId=148 sz=1}
        Run #0: ActivityRecord{bc6313c u0 com.zygote.insight/.MainActivity t2348}
 // 这个ActivityStack 里面处于 resummed 的activity
    mResumedActivity: ActivityRecord{bc6313c u0 com.zygote.insight/.MainActivity t2348}
  // 第二个 ActivityStack stackId=0.表示是桌面launcher
  Stack #0: type=home mode=fullscreen
  isSleeping=false
  mBounds=Rect(0, 0 - 0, 0)

    Task id #2202
    mBounds=Rect(0, 0 - 0, 0)
    mMinWidth=-1
    mMinHeight=-1
    mLastNonFullscreenBounds=null
    * TaskRecord{6c53624 #2202 I=net.oneplus.launcher/.Launcher U=0 StackId=0 sz=1}
      userId=0 effectiveUid=u0a60 mCallingUid=0 mUserSetupComplete=true mCallingPackage=null
      intent={act=android.intent.action.MAIN cat=[android.intent.category.HOME] flg=0x10000100 cmp=net.oneplus.launcher/.Launcher}
      mActivityComponent=net.oneplus.launcher/.Launcher
      autoRemoveRecents=false isPersistable=true numFullscreen=1 activityType=2
      rootWasReset=false mNeverRelinquishIdentity=true mReuseTask=false mLockTaskAuth=LOCK_TASK_AUTH_PINNABLE
      // 也只有一个 ActivityRecord 
      Activities=[ActivityRecord{a61e3f0 u0 net.oneplus.launcher/.Launcher t2202}]
      askedCompatMode=false inRecents=true isAvailable=true
      mRootProcess=ProcessRecord{8c8b5f2 3918:net.oneplus.launcher/u0a60}
      stackId=0
      hasBeenVisible=true mResizeMode=RESIZE_MODE_RESIZEABLE mSupportsPictureInPicture=false isResizeable=true lastActiveTime=861969168 (inactive for 48s)
      mAppSceneMode == 0
      * Hist #0: ActivityRecord{a61e3f0 u0 net.oneplus.launcher/.Launcher t2202}
          packageName=net.oneplus.launcher processName=net.oneplus.launcher
          launchedFromUid=0 launchedFromPackage=null userId=0
          app=ProcessRecord{8c8b5f2 3918:net.oneplus.launcher/u0a60}
          Intent { act=android.intent.action.MAIN cat=[android.intent.category.HOME] flg=0x10000100 cmp=net.oneplus.launcher/.Launcher }
          frontOfTask=true task=TaskRecord{6c53624 #2202 I=net.oneplus.launcher/.Launcher U=0 StackId=0 sz=1}
          taskAffinity=null
          mActivityComponent=net.oneplus.launcher/.Launcher
          baseDir=/data/app/net.oneplus.launcher-kY2BS4AdlYlco5EYBkCDWQ==/base.apk
          dataDir=/data/user/0/net.oneplus.launcher
          stateNotNeeded=true componentSpecified=false mActivityType=home
          compat={420dpi} labelRes=0x7f130081 icon=0x7f100005 theme=0x7f140118
          mLastReportedConfigurations:
           mGlobalConfig={1.0 0  ?mcc?mnc [zh_CN] ldltr sw411dp w411dp h750dp 420dpi nrml long port force dark=0 finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2160) mAppBounds=Rect(0, 0 - 1080, 2034) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=undefined mAlwaysOnTop=undefined mRotation=ROTATION_0} s.21FontSeq = 1, FontUserId = 0, FontID = 1}
           mOverrideConfig={1.0 0  ?mcc?mnc [zh_CN] ldltr sw411dp w411dp h750dp 420dpi nrml long port force dark=0 finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2160) mAppBounds=Rect(0, 0 - 1080, 2034) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=home mAlwaysOnTop=undefined mRotation=ROTATION_0} s.2FontSeq = 1, FontUserId = 0, FontID = 1}
          CurrentConfiguration={1.0 0  ?mcc?mnc [zh_CN] ldltr sw411dp w411dp h750dp 420dpi nrml long port force dark=0 finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2160) mAppBounds=Rect(0, 0 - 1080, 2034) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=home mAlwaysOnTop=undefined mRotation=ROTATION_0} s.2FontSeq = 1, FontUserId = 0, FontID = 1}
          RequestedOverrideConfiguration={0.0 0  ?mcc?mnc ?localeList ?layoutDir ?swdp ?wdp ?hdp ?density ?lsize ?long ?ldr ?wideColorGamut ?orien ?uimode ?night force dark=0 ?touch ?keyb/?/? ?nav/? winConfig={ mBounds=Rect(0, 0 - 0, 0) mAppBounds=null mWindowingMode=undefined mDisplayWindowingMode=undefined mActivityType=home mAlwaysOnTop=undefined mRotation=undefined}FontSeq = 0, FontUserId = -1, FontID = 1}
          taskDescription: label="null" icon=null iconResource=0 iconFilename=null primaryColor=fff5f5f5
           backgroundColor=ffffffff
           statusBarColor=0
           navigationBarColor=0
          launchFailed=false launchCount=0 lastLaunchTime=-2d10h10m36s735ms
          haveState=true icicle=Bundle[mParcelledData.dataSize=11708]
          state=STOPPED stopped=true delayedResume=false finishing=false
          keysPaused=false inHistory=true visible=false sleeping=false idle=true mStartingWindowState=STARTING_WINDOW_REMOVED
          fullscreen=true noDisplay=false immersive=false launchMode=2
          frozenBeforeDestroy=false forceNewConfig=false
          mActivityType=home
           nowVisible=false lastVisibleTime=-4m19s96ms
          resizeMode=RESIZE_MODE_RESIZEABLE
          mLastReportedMultiWindowMode=false mLastReportedPictureInPictureMode=false

    //
    Running activities (most recent first):
      TaskRecord{6c53624 #2202 I=net.oneplus.launcher/.Launcher U=0 StackId=0 sz=1}
        Run #0: ActivityRecord{a61e3f0 u0 net.oneplus.launcher/.Launcher t2202}

    mLastPausedActivity: ActivityRecord{a61e3f0 u0 net.oneplus.launcher/.Launcher t2202}

 ResumedActivity:ActivityRecord{bc6313c u0 com.zygote.insight/.MainActivity t2348}

  ResumedActivity: ActivityRecord{bc6313c u0 com.zygote.insight/.MainActivity t2348}

// ActivityStackSupervisor 的状态
ActivityStackSupervisor state:
   // 当前获取到焦点的栈 ActivityStack stackID=148.
  topDisplayFocusedStack=ActivityStack{c4f358d stackId=148 type=standard mode=fullscreen visible=true translucent=false, 1 tasks}
  displayId=0 stacks=2 //一块屏幕，两个 ActivityStack。
   mHomeStack=ActivityStack{818f342 stackId=0 type=home mode=fullscreen visible=false translucent=true, 1 tasks}
   mPreferredTopFocusableStack=ActivityStack{c4f358d stackId=148 type=standard mode=fullscreen visible=true translucent=false, 1 tasks}
   mLastFocusedStack=ActivityStack{c4f358d stackId=148 type=standard mode=fullscreen visible=true translucent=false, 1 tasks}
  mCurTaskIdForUser={0=2348}
  mUserStackInFront={}
  isHomeRecentsComponent=true  KeyguardController:
    mKeyguardShowing=false
    mAodShowing=false
    mKeyguardGoingAway=false
    Occluded=false DismissingKeyguardActivity=null at display=0
    mDismissalRequested=false
    mVisibilityTransactionDepth=0
  LockTaskController
    mLockTaskModeState=NONE
    mLockTaskModeTasks=
    mLockTaskPackages (userId:packages)=
      u0:[]
```

如下： 在清单的SecondActivity中添加了taskAffinity还是在同一个栈？ 为啥？

```
TaskInfo{userId=0 stackId=148 taskId=2348 displayId=0 isRunning=true baseIntent=Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.zygote.insight/.MainActivity } baseActivity=ComponentInfo{com.zygote.insight/com.zygote.insight.MainActivity} topActivity=ComponentInfo{com.zygote.insight/com.zygote.insight.MainActivity} origActivity=null realActivity=ComponentInfo{com.zygote.insight/com.zygote.insight.MainActivity} numActivities=1 lastActiveTime=861969273 supportsSplitScreenMultiWindow=true resizeMode=1

TaskInfo{userId=0 stackId=148 taskId=2348 displayId=0 isRunning=true baseIntent=Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.zygote.insight/.MainActivity } baseActivity=ComponentInfo{com.zygote.insight/com.zygote.insight.MainActivity} topActivity=ComponentInfo{com.zygote.insight/com.zygote.insight.SecondActivity} origActivity=null realActivity=ComponentInfo{com.zygote.insight/com.zygote.insight.MainActivity} numActivities=2 lastActiveTime=863268388 supportsSplitScreenMultiWindow=true resizeMode=1

```

dumpsys如下：

```
  Stack #148: type=standard mode=fullscreen
  isSleeping=false
  mBounds=Rect(0, 0 - 0, 0)
    Task id #2348
    mBounds=Rect(0, 0 - 0, 0)
    mMinWidth=-1
    mMinHeight=-1
    mLastNonFullscreenBounds=null
    * TaskRecord{8fd0eb7 #2348 A=com.zygote.insight U=0 StackId=148 sz=2}
      userId=0 effectiveUid=u0a1398 mCallingUid=2000 mUserSetupComplete=true mCallingPackage=null
      affinity=com.zygote.insight
      intent={act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.zygote.insight/.MainActivity}
      mActivityComponent=com.zygote.insight/.MainActivity
      autoRemoveRecents=false isPersistable=true numFullscreen=2 activityType=1
      rootWasReset=false mNeverRelinquishIdentity=true mReuseTask=false mLockTaskAuth=LOCK_TASK_AUTH_PINNABLE
      Activities=[ActivityRecord{bc6313c u0 com.zygote.insight/.MainActivity t2348}, ActivityRecord{6bcb748 u0 com.zygote.insight/.SecondActivity t2348}]
      askedCompatMode=false inRecents=true isAvailable=true
      mRootProcess=ProcessRecord{4793241 4916:com.zygote.insight/u0a1398}
      stackId=148
      hasBeenVisible=true mResizeMode=RESIZE_MODE_RESIZEABLE_VIA_SDK_VERSION mSupportsPictureInPicture=false isResizeable=true lastActiveTime=863268388 (inactive for 51s)
      mAppSceneMode == 0
      * Hist #1: ActivityRecord{6bcb748 u0 com.zygote.insight/.SecondActivity t2348}
          packageName=com.zygote.insight processName=com.zygote.insight
          launchedFromUid=11398 launchedFromPackage=com.zygote.insight userId=0
          app=ProcessRecord{4793241 4916:com.zygote.insight/u0a1398}
          Intent { cmp=com.zygote.insight/.SecondActivity }
          frontOfTask=false task=TaskRecord{8fd0eb7 #2348 A=com.zygote.insight U=0 StackId=148 sz=2}
          taskAffinity=com.zygote.insight.task11 // 这里确实设置了 taskAffinity，为啥还在同一个栈
          mActivityComponent=com.zygote.insight/.SecondActivity
          baseDir=/data/app/com.zygote.insight-GQw7Ovt1SRUEWk9rY6XxkQ==/base.apk
          dataDir=/data/user/0/com.zygote.insight
          stateNotNeeded=false componentSpecified=true mActivityType=standard
          compat={420dpi} labelRes=0x7f0c001e icon=0x7f0b0001 theme=0x7f0d0006
          mLastReportedConfigurations:
           mGlobalConfig={1.0 0  ?mcc?mnc [zh_CN] ldltr sw411dp w411dp h750dp 420dpi nrml long port force dark=0 finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2160) mAppBounds=Rect(0, 0 - 1080, 2034) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=undefined mAlwaysOnTop=undefined mRotation=ROTATION_0} s.21FontSeq = 1, FontUserId = 0, FontID = 1}
           mOverrideConfig={1.0 0  ?mcc?mnc [zh_CN] ldltr sw411dp w411dp h750dp 420dpi nrml long port force dark=0 finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2160) mAppBounds=Rect(0, 0 - 1080, 2034) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} s.1FontSeq = 1, FontUserId = 0, FontID = 1}
          CurrentConfiguration={1.0 0  ?mcc?mnc [zh_CN] ldltr sw411dp w411dp h750dp 420dpi nrml long port force dark=0 finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2160) mAppBounds=Rect(0, 0 - 1080, 2034) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} s.1FontSeq = 1, FontUserId = 0, FontID = 1}
          taskDescription: label="null" icon=null iconResource=0 iconFilename=null primaryColor=ff6200ee
           backgroundColor=fffafafa
           statusBarColor=ff3700b3
           navigationBarColor=ff000000
          launchFailed=false launchCount=1 lastLaunchTime=-51s78ms
          haveState=false icicle=null
          state=RESUMED stopped=false delayedResume=false finishing=false
          keysPaused=false inHistory=true visible=true sleeping=false idle=true mStartingWindowState=STARTING_WINDOW_NOT_SHOWN
          fullscreen=true noDisplay=false immersive=false launchMode=0
          frozenBeforeDestroy=false forceNewConfig=false
          mActivityType=standard
           nowVisible=true lastVisibleTime=-50s583ms
          resizeMode=RESIZE_MODE_RESIZEABLE_VIA_SDK_VERSION
          mLastReportedMultiWindowMode=false mLastReportedPictureInPictureMode=false
      * Hist #0: ActivityRecord{bc6313c u0 com.zygote.insight/.MainActivity t2348}
          packageName=com.zygote.insight processName=com.zygote.insight
          launchedFromUid=2000 launchedFromPackage=null userId=0
          app=ProcessRecord{4793241 4916:com.zygote.insight/u0a1398}
          Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10000000 cmp=com.zygote.insight/.MainActivity }
          frontOfTask=true task=TaskRecord{8fd0eb7 #2348 A=com.zygote.insight U=0 StackId=148 sz=2}
          taskAffinity=com.zygote.insight
          mActivityComponent=com.zygote.insight/.MainActivity
          baseDir=/data/app/com.zygote.insight-GQw7Ovt1SRUEWk9rY6XxkQ==/base.apk
          dataDir=/data/user/0/com.zygote.insight
          stateNotNeeded=false componentSpecified=true mActivityType=standard
          compat={420dpi} labelRes=0x7f0c001e icon=0x7f0b0001 theme=0x7f0d0006
          mLastReportedConfigurations:
           mGlobalConfig={1.0 0  ?mcc?mnc [zh_CN] ldltr sw411dp w411dp h750dp 420dpi nrml long port force dark=0 finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2160) mAppBounds=Rect(0, 0 - 1080, 2034) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=undefined mAlwaysOnTop=undefined mRotation=ROTATION_0} s.21FontSeq = 1, FontUserId = 0, FontID = 1}
           mOverrideConfig={1.0 0  ?mcc?mnc [zh_CN] ldltr sw411dp w411dp h750dp 420dpi nrml long port force dark=0 finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2160) mAppBounds=Rect(0, 0 - 1080, 2034) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} s.1FontSeq = 1, FontUserId = 0, FontID = 1}
          CurrentConfiguration={1.0 0  ?mcc?mnc [zh_CN] ldltr sw411dp w411dp h750dp 420dpi nrml long port force dark=0 finger -keyb/v/h -nav/h winConfig={ mBounds=Rect(0, 0 - 1080, 2160) mAppBounds=Rect(0, 0 - 1080, 2034) mWindowingMode=fullscreen mDisplayWindowingMode=fullscreen mActivityType=standard mAlwaysOnTop=undefined mRotation=ROTATION_0} s.1FontSeq = 1, FontUserId = 0, FontID = 1}
          taskDescription: label="null" icon=null iconResource=0 iconFilename=null primaryColor=ff6200ee
           backgroundColor=fffafafa
           statusBarColor=ff3700b3
           navigationBarColor=ff000000
          launchFailed=false launchCount=0 lastLaunchTime=-22m30s263ms
          haveState=true icicle=Bundle[mParcelledData.dataSize=1588]
          state=STOPPED stopped=true delayedResume=false finishing=false
          keysPaused=false inHistory=true visible=false sleeping=false idle=true mStartingWindowState=STARTING_WINDOW_REMOVED
          fullscreen=true noDisplay=false immersive=false launchMode=0
          frozenBeforeDestroy=false forceNewConfig=false
          mActivityType=standard
           nowVisible=false lastVisibleTime=-22m29s264ms
          resizeMode=RESIZE_MODE_RESIZEABLE_VIA_SDK_VERSION
          mLastReportedMultiWindowMode=false mLastReportedPictureInPictureMode=false

    Running activities (most recent first):
      TaskRecord{8fd0eb7 #2348 A=com.zygote.insight U=0 StackId=148 sz=2}
        Run #1: ActivityRecord{6bcb748 u0 com.zygote.insight/.SecondActivity t2348}
        Run #0: ActivityRecord{bc6313c u0 com.zygote.insight/.MainActivity t2348}

    mResumedActivity: ActivityRecord{6bcb748 u0 com.zygote.insight/.SecondActivity t2348}
    mLastPausedActivity: ActivityRecord{bc6313c u0 com.zygote.insight/.MainActivity t2348}

  Stack #0: type=home mode=fullscreen
  isSleeping=false
  mBounds=Rect(0, 0 - 0, 0)

   ..... //下面都大同小异

```

总结： taskAffinity 表示栈的名字。优先级小于launchMode。如果启动模式standard/singletop，那么不会起作用。因为属于正常启动嘛！
如果启动模式为singelTask，表示我想要一个独立的栈来展示。那么会重新创建一个独立的ActivityStack、独立的taskRecord。 此时，app有两个栈。

关键点就是： 先看启动模式，你想不想单独在一个栈里面呢？ singleTask？ singleInstance？？

所谓包，都是静态形式存在磁盘中的文件。apk、so、jar都可以被称为包。包的管理者面临的第一任务： 如何让静态的文件转变内存中的数据结构。 负责将静态文件转变为内存的数据结构的是
packageParse 包解析者。

一个包就对应内存中的一个 Package。而Package有哪些属性？ 这些属性是如何赋值的？这就是解析过程。

```
apk--------PP----->Package
apk--------PP----->Package
apk--------PP----->Package
```

1，ActivityInfo 到底有什么？！

- parcelable我知道，可以跨进程传输。那么是app和 ams之间传输吗？
- ActivityInfo继承了 ComponentInfo，因此，ComponentInfo 有什么用？

```
Component：
  | intent 为何会有这个属性呢。因为activity可以配置意图啊！草！是的哦！
  | classname
  | ...
  
```

2， 是不是还有 ServiceInfo、broadcastInfo、providerInfo？ 是的。还有权限permission，这些都是通过解析manifest.xml文件来获得的。

```
public class ActivityInfo extends ComponentInfo implements Parcelable{}

```

3， ResolveInfo 是什么？ 是intent解析后的最终数据结构。 包含
activityInfo、serviceInfo、providerInfo、broadcastInfo，其中只有一个不为空。

ActivityIntentResolver 用来解析发往 Activity和broadcast的 intent。 ServiceIntentResolve 用来解析发往 Service的
intent。 ProviderIntentResolve 用力啊解析发往 Provider的 intent。 系统每收到一个intent，就会使用对应的解析器来解析。以上三者都是
IntentResolver 的子类。

4，ActivityRecord 是什么？ An entry in the history stack, representing an activity.

5, TaskRecord ?? 任务栈。栈里面有很多activity？

6,ActivityClientRecord 在app端的 activity封装。ActivityClientRecord是ActivityThread的内部类

7, HandlerParams 是PMS中的抽象内部类。 是packageHandler 中使用的数据结构。子类有：

InstallParams： MultiPackageInstallParams:
8,为了让apk在 内部空间和sd卡可以移动 InstallArgs

# 启动流程

## Activity.java

```
  public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }
```

```
 public void startActivityForResult(@RequiresPermission Intent intent, int requestCode) {
        startActivityForResult(intent, requestCode, null);
    }
```

```
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
        @Nullable Bundle options) {
    if (mParent == null) {
        options = transferSpringboardActivityOptions(options);
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
        if (ar != null) {
            mMainThread.sendActivityResult(
                mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                ar.getResultData());
        }
        if (requestCode >= 0) {
            // If this start is requesting a result, we can avoid making
            // the activity visible until the result is received.  Setting
            // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
            // activity hidden during this time, to avoid flickering.
            // This can only be done when a result is requested because
            // that guarantees we will get information back when the
            // activity is finished, no matter what happens to it.
            mStartedActivity = true;
        }

        cancelInputsAndStartExitTransition(options);
        // TODO Consider clearing/flushing other event sources and events for child windows.
    } else {
        if (options != null) {
            mParent.startActivityFromChild(this, intent, requestCode, options);
        } else {
            // Note we want to go through this method for compatibility with
            // existing applications that may have overridden it.
            mParent.startActivityFromChild(this, intent, requestCode);
        }
    }
}
```

# Instrumentation.execStartActivity

```
  public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
            //发起者的 IApplicationThread ，用来给AMS回调
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        // 引荐者。表示谁发起的这个启动Acitivity请求
        Uri referrer = target != null ? target.onProvideReferrer() : null;
        if (referrer != null) {
            intent.putExtra(Intent.EXTRA_REFERRER, referrer);
        }
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    ActivityResult result = null;
                    if (am.ignoreMatchingSpecificIntents()) {
                        result = am.onStartActivity(intent);
                    }
                    if (result != null) {
                        am.mHits++;
                        return result;
                    } else if (am.match(who, null, intent)) {
                        am.mHits++;
                        if (am.isBlocking()) {
                            return requestCode >= 0 ? am.getResult() : null;
                        }
                        break;
                    }
                }
            }
        }
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
            // atm服务 
            int result = ActivityTaskManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```

简简单单的一个binder请求。到atm侧： Android 10 已经把 activity相关的逻辑都转移到了atm中。 疑问： token到底是啥？ 用来干什么？

```
    @Override
    public final int startActivity(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
            // startFlags=0；profileinfo=null；
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId()); // 添加了userid参数
    }
```

```

   @Override
    public int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId) {
        return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
                resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
                true /*validateIncomingUser*/); // 经过验证的用户：true。
    }

```

```
 int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
            boolean validateIncomingUser) {
        enforceNotIsolatedCaller("startActivityAsUser");

        userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
                Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

        // TODO: Switch to user app stacks here.
        return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(bOptions)
                .setMayWait(userId) //这个很关键
                .execute();

    }
```

# ActivityStack.execute()

```
 int execute() {
        try {
            // TODO(b/64750076): Look into passing request directly to these methods to allow
            // for transactional diffs and preprocessing.
            if (mRequest.mayWait) {
            // 走这个分支
                return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                        mRequest.callingPackage, mRequest.realCallingPid, mRequest.realCallingUid,
                        mRequest.intent, mRequest.resolvedType,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                        mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                        mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                        mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup,
                        mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
            } else {
                return startActivity(mRequest.caller, mRequest.intent, mRequest.ephemeralIntent,
                        mRequest.resolvedType, mRequest.activityInfo, mRequest.resolveInfo,
                        mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                        mRequest.resultWho, mRequest.requestCode, mRequest.callingPid,
                        mRequest.callingUid, mRequest.callingPackage, mRequest.realCallingPid,
                        mRequest.realCallingUid, mRequest.startFlags, mRequest.activityOptions,
                        mRequest.ignoreTargetSecurity, mRequest.componentSpecified,
                        mRequest.outActivity, mRequest.inTask, mRequest.reason,
                        mRequest.allowPendingRemoteAnimationRegistryLookup,
                        mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
            }
        } finally {
            onExecutionComplete();
        }
    }
```






