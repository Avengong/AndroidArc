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

## 3.1 Context.registerReceiver()

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
          // 从缓存中获取 
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

# 四、AMS端 registerReceiver()
























