# 启动模式

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


