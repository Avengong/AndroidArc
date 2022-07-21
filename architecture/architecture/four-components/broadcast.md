# 一、概述

想要收到广播，首先就应该声明一个广播接收者，同时注册到系统中。 然后，程序发送能够匹配该注册广播的intent，由系统来匹配到，然后回调到注册者。

疑问： 相同的广播可以有多个接收者吗？ 可以有多个发送者吗？

# 二、广播简单使用

## 2.1 静态注册

1. 在清单文件中注册

```
<receiver
    android:name=".broadcast.MyStaticBroadcastReceiver"
    android:exported="true">
    <intent-filter>
         <action android:name="com.example.broadcast.MY_NOTIFICATION" />
    </intent-filter>

</receiver>
```

2. 直接继承 BroadcastReceiver 在 onReceive()中回调

```
class MyBroadcastReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context?, intent: Intent?) {
          var uid= intent?.getStringExtra("uid")
        Log.d(MyBroadcastReceiver.TAG, "onReceive, uid:$uid ")
    }
}
```

在apk安装的时候，由PMS来完成注册过程。

3. 发送广播

```
Intent intent = new Intent();
intent.setAction("com.example.broadcast.MY_NOTIFICATION");
intent.putExtra("uid", "1111111");
// Android8.0(26)，无法接受静态注册的广播，因此需要加入指定包名
intent.setPackage(getPackageName());
sendBroadcast(intent);
```

> 当app静态注册了广播，在未启动的情况下，系统会拉起app。因此，如果多个app静态注册了系统的广播，那么就会拉起很多的app。
> 为了优化性能，在26后静态注册的广播无法接收。建议使用动态注册，或者发送广播时指定包名。

## 2.2 动态注册

1. new IntentFilter 对象

```
MyBroadcastReceiver mReceiver = new MyBroadcastReceiver();
```

2. 通过context 上下文注册

```
IntentFilter intentFilter = new IntentFilter();
intentFilter.addAction("com.example.broadcast.MY_NOTIFICATION");
this.registerReceiver(mReceiver, intentFilter);
```

3. 发送广播

```
Intent intent = new Intent();
intent.setAction("com.example.broadcast.MY_NOTIFICATION");
intent.putExtra("uid", "1111111");
intent.setPackage(getPackageName());
sendBroadcast(intent);
```

4. 解绑广播

```
unregisterReceiver(mReceiver);
```

当上下问不在时，需要销毁广播，以防止内存泄漏。 通过上下文注册的广播，生命周期跟当前的context一致。

在这里，分析的是动态注册过程。

## 2.3 发送广播方式

## 2.3.1 全局广播

- sendOrderedBroadcast(Intent, String)
  发送有序广播，按照顺序传递结果
- sendBroadcast(Intent)
  发送随机广播

## 2.3.2 本地广播

- LocalBroadcastManager.getInstance(context).sendBroadcast 发送本进程的广播，不能跨进程。更高效、更安全。不用担心数据泄漏问题。 注册：

````
IntentFilter intentFilter = new IntentFilter();
intentFilter.addAction("com.example.broadcast.MY_NOTIFICATION");
LocalBroadcastManager.getInstance(getApplicationContext()).registerReceiver(mReceiver,intentFilter);
````

发送：

```
Intent intent = new Intent();
intent.setAction("com.example.broadcast.MY_NOTIFICATION");
intent.putExtra("uid", "1111111");
LocalBroadcastManager.getInstance(getApplicationContext()).sendBroadcast(intent);
```

# 三、注册过程

## 3.1 ContextImpl.registerReceiver()

> ContextImpl.java

```
@Override
public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {
    return registerReceiver(receiver, filter, null, null);
}

@Override
public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
        int flags) {
    return registerReceiver(receiver, filter, null, null, flags);
}
```

### 3.1.1 registerReceiver()

```
@Override
  public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
          String broadcastPermission, Handler scheduler, int flags) {
      return registerReceiverInternal(receiver, getUserId(),
              filter, broadcastPermission, scheduler, getOuterContext(), flags);
  }
```

### 3.1.2 registerReceiverInternal()

```
 private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
            IntentFilter filter, String broadcastPermission,
            Handler scheduler, Context context, int flags) {
        IIntentReceiver rd = null;
        if (receiver != null) {
            // 1 获取 IIntentReceiver binder对象，用来给AMS回调回来用
            if (mPackageInfo != null && context != null) {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = mPackageInfo.getReceiverDispatcher(
                    receiver, context, scheduler,
                    mMainThread.getInstrumentation(), true);
            } else {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = new LoadedApk.ReceiverDispatcher(
                        receiver, context, scheduler, null, true).getIIntentReceiver();
            }
        }
        try {
                // 2 调用AMS的 registerReceiver()
            final Intent intent = ActivityManager.getService().registerReceiver(
                    mMainThread.getApplicationThread(), mBasePackageName, rd, filter,
                    broadcastPermission, userId, flags);
            if (intent != null) {
                intent.setExtrasClassLoader(getClassLoader());
                intent.prepareToEnterProcess();
            }
            return intent;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```

1. 获取 IIntentReceiver binder对象，作为Binder服务用来给AMS回调回来用。ReceiverDispatcher.InnerReceiver 就是
   IIntentReceiver的子类
2. 调用AMS的 registerReceiver()

## 3.2 LoadApk.getReceiverDispatcher

```
public IIntentReceiver getReceiverDispatcher(BroadcastReceiver r,
          Context context, Handler handler,
          Instrumentation instrumentation, boolean registered) {
      synchronized (mReceivers) {
          LoadedApk.ReceiverDispatcher rd = null;
          ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher> map = null;
          // 如果已经注册过了，则从缓存中获取 
          if (registered) {
              map = mReceivers.get(context);
              if (map != null) {
                  rd = map.get(r);
              }
          }
          if (rd == null) {
          // 没有则 new一个 ReceiverDispatcher
              rd = new ReceiverDispatcher(r, context, handler,
                      instrumentation, registered);
              if (registered) {
                  if (map == null) {
                      map = new ArrayMap<BroadcastReceiver, LoadedApk.ReceiverDispatcher>();
                      mReceivers.put(context, map);
                  }
                  map.put(r, rd);
              }
          } else {
          
              rd.validate(context, handler);
          }
          rd.mForgotten = false;
          return rd.getIIntentReceiver();
      }
  }

```

- 首先从缓存中获取
- 没有则new 一个 ReceiverDispatcher对象

ReceiverDispatcher 是 LoadApk中的一个静态内部类。

### 3.2.1 ReceiverDispatcher 构造方法

```
ReceiverDispatcher(BroadcastReceiver receiver, Context context,
                Handler activityThread, Instrumentation instrumentation,
                boolean registered) {
    if (activityThread == null) {
        throw new NullPointerException("Handler must not be null");
    }
    // 创建 InnerReceiver 对象
    mIIntentReceiver = new InnerReceiver(this, !registered);
    mReceiver = receiver;
    mContext = context;
    mActivityThread = activityThread;
    mInstrumentation = instrumentation;
    mRegistered = registered;
    mLocation = new IntentReceiverLeaked(null);
    mLocation.fillInStackTrace();
}
```

创建 InnerReceiver 对象， InnerReceiver 继承了 IIntentReceiver.Stub类。 getIIntentReceiver()方法返回的就是
IIntentReceiver对象。

## 3.3 AMS.registerReceiver()
直接看 AMS的
# 四、AMS端 registerReceiver()
```



```




## 4.1 registerReceiver()

```
public Intent registerReceiver(IApplicationThread caller, String callerPackage,
        IIntentReceiver receiver, IntentFilter filter, String permission, int userId,
        int flags) {
    enforceNotIsolatedCaller("registerReceiver");
    ArrayList<Intent> stickyIntents = null;
    ProcessRecord callerApp = null;
    // ...
    synchronized(this) {
        if (caller != null) {
            callerApp = getRecordForAppLocked(caller);
            ...
            callingUid = callerApp.info.uid;
            callingPid = callerApp.pid;
        } else {
            callerPackage = null;
            callingUid = Binder.getCallingUid();
            callingPid = Binder.getCallingPid();
        }

        instantApp = isInstantApp(callerApp, callerPackage, callingUid);
        userId = mUserController.handleIncomingUser(callingPid, callingUid, userId, true,
                ALLOW_FULL_ONLY, "registerReceiver", callerPackage);
        // 获取从app端传递过来的广播的actions， 一个广播接受者可能会接收多个action，如自定义、系统的开机、网络变更等
        Iterator<String> actions = filter.actionsIterator();
        if (actions == null) {
            ArrayList<String> noAction = new ArrayList<String>(1);
            noAction.add(null);
            actions = noAction.iterator();
        }

        // Collect stickies of users
        int[] userIds = { UserHandle.USER_ALL, UserHandle.getUserId(callingUid) };
        // 开始遍历 actions 
        while (actions.hasNext()) {
            String action = actions.next();
            // 遍历userIds集合 ，一般情况下就一个
            for (int id : userIds) {
                // 得到 map，key是 action，value是该action对应的所有的intent集合  
                ArrayMap<String, ArrayList<Intent>> stickies = mStickyBroadcasts.get(id);
                if (stickies != null) {
                    // 根据action 对应多个intent 
                    ArrayList<Intent> intents = stickies.get(action);
                    if (intents != null) {
                        if (stickyIntents == null) {
                            stickyIntents = new ArrayList<Intent>();
                        }
                        // 加到 stickyIntents 集合
                        stickyIntents.addAll(intents);
                    }
                }
            }
        }
    }

    ArrayList<Intent> allSticky = null;
    if (stickyIntents != null) {
        final ContentResolver resolver = mContext.getContentResolver();
        // Look for any matching sticky broadcasts... 开始查找那些匹配的广播
        for (int i = 0, N = stickyIntents.size(); i < N; i++) {
            Intent intent = stickyIntents.get(i);
            // Don't provided intents that aren't available to instant apps.
            if (instantApp &&
                    (intent.getFlags() & Intent.FLAG_RECEIVER_VISIBLE_TO_INSTANT_APPS) == 0) {
                continue;
            }
            // If intent has scheme "content", it will need to acccess
            // provider that needs to lock mProviderMap in ActivityThread
            // and also it may need to wait application response, so we
            // cannot lock ActivityManagerService here.
            // 如果匹配
            if (filter.match(resolver, intent, true, TAG) >= 0) {
                if (allSticky == null) {
                    allSticky = new ArrayList<Intent>();
                }
                // 加入到 allSticky 集合
                allSticky.add(intent);
            }
        }
    }

    // The first sticky in the list is returned directly back to the client.
    Intent sticky = allSticky != null ? allSticky.get(0) : null;
    if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Register receiver " + filter + ": " + sticky);
    if (receiver == null) {
        return sticky;
    }

    synchronized (this) {
        if (callerApp != null && (callerApp.thread == null
                || callerApp.thread.asBinder() != caller.asBinder())) {
            // Original caller already died
            return null;
        }
        // mRegisteredReceivers 表示所有注册过的广播。 key为receiver Binder对象，value为ReceiverList，ReceiverList
        //是一个集合。
        ReceiverList rl = mRegisteredReceivers.get(receiver.asBinder());
        if (rl == null) {
            rl = new ReceiverList(this, callerApp, callingPid, callingUid,
                    userId, receiver);
            if (rl.app != null) {
                final int totalReceiversForApp = rl.app.receivers.size();
                if (totalReceiversForApp >= MAX_RECEIVERS_ALLOWED_PER_APP) {
                    throw new IllegalStateException("Too many receivers, total of "
                            + totalReceiversForApp + ", registered for pid: "
                            + rl.pid + ", callerPackage: " + callerPackage);
                }
                rl.app.receivers.add(rl);
            } else {
                try {
                    receiver.asBinder().linkToDeath(rl, 0);
                } catch (RemoteException e) {
                    return sticky;
                }
                rl.linkedToDeath = true;
            }
            mRegisteredReceivers.put(receiver.asBinder(), rl);
        } else if (rl.uid != callingUid) {
            throw new IllegalArgumentException(
                    "Receiver requested to register for uid " + callingUid
                    + " was previously registered for uid " + rl.uid
                    + " callerPackage is " + callerPackage);
        } else if (rl.pid != callingPid) {
            throw new IllegalArgumentException(
                    "Receiver requested to register for pid " + callingPid
                    + " was previously registered for pid " + rl.pid
                    + " callerPackage is " + callerPackage);
        } else if (rl.userId != userId) {
            throw new IllegalArgumentException(
                    "Receiver requested to register for user " + userId
                    + " was previously registered for user " + rl.userId
                    + " callerPackage is " + callerPackage);
        }
        BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,
                permission, callingUid, userId, instantApp, visibleToInstantApps);
        if (rl.containsFilter(filter)) {
            Slog.w(TAG, "Receiver with filter " + filter
                    + " already registered for pid " + rl.pid
                    + ", callerPackage is " + callerPackage);
        } else {
            rl.add(bf);
            if (!bf.debugCheck()) {
                Slog.w(TAG, "==> For Dynamic broadcast");
            }
            mReceiverResolver.addFilter(bf);
        }

        // Enqueue broadcasts for all existing stickies that match
        // this filter.
        if (allSticky != null) {
            ArrayList receivers = new ArrayList();
            receivers.add(bf);

            final int stickyCount = allSticky.size();
            for (int i = 0; i < stickyCount; i++) {
                Intent intent = allSticky.get(i);
                BroadcastQueue queue = broadcastQueueForIntent(intent);
                BroadcastRecord r = new BroadcastRecord(queue, intent, null,
                        null, -1, -1, false, null, null, OP_NONE, null, receivers,
                        null, 0, null, null, false, true, true, -1, false,
                        false /* only PRE_BOOT_COMPLETED should be exempt, no stickies */);
                queue.enqueueParallelBroadcastLocked(r);
                queue.scheduleBroadcastsLocked();
            }
        }

        return sticky;
    }
}
```

- mReceiverResolver ： 给已注册的广播接收者解析intent。 持有 BroadcastFilter 引用(IntentFilter的子类)
- ReceiverList：

注册，目前到此结束。

从内存中对象的角度来理解？

我目前的理解： 一个广播接受者，可以对应多个 intent-filter接收器，也就是actions。

mReceiverResolver 持有 BroadcastFilter对象，BroadcastFilter对象又持有 ReceiverList对象， ReceiverList对象 又持有
receiver对象，receiver对象就是 app传递过来的 ServiceDispatcher对象。

目前的疑惑： 广播发送的时候是从哪里寻找的？？

# 五、App端发送广播

## 5.1 ContextImpl.sendBroadcast()

```
 @Override
    public void sendBroadcast(Intent intent) {
        warnIfCallingFromSystemProcess();
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        try {
            intent.prepareToLeaveProcess(this);
            ActivityManager.getService().broadcastIntent(
                    mMainThread.getApplicationThread(), intent, resolvedType, null,
                    Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, null, false, false,
                    getUserId());
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```

其实发送广播，就是发送intent，然后从系统中过滤出来 intent-filter规则一样的intent。

## 5.2 AMS.broadcastIntent()

```
public final int broadcastIntent(IApplicationThread caller,
        Intent intent, String resolvedType, IIntentReceiver resultTo,
        int resultCode, String resultData, Bundle resultExtras,
        String[] requiredPermissions, int appOp, Bundle bOptions,
        boolean serialized, boolean sticky, int userId) {
    enforceNotIsolatedCaller("broadcastIntent");
    
    synchronized(this) {
        // 验证intent合法性
        intent = verifyBroadcastLocked(intent);

        final ProcessRecord callerApp = getRecordForAppLocked(caller);
        final int callingPid = Binder.getCallingPid();
        final int callingUid = Binder.getCallingUid();

        final long origId = Binder.clearCallingIdentity();
        try {
            return broadcastIntentLocked(callerApp,
                    callerApp != null ? callerApp.info.packageName : null,
                    intent, resolvedType, resultTo, resultCode, resultData, resultExtras,
                    requiredPermissions, appOp, bOptions, serialized, sticky,
                    callingPid, callingUid, callingUid, callingPid, userId);
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
}

```
serialized : false，不是有序广播

## 5.3 broadcastIntentLocked()

```
@GuardedBy("this")
final int broadcastIntentLocked(ProcessRecord callerApp,
        String callerPackage, Intent intent, String resolvedType,
        IIntentReceiver resultTo, int resultCode, String resultData,
        Bundle resultExtras, String[] requiredPermissions, int appOp, Bundle bOptions,
        boolean ordered, boolean sticky, int callingPid, int callingUid, int realCallingUid,
        int realCallingPid, int userId) {
    return broadcastIntentLocked(callerApp, callerPackage, intent, resolvedType, resultTo,
        resultCode, resultData, resultExtras, requiredPermissions, appOp, bOptions, ordered,
        sticky, callingPid, callingUid, realCallingUid, realCallingPid, userId,
        false /* allowBackgroundActivityStarts */);
}

```
ordered： false 
## 5.4 broadcastIntentLocked()

```
@GuardedBy("this")
final int broadcastIntentLocked(ProcessRecord callerApp,
        String callerPackage, Intent intent, String resolvedType,
        IIntentReceiver resultTo, int resultCode, String resultData,
        Bundle resultExtras, String[] requiredPermissions, int appOp, Bundle bOptions,
        boolean ordered, boolean sticky, int callingPid, int callingUid, int realCallingUid,
        int realCallingPid, int userId, boolean allowBackgroundActivityStarts) {
        
        //重新创建一个intent 
        intent = new Intent(intent);

         // user用户验证
         
         // 检测是不是只有系统才能够发送的广播？ 是不是系统的广播？   
            
        // 判断是否是用户被移除、包添加或移除、替换
        
         // Figure out who all will receive this broadcast.
         
        List receivers = null;  // 元素类型为 ：ResolveInfo
      // Need to resolve the intent to interested receivers... 
        if ((intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY)
                 == 0) {
                 //解析与这次intent匹配的接收者
            receivers = collectReceiverComponents(intent, resolvedType, callingUid, users);
        }
    
        // 得到所有的静态和动态广播，加入到 receivers列表中
          if ((receivers != null && receivers.size() > 0)
                || resultTo != null) {
            
            // 根据intent 获取 BroadcastQueue 
            BroadcastQueue queue = broadcastQueueForIntent(intent);
            // 创建一个br对象，把 receivers传进去
            BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                    callerPackage, callingPid, callingUid, callerInstantApp, resolvedType,
                    requiredPermissions, appOp, brOptions, receivers, resultTo, resultCode,
                    resultData, resultExtras, ordered, sticky, false, userId,
                    allowBackgroundActivityStarts, timeoutExempt);

            if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Enqueueing ordered broadcast " + r);

            final BroadcastRecord oldRecord =
                    replacePending ? queue.replaceOrderedBroadcastLocked(r) : null;
            if (oldRecord != null) {
                //...
            } else {
                // 把这次发送广播的工作加入到队列中
                queue.enqueueOrderedBroadcastLocked(r);
                // 开始发送
                queue.scheduleBroadcastsLocked();
            }
        } else {
           //...
        }

        return ActivityManager.BROADCAST_SUCCESS;
        

}
```

既然说把所有的静态、动态的广播都加入到 receivers列表中。那么，静态和动态的广播从哪里来的？

跟之前的注册的有什么关系？ 注册？ 到底是注册到什么地方呢？ ProcessRecord？

## 5.5 enqueueOrderedBroadcastLocked()
> BroadcastQueue.java

## 5.6 scheduleBroadcastsLocked()
> BroadcastQueue.java
```
public void scheduleBroadcastsLocked() {
    if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Schedule broadcasts ["
            + mQueueName + "]: current="
            + mBroadcastsScheduled);

    if (mBroadcastsScheduled) {
        return;
    }
    mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
    mBroadcastsScheduled = true;
}
```
发送了一个消息到 handler。
```
 private final class BroadcastHandler extends Handler {
        public BroadcastHandler(Looper looper) {
            super(looper, null, true);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case BROADCAST_INTENT_MSG: {
                    if (DEBUG_BROADCAST) Slog.v(
                            TAG_BROADCAST, "Received BROADCAST_INTENT_MSG ["
                            + mQueueName + "]");
                    processNextBroadcast(true);
                } break;
                case BROADCAST_TIMEOUT_MSG: {
                    synchronized (mService) {
                        broadcastTimeoutLocked(true);
                    }
                } break;
            }
        }
    }
```
## 5.7 processNextBroadcast()
> BroadcastQueue.java
```
final void processNextBroadcast(boolean fromMsg) {
    synchronized (mService) {
        processNextBroadcastLocked(fromMsg, false);
    }
}
```
## 5.8 processNextBroadcastLocked()
> BroadcastQueue.java
```


```























