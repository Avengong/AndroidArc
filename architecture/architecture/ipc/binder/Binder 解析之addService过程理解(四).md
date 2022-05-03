bBinder 解析之 addService 过程理解(三)

# 背景

从addService场景入手分析:此时，Server作为客户端，ServiceManager作为服务端。

# 一、客户端发起请求的过程

java层注册service，通过ServiceManager的addService。最终调用ServiceManagerProxy的addService：

```
     private IBinder mRemote; // 本质上是BinderProxy对象

   public void addService(String name, IBinder service, boolean allowIsolated)
            throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IServiceManager.descriptor);
        data.writeString(name);
        // 把service写入到parcel容器中
        data.writeStrongBinder(service);
        data.writeInt(allowIsolated ? 1 : 0);
        mRemote.transact(ADD_SERVICE_TRANSACTION, data, reply, 0);
        reply.recycle();
        data.recycle();
    }
```

mRemote是什么？ ？

先回答： mRemote 是一个BinderProxy对象。

## 1.1 native层 BpBinder 对象的创建

初始化如下：

```
    private static IServiceManager getIServiceManager() {
        if (sServiceManager != null) {
            return sServiceManager;
        }

        // Find the service manager
        sServiceManager = ServiceManagerNative
                .asInterface(Binder.allowBlocking(BinderInternal.getContextObject()));
        return sServiceManager;
    }
```

BinderInternal的getContextObject():

```
    //返回IBinder 的是IServiceManager的实现 
 public static final native IBinder getContextObject();
``` 

是一个native方法，调用到jni层的方法，最终返回一个native的BpBinder对象。

```
/frameworks/base/core/jni/android_util_Binder.cpp

static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
  {
      sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
      // 返回给java层
      return javaObjectForIBinder(env, b);
  }
```

根据ProcessState单例对象，得到一个BpBinder()。 继续深入ProcessState的getContextObject：

```
//传入的null。0 表示ServiceManager的 handle值。
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
117  {
118      return getStrongProxyForHandle(0);
119  }

```

0 表示ServiceManager进程的 handle值，继续看getStrongProxyForHandle(0):

```
sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
260  {
261      sp<IBinder> result;
262  
263      AutoMutex _l(mLock);
264  
265      handle_entry* e = lookupHandleLocked(handle);
266  
267      if (e != nullptr) {
268          // We need to create a new BpBinder if there isn't currently one, OR we
269          // are unable to acquire a weak reference on this current one.  See comment
270          // in getWeakProxyForHandle() for more info about this.
271          IBinder* b = e->binder;
272             ...
299              // 创建一个BpBinder 对象
300              b = BpBinder::create(handle);
301              e->binder = b;
302              if (b) e->refs = b->getWeakRefs();
303              result = b;
304          } else {
305              // This little bit of nastyness is to allow us to add a primary
306              // reference to the remote proxy when this team doesn't have one
307              // but another team is sending the handle to us.
308              result.force_set(b);
309              e->refs->decWeak(this);
310          }
311      }
312  
313      return result;
314  }

```

至此，通过BpBinder::create(handle)创建了一个BpBinder对象并返回给java层， handle表示ServiceManager的引用。BpBinder
对象里面的mRemote就等于此处handle。

那么是如何返回给java层的呢？ 通过javaObjectForIBinder()。

## 1.2 java层 BinderProxy 对象的创建

```
661  
662  // If the argument is a JavaBBinder, return the Java object that was used to create it.
663  // Otherwise return a BinderProxy for the IBinder. If a previous call was passed the
664  // same IBinder, and the original BinderProxy is still alive, return the same BinderProxy.
     // 如果参数是一个JavaBBinder，那么久返回一个JavaBBinder 的java对象。否则返回 BinderProxy java对象
665  jobject javaObjectForIBinder(JNIEnv* env, const sp<IBinder>& val)
666  {
667      if (val == NULL) return NULL;
668  
669      if (val->checkSubclass(&gBinderOffsets)) {
670          // It's a JavaBBinder created by ibinderForJavaObject. Already has Java object.
671          jobject object = static_cast<JavaBBinder*>(val.get())->object();
672          LOGDEATH("objectForBinder %p: it's our own %p!\n", val.get(), object);
673          return object;
674      }
675  
676      BinderProxyNativeData* nativeData = new BinderProxyNativeData();
677      nativeData->mOrgue = new DeathRecipientList;
678      nativeData->mObject = val;
679         // 反射构造 BinderProxy java对象，同时long nativeData 成员变量指向 native的 BpBinder对象
            //这样，java层就与native层建立了通信通道。
680      jobject object = env->CallStaticObjectMethod(gBinderProxyOffsets.mClass,
681              gBinderProxyOffsets.mGetInstance, (jlong) nativeData, (jlong) val.get());
682      if (env->ExceptionCheck()) {
683          // In the exception case, getInstance still took ownership of nativeData.
684          return NULL;
685      }
686      BinderProxyNativeData* actualNativeData = getBPNativeData(env, object);
687      if (actualNativeData == nativeData) {
688          // Created a new Proxy
689          uint32_t numProxies = gNumProxies.fetch_add(1, std::memory_order_relaxed);
690          uint32_t numLastWarned = gProxiesWarned.load(std::memory_order_relaxed);
691          if (numProxies >= numLastWarned + PROXY_WARN_INTERVAL) {
692              // Multiple threads can get here, make sure only one of them gets to
693              // update the warn counter.
694              if (gProxiesWarned.compare_exchange_strong(numLastWarned,
695                          numLastWarned + PROXY_WARN_INTERVAL, std::memory_order_relaxed)) {
696                  ALOGW("Unexpectedly many live BinderProxies: %d\n", numProxies);
697              }
698          }
699      } else {
700          delete nativeData;
701      }
702  
703      return object;
704  }

```

反射构造 BinderProxy java对象，同时它的long nativeData 成员变量指向 native的 BpBinder对象。 这样，java层就与native层建立了通信通道。

终于，BinderInternal.getContextObject()为我们返回了一个BinderProxy java对象，BinderProxy的mNativeData变量
指向了C++的IBinder对象(其实就是BpBinder)。

# 二、向服务发起请求

## 2.1 java层发起请求

客户端得到BinderProxy对象后，相当于拿到了ServiceManager的引用。可以发起addService请求了。

构造Parcel容易，把数据写入到Parcel容器中。接着调用mRemote.transact(ADD_SERVICE_TRANSACTION, data, reply, 0).
而mRemote就是BinderProxy。

```
 public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
     ...
     try {  // native方法
            return transactNative(code, data, reply, flags);
        } finally {
          ...
        }
 
 }
```

code：需要调用的方法编号。这里对应addService方法。 data：Parcel容器对象。包含发送的数据 reply： 返回接收数据的容器。 flags：一般为0。

transactNative()根据jni注册表可知，对应的jni方法为android_os_BinderProxy_transact

````
 static const JNINativeMethod gBinderProxyMethods[] = {
1429       /* name, signature, funcPtr */
1430      {"pingBinder",          "()Z", (void*)android_os_BinderProxy_pingBinder},
1431      {"isBinderAlive",       "()Z", (void*)android_os_BinderProxy_isBinderAlive},
1432      {"getInterfaceDescriptor", "()Ljava/lang/String;", (void*)android_os_BinderProxy_getInterfaceDescriptor},
1433      {"transactNative",      "(ILandroid/os/Parcel;Landroid/os/Parcel;I)Z", (void*)android_os_BinderProxy_transact},
1434      {"linkToDeath",         "(Landroid/os/IBinder$DeathRecipient;I)V", (void*)android_os_BinderProxy_linkToDeath},
1435      {"unlinkToDeath",       "(Landroid/os/IBinder$DeathRecipient;I)Z", (void*)android_os_BinderProxy_unlinkToDeath},
1436      {"getNativeFinalizer",  "()J", (void*)android_os_BinderProxy_getNativeFinalizer},
1437  };
````

继续深入 android_os_BinderProxy_transact()：

```

static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
1280          jint code, jobject dataObj, jobject replyObj, jint flags) // throws RemoteException
1281  {
            ...
            // mObject 为BinderProxy对象，这里获取到mNative指向的BpBinder对象
            // 因此，target指针指向的就是BpBinder对象
1296      IBinder* target = getBPNativeData(env, obj)->mObject.get();
1297      if (target == NULL) {
1298          jniThrowException(env, "java/lang/IllegalStateException", "Binder has been finalized!");
1299          return JNI_FALSE;
1300      }

1318      //printf("Transact from Java code to %p sending: ", target); data->print();
            // 调用BpBinder的transact方法
1319      status_t err = target->transact(code, *data, reply, flags);

1320      //if (reply) printf("Transact from Java code to %p received: ", target); reply->print();
           ....
1333  
1334      signalExceptionForError(env, obj, err, true /*canThrowRemoteException*/, data->dataSize());
1335      return JNI_FALSE;
1336  }

```

target指针指向的就是BpBinder对象。

## 2.1 native层发起请求

BpBinder的transact方法:
> /frameworks/native/libs/binder/BpBinder.cpp

```
status_t BpBinder::transact(
211      uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
212  {
213      // Once a binder has died, it will never come back to life.
214      if (mAlive) {
215          status_t status = IPCThreadState::self()->transact(
216              mHandle, code, data, reply, flags);
217          if (status == DEAD_OBJECT) mAlive = 0;
218          return status;
219      }
220  
221      return DEAD_OBJECT;
222  }
223  
```

里面调用的是 IPCThreadState单例对象来发起请求。 IPCThreadState是每个线程独有的单例对象。

mHandle ： 表示服务端进程的引用。这里为0，对应ServiceManager进程。

查看它的transact方法：

```
status_t IPCThreadState::transact(int32_t handle,
651                                    uint32_t code, const Parcel& data,
652                                    Parcel* reply, uint32_t flags)
653  {
654      status_t err;
655  
656      flags |= TF_ACCEPT_FDS;
657      ... 
        // 构造数据到mOut变量中 
667      err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, nullptr);
668  
669      if (err != NO_ERROR) {
            //调用出错
670          if (reply) reply->setError(err);
671          return (mLastError = err);
672      }
673  
674      if ((flags & TF_ONE_WAY) == 0) {
675          if (UNLIKELY(mCallRestriction != ProcessState::CallRestriction::NONE)) {
676              if (mCallRestriction == ProcessState::CallRestriction::ERROR_IF_NOT_ONEWAY) {
677                  ALOGE("Process making non-oneway call but is restricted.");
678                  CallStack::logStack("non-oneway call", CallStack::getCurrent(10).get(),
679                      ANDROID_LOG_ERROR);
680              } else /* FATAL_IF_NOT_ONEWAY */ {
681                  LOG_ALWAYS_FATAL("Process may not make oneway calls.");
682              }
683          }
684  
685          #if 0
686          if (code == 4) { // relayout
687              ALOGI(">>>>>> CALLING transaction 4");
688          } else {
689              ALOGI(">>>>>> CALLING transaction %d", code);
690          }
691          #endif
692          if (reply) {
693              err = waitForResponse(reply);
694          } else {
695              Parcel fakeReply;
696              err = waitForResponse(&fakeReply);
697          }
698        
705  
706          
713      } else {
714          err = waitForResponse(nullptr, nullptr);
715      }
716  
717      return err;
718  }

```

构造数据到mOut变量中,mOut变量是parcel类型。

```
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
1026      int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
1027  {
            // 构造binder_transaction_data 对象
1028      binder_transaction_data tr;
1029  
1030      tr.target.ptr = 0; /* Don't pass uninitialized stack data to a remote process */
1031      tr.target.handle = handle; // 目标进程
1032      tr.code = code; //想要调用哪个函数
1033      tr.flags = binderFlags;
1034      tr.cookie = 0;
1035      tr.sender_pid = 0;
1036      tr.sender_euid = 0;
1037  
1038      const status_t err = data.errorCheck();
1039      if (err == NO_ERROR) {
1040          tr.data_size = data.ipcDataSize();
1041          tr.data.ptr.buffer = data.ipcData();
1042          tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t);
1043          tr.data.ptr.offsets = data.ipcObjects();
1044      } else if (statusBuffer) {
1045          tr.flags |= TF_STATUS_CODE;
1046          *statusBuffer = err;
1047          tr.data_size = sizeof(status_t);
1048          tr.data.ptr.buffer = reinterpret_cast<uintptr_t>(statusBuffer);
1049          tr.offsets_size = 0;
1050          tr.data.ptr.offsets = 0;
1051      } else {
1052          return (mLastError = err);
1053      }
1054  
1055      mOut.writeInt32(cmd);
            //把tr写入parcel容器中
1056      mOut.write(&tr, sizeof(tr));
1057  
1058      return NO_ERROR;
1059  }
```

这里只是把数据写到mOut中，那在哪里执行真正的发送逻辑呢？

当调用 IPCThreadState::joinThreadPool()的时候，该binder线程就会调用talkWithDriver()，不断的与驱动通信。

```
void IPCThreadState::joinThreadPool(bool isMain)
585  {
586      LOG_THREADPOOL("**** THREAD %p (PID %d) IS JOINING THE THREAD POOL\n", (void*)pthread_self(), getpid());
587  
588      mOut.writeInt32(isMain ? BC_ENTER_LOOPER : BC_REGISTER_LOOPER);
589  
590      status_t result;
591      do {
592          processPendingDerefs();
593          // now get the next command to be processed, waiting if necessary
            //不断循环 向驱动写入数据 或 从驱动读取数据，如果没有数据，则休眠挂起
594          result = getAndExecuteCommand();
595  
596          if (result < NO_ERROR && result != TIMED_OUT && result != -ECONNREFUSED && result != -EBADF) {
597              ALOGE("getAndExecuteCommand(fd=%d) returned unexpected error %d, aborting",
598                    mProcess->mDriverFD, result);
599              abort();
600          }
601  
602          // Let this thread exit the thread pool if it is no longer
603          // needed and it is not the main process thread.
604          if(result == TIMED_OUT && !isMain) {
605              break;
606          }
607      } while (result != -ECONNREFUSED && result != -EBADF);
608  
609      LOG_THREADPOOL("**** THREAD %p (PID %d) IS LEAVING THE THREAD POOL err=%d\n",
610          (void*)pthread_self(), getpid(), result);
611     // 线程出错，正常情况不会执行到这里
612      mOut.writeInt32(BC_EXIT_LOOPER);
         
613      talkWithDriver(false);
614  }
```

不断循环 向驱动写入数据 或 从驱动读取数据，如果没有数据，则休眠挂起。深入看看 getAndExecuteCommand()：

```
status_t IPCThreadState::getAndExecuteCommand()
492  {
493      status_t result;
494      int32_t cmd;
495      // 不断的读取
496      result = talkWithDriver();
497      if (result >= NO_ERROR) {
498          size_t IN = mIn.dataAvail();
499          if (IN < sizeof(int32_t)) return result;
500          cmd = mIn.readInt32();
501          IF_LOG_COMMANDS() {
502              alog << "Processing top-level Command: "
503                   << getReturnString(cmd) << endl;
504          }
505  
506          pthread_mutex_lock(&mProcess->mThreadCountLock);
507          mProcess->mExecutingThreadsCount++;
508          if (mProcess->mExecutingThreadsCount >= mProcess->mMaxThreads &&
509                  mProcess->mStarvationStartTimeMs == 0) {
510              mProcess->mStarvationStartTimeMs = uptimeMillis();
511          }
512          pthread_mutex_unlock(&mProcess->mThreadCountLock);
513          // 处理数据，执行逻辑
514          result = executeCommand(cmd);
515  
516          pthread_mutex_lock(&mProcess->mThreadCountLock);
517          mProcess->mExecutingThreadsCount--;
518          if (mProcess->mExecutingThreadsCount < mProcess->mMaxThreads &&
519                  mProcess->mStarvationStartTimeMs != 0) {
520              int64_t starvationTimeMs = uptimeMillis() - mProcess->mStarvationStartTimeMs;
521              if (starvationTimeMs > 100) {
522                  ALOGE("binder thread pool (%zu threads) starved for %" PRId64 " ms",
523                        mProcess->mMaxThreads, starvationTimeMs);
524              }
525              mProcess->mStarvationStartTimeMs = 0;
526          }
527          pthread_cond_broadcast(&mProcess->mThreadCountDecrement);
528          pthread_mutex_unlock(&mProcess->mThreadCountLock);
529      }
530  
531      return result;
532  }
```

通过 talkWithDriver()不断的读取数据，executeCommand()不断的处理数据。

继续看： talkWithDriver()：

```
status_t IPCThreadState::talkWithDriver(bool doReceive)
923  {
924      if (mProcess->mDriverFD <= 0) {
925          return -EBADF;
926      }
927       // 构造用户空间的 bwr对象
928      binder_write_read bwr;
         ...
938      bwr.write_size = outAvail;
939      bwr.write_buffer = (uintptr_t)mOut.data();
940  
941      // This is what we'll read.
942      if (doReceive && needRead) {
943          bwr.read_size = mIn.dataCapacity();
944          bwr.read_buffer = (uintptr_t)mIn.data();
945      } else {
946          bwr.read_size = 0;
947          bwr.read_buffer = 0;
948      }
949  
964      // Return immediately if there is nothing to do.
965      if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;
966  
967      bwr.write_consumed = 0;
968      bwr.read_consumed = 0;
969      status_t err;
970      do {
                //重点： 通过ooctl直接与驱动层通信
975          if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
976              err = NO_ERROR;
977          else
978              err = -errno;
979
988      } while (err == -EINTR);
989  
1022      return err;
1023  }
```

bwr中包含： 目的进程的handle，调用函数的序号，数据等内容。 最终，通过ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr)
，实现与驱动读、写数据。

至此，我们明白IPCThreadState会不断从驱动读写数据，那么是什么时候调用 IPCThreadState::joinThreadPool()呢？？

## 2.3 何时开启循环读写数据？

我们知道，在system server进程拉起的时候，会执行到如下代码(不熟悉的，可以看之前的system篇文章)：

```
继续看AndroidRuntime.cpp 的 onZygoteInit()

virtual void onZygoteInit()
{
sp<ProcessState> proc = ProcessState::self();
ALOGV("App process: starting thread pool.\n");
// 开启了Binder线程
proc->startThreadPool();
}
```

继续查看 startThreadPool():

```
void ProcessState::startThreadPool()
164  {
165      AutoMutex _l(mLock);
166      if (!mThreadPoolStarted) {
167          mThreadPoolStarted = true;
              //开启线程
168          spawnPooledThread(true);
169      }
170  }

```

spawnPooledThread():

```
 void ProcessState::spawnPooledThread(bool isMain)
368  {
369      if (mThreadPoolStarted) {
370          String8 name = makeBinderThreadName();
371          ALOGV("Spawning new pooled thread, name=%s\n", name.string());
372          sp<Thread> t = new PoolThread(isMain);
373          t->run(name.string());
374      }
375  }
```

```
 class PoolThread : public Thread
57  {
58  public:
59      explicit PoolThread(bool isMain)
60          : mIsMain(isMain)
61      {
62      }
63  
64  protected:
65      virtual bool threadLoop()
66      {   // 调用了 IPCThreadState的joinThreadPool方法
67          IPCThreadState::self()->joinThreadPool(mIsMain);
68          return false;
69      }
70  
71      const bool mIsMain;
72  };
```

总结： 创建了新线程，并且调用了 IPCThreadState的joinThreadPool方法，通过ioctl()不断的与驱动进行通信。

不过我们要明白一个知识：想要与驱动通信必须要经过三步：

1. open binder驱动
2. mmap()完成虚拟地址与内核物理内存的映射
3. 不断的读取数据

目前只是第三步，那么前面两步是在哪里开始的呢？ 其实是在ProcessState的构造函数中

```
ProcessState::ProcessState(const char *driver)
425      : mDriverName(String8(driver))
426      , mDriverFD(open_driver(driver)) // 内部调用了open函数，打开驱动。
427      , mVMStart(MAP_FAILED)
428      , mThreadCountLock(PTHREAD_MUTEX_INITIALIZER)
429      , mThreadCountDecrement(PTHREAD_COND_INITIALIZER)
430      , mExecutingThreadsCount(0)
431      , mMaxThreads(DEFAULT_MAX_BINDER_THREADS)
432      , mStarvationStartTimeMs(0)
433      , mManagesContexts(false)
434      , mBinderContextCheckFunc(nullptr)
435      , mBinderContextUserData(nullptr)
436      , mThreadPoolStarted(false)
437      , mThreadPoolSeq(1)
438      , mCallRestriction(CallRestriction::NONE)
439  {
440      if (mDriverFD >= 0) {
441          // mmap the binder, providing a chunk of virtual address space to receive transactions.
               // mmap 映射虚拟地址到内核物理空间
442          mVMStart = mmap(nullptr, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
443          if (mVMStart == MAP_FAILED) {
444              // *sigh*
445              ALOGE("Using %s failed: unable to mmap transaction memory.\n", mDriverName.c_str());
446              close(mDriverFD);
447              mDriverFD = -1;
448              mDriverName.clear();
449          }
450      }
451  
452      LOG_ALWAYS_FATAL_IF(mDriverFD < 0, "Binder driver could not be opened.  Terminating.");
453  }

```

内部调用了open函数，打开驱动。并且把驱动的fd赋值给mDriverFD变量。

经历过以上三个步骤，作为客户端终于可以向ServiceManager注册服务了。

至此，我们清楚了作为客户端是如何发送请求到驱动层的。通过ioctl()从用户态进入到内核态，驱动内核会根据我们传入的handle值
来找到ServiceManager进程的内核态，最终从内核态回到ServiceManager的用户态。具体的查找过程，可以看驱动篇文章。

下面来分析ServiceManager进程是如何读取数据的。

# 四、ServiceManager添加服务过程

在ServiceManager进程启动分析的时候，我们知道做了三件事情：

1. open 驱动，完成mmap映射
2. 设置全局binder管理者，handle值对应为0
3. 开启loop循环，不断的从驱动中读取数据，处理数据

尤其是在第三步的解析过程中，如果有数据，则会回调svcmgr_handler函数。 然后根据code来调用对应的函数，如 addService、 getService等。

在这里，当客户端发起addService请求后，首先会在内核构造一个binder_node节点对应该服务。
唤醒ServiceManager进程后，再构造一个binder_ref，其中ref的node节点指向之前的binder_node节点。 分配一个desc,返回到用户态。
构建一个svcInfo对象，包含desc和name。存入链表。 addService指向完毕。

# 五、client调用Server某个服务的过程

当client调用Server某个服务的时候，具体的过程是怎么样的呢？ 内核是如何回到native层的？

## 5.1 java层的Binder类

作为服务端，会继承stub类，然后实现接口方法。 如：

```
IHelloService serviceImpl=  new IHelloService.Stub(){
            @Override
            public void sayHello() throws RemoteException {

            }

            @Override
            public int sayHelloTo(String name) throws RemoteException {
                return 0;
            }
        };
```

而抽象类 stub继承了binder类。我们来看看 Binder类的构造方法：

```
   //指向了 native层的 JavaBBinderHolder 对象。
  private final long mObject;

  public Binder(@Nullable String descriptor)  {
        // 获取native层的 JavaBBinderHolder 对象
        mObject = getNativeBBinderHolder();
        NoImagePreloadHolder.sRegistry.registerNativeAllocation(this, mObject);

        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Binder> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Binder class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }
        mDescriptor = descriptor;
    }
```

调用了jni方法 getNativeBBinderHolder():

## 5.2 native的 JavaBinder

> /frameworks/base/core/jni/android_util_Binder.cpp

```

static jlong android_os_Binder_getNativeBBinderHolder(JNIEnv* env, jobject clazz)
938  {
939      JavaBBinderHolder* jbh = new JavaBBinderHolder();
940      return (jlong) jbh;
941  }
```

JavaBBinderHolder 构造方法：

```
class JavaBBinderHolder
421  {
422  public:
423      sp<JavaBBinder> get(JNIEnv* env, jobject obj)
424      {
425          AutoMutex _l(mLock);
426          sp<JavaBBinder> b = mBinder.promote();
427          if (b == NULL) {
                 // 实例化JavaBBinder 对象 ，赋值给mBinder. obj表示java层的binder对象
428              b = new JavaBBinder(env, obj);
429              mBinder = b;
430              ALOGV("Creating JavaBinder %p (refs %p) for Object %p, weakCount=%" PRId32 "\n",
431                   b.get(), b->getWeakRefs(), obj, b->getWeakRefs()->getWeakCount());
432          }
433  
434          return b;
435      }
436  
437      sp<JavaBBinder> getExisting()
438      {
439          AutoMutex _l(mLock);
440          return mBinder.promote();
441      }
442  
443  private:
444      Mutex           mLock;
445      wp<JavaBBinder> mBinder;
446  };
```

JavaBBinder 继承了 BBinder。

```
class JavaBBinder : public BBinder
302  {
303  public:
304      JavaBBinder(JNIEnv* env, jobject /* Java Binder */ object)
305          : mVM(jnienv_to_javavm(env)), mObject(env->NewGlobalRef(object))
306      {
307          ALOGV("Creating JavaBBinder %p\n", this);
308          gNumLocalRefsCreated.fetch_add(1, std::memory_order_relaxed);
309          gcIfManyNewRefs(env);
310      }
311  
312      bool    checkSubclass(const void* subclassID) const
313      {
314          return subclassID == &gBinderOffsets;
315      }
316  
317      jobject object() const
318      {
319          return mObject; // 表示 java的binder
320      }
321  
322  protected:
323      virtual ~JavaBBinder()
324      {
325          ALOGV("Destroying JavaBBinder %p\n", this);
326          gNumLocalRefsDeleted.fetch_add(1, memory_order_relaxed);
327          JNIEnv* env = javavm_to_jnienv(mVM);
328          env->DeleteGlobalRef(mObject);
329      }
330  
      // 重写了onTransact函数 
      status_t onTransact(
356          uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags = 0) override
357      {
358          JNIEnv* env = javavm_to_jnienv(mVM);
359  
360          ALOGV("onTransact() on %p calling object %p in env %p vm %p\n", this, mObject, env, mVM);
361  
362          IPCThreadState* thread_state = IPCThreadState::self();
363          const int32_t strict_policy_before = thread_state->getStrictModePolicy();
364  
365          //printf("Transact from %p to Java code sending: ", this);
366          //data.print();
367          //printf("\n");
             // 回调java层的 Binder的 onTransact方法
368          jboolean res = env->CallBooleanMethod(mObject, gBinderOffsets.mExecTransact,
369              code, reinterpret_cast<jlong>(&data), reinterpret_cast<jlong>(reply), flags);
370  
371          if (env->ExceptionCheck()) {
372              ScopedLocalRef<jthrowable> excep(env, env->ExceptionOccurred());
373              report_exception(env, excep.get(),
374                  "*** Uncaught remote exception!  "
375                  "(Exceptions are not yet supported across processes.)");
376              res = JNI_FALSE;
377          }
378  
379          // Check if the strict mode state changed while processing the
380          // call.  The Binder state will be restored by the underlying
381          // Binder system in IPCThreadState, however we need to take care
382          // of the parallel Java state as well.
383          if (thread_state->getStrictModePolicy() != strict_policy_before) {
384              set_dalvik_blockguard_policy(env, strict_policy_before);
385          }
386  
387          if (env->ExceptionCheck()) {
388              ScopedLocalRef<jthrowable> excep(env, env->ExceptionOccurred());
389              report_exception(env, excep.get(),
390                  "*** Uncaught exception in onBinderStrictModePolicyChange");
391          }
398  
399          //aout << "onTransact to Java code; result=" << res << endl
400          //    << "Transact from " << this << " to Java code returning "
401          //    << reply << ": " << *reply << endl;
402          return res != JNI_FALSE ? NO_ERROR : UNKNOWN_TRANSACTION;
403      }
```

重点： 1.JavaBBinder重写了onTransact函数。 2.回调java层的 Binder的 onTransact方法
当后续处理完数据后，就会回调到这里，最终就回调到java层的Binder的onTransact方法中去。

至此，我们知道了java层的HelloService服务 是被BBinder的子类JavaBBinder中的mObject所引用的。

那么，什么时候会回调JavaBinder的onTransact()方法呢？

## 5.3 JavaBinder 回调时机

还记得在IPCThreadState中会通过talkWitchDriver()不断从驱动中读取数据，然后处理数据。

```
status_t IPCThreadState::getAndExecuteCommand()
492  {
493      status_t result;
494      int32_t cmd;
495      // 不断的读取
496      result = talkWithDriver();
497      if (result >= NO_ERROR) {
498          size_t IN = mIn.dataAvail();
499          if (IN < sizeof(int32_t)) return result;
500          cmd = mIn.readInt32();
501          IF_LOG_COMMANDS() {
502              alog << "Processing top-level Command: "
503                   << getReturnString(cmd) << endl;
504          }
505  
506          pthread_mutex_lock(&mProcess->mThreadCountLock);
507          mProcess->mExecutingThreadsCount++;
508          if (mProcess->mExecutingThreadsCount >= mProcess->mMaxThreads &&
509                  mProcess->mStarvationStartTimeMs == 0) {
510              mProcess->mStarvationStartTimeMs = uptimeMillis();
511          }
512          pthread_mutex_unlock(&mProcess->mThreadCountLock);
513          // 处理数据，执行逻辑
514          result = executeCommand(cmd);
515  
516          pthread_mutex_lock(&mProcess->mThreadCountLock);
517          mProcess->mExecutingThreadsCount--;
518          if (mProcess->mExecutingThreadsCount < mProcess->mMaxThreads &&
519                  mProcess->mStarvationStartTimeMs != 0) {
520              int64_t starvationTimeMs = uptimeMillis() - mProcess->mStarvationStartTimeMs;
521              if (starvationTimeMs > 100) {
522                  ALOGE("binder thread pool (%zu threads) starved for %" PRId64 " ms",
523                        mProcess->mMaxThreads, starvationTimeMs);
524              }
525              mProcess->mStarvationStartTimeMs = 0;
526          }
527          pthread_cond_broadcast(&mProcess->mThreadCountDecrement);
528          pthread_mutex_unlock(&mProcess->mThreadCountLock);
529      }
530  
531      return result;
532  }
```

在看看 executeCommand ()方法：

```
  status_t IPCThreadState::executeCommand(int32_t cmd)
1069  {
1070      BBinder* obj;
1071      RefBase::weakref_type* refs;
1072      status_t result = NO_ERROR;
1073  
          switch ((uint32_t)cmd) {
            ...
          case BR_TRANSACTION: // 重点关注这里
          
1149          {
1150              binder_transaction_data_secctx tr_secctx;
1151              binder_transaction_data& tr = tr_secctx.transaction_data;
1152  
1153              if (cmd == (int) BR_TRANSACTION_SEC_CTX) {
1154                  result = mIn.read(&tr_secctx, sizeof(tr_secctx));
1155              } else {
1156                  result = mIn.read(&tr, sizeof(tr));
1157                  tr_secctx.secctx = 0;
1158              }
1159  
1160              ALOG_ASSERT(result == NO_ERROR,
1161                  "Not enough command data for brTRANSACTION");
1162              if (result != NO_ERROR) break;
1163  
1164              //Record the fact that we're in a binder call.
1165              mIPCThreadStateBase->pushCurrentState(
1166                  IPCThreadStateBase::CallState::BINDER);
1167              Parcel buffer;
1168              buffer.ipcSetDataReference(
1169                  reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
1170                  tr.data_size,
1171                  reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
1172                  tr.offsets_size/sizeof(binder_size_t), freeBuffer, this);
1173  
1174              const pid_t origPid = mCallingPid;
1175              const char* origSid = mCallingSid;
1176              const uid_t origUid = mCallingUid;
1177              const int32_t origStrictModePolicy = mStrictModePolicy;
1178              const int32_t origTransactionBinderFlags = mLastTransactionBinderFlags;
1179              const int32_t origWorkSource = mWorkSource;
1180              const bool origPropagateWorkSet = mPropagateWorkSource;
1181              // Calling work source will be set by Parcel#enforceInterface. Parcel#enforceInterface
1182              // is only guaranteed to be called for AIDL-generated stubs so we reset the work source
1183              // here to never propagate it.
1184              clearCallingWorkSource();
1185              clearPropagateWorkSource();
1186  
1187              mCallingPid = tr.sender_pid;
1188              mCallingSid = reinterpret_cast<const char*>(tr_secctx.secctx);
1189              mCallingUid = tr.sender_euid;
1190              mLastTransactionBinderFlags = tr.flags;
1191  
1192              // ALOGI(">>>> TRANSACT from pid %d sid %s uid %d\n", mCallingPid,
1193              //    (mCallingSid ? mCallingSid : "<N/A>"), mCallingUid);
1194  
1195              Parcel reply;
1196              status_t error;
1197             
1208              if (tr.target.ptr) {
1209                  // We only have a weak reference on the target object, so we must first try to
1210                  // safely acquire a strong reference before doing anything else with it.
1211                  if (reinterpret_cast<RefBase::weakref_type*>(
1212                          tr.target.ptr)->attemptIncStrong(this)) {
                            // 根据tr中的cookie来生成对应的JavaBBinder对象并调用它的transact方法
1213                      error = reinterpret_cast<BBinder*>(tr.cookie)->transact(tr.code, buffer,
1214                              &reply, tr.flags);
1215                      reinterpret_cast<BBinder*>(tr.cookie)->decStrong(this);
1216                  } else {
1217                      error = UNKNOWN_TRANSACTION;
1218                  }
1219  
1220              } else {
1221                  error = the_context_object->transact(tr.code, buffer, &reply, tr.flags);
1222              }
1223  
1224              mIPCThreadStateBase->popCurrentState();
1225              //ALOGI("<<<< TRANSACT from pid %d restore pid %d sid %s uid %d\n",
1226              //     mCallingPid, origPid, (origSid ? origSid : "<N/A>"), origUid);
1227  
1228              if ((tr.flags & TF_ONE_WAY) == 0) {
1229                  LOG_ONEWAY("Sending reply to %d!", mCallingPid);
1230                  if (error < NO_ERROR) reply.setError(error);
1231                  sendReply(reply, 0);
1232              } else {
1233                  LOG_ONEWAY("NOT sending reply to %d!", mCallingPid);
1234              }
1235  
1236              mCallingPid = origPid;
1237              mCallingSid = origSid;
1238              mCallingUid = origUid;
1239              mStrictModePolicy = origStrictModePolicy;
1240              mLastTransactionBinderFlags = origTransactionBinderFlags;
1241              mWorkSource = origWorkSource;
1242              mPropagateWorkSource = origPropagateWorkSet;
1243  
1244              IF_LOG_TRANSACTIONS() {
1245                  TextOutput::Bundle _b(alog);
1246                  alog << "BC_REPLY thr " << (void*)pthread_self() << " / obj "
1247                      << tr.target.ptr << ": " << indent << reply << dedent << endl;
1248              }
1249  
1250          }
1251          break;
         return result;
1289  }

```

根据tr(binder_transact_data)中的cookie来生成对应的JavaBBinder对象并调用它的transact方法： BBinder的transact方法：

```
status_t BBinder::transact(
124      uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
125  {
126      data.setDataPosition(0);
127  
128      status_t err = NO_ERROR;
129      switch (code) {
130          case PING_TRANSACTION:
131              reply->writeInt32(pingBinder());
132              break;
133          default:
134              err = onTransact(code, data, reply, flags);
135              break;
136      }
137  
138      if (reply != nullptr) {
139          reply->setDataPosition(0);
140      }
141  
142      return err;
143  }
```

会调用 onTransact()方法，会调用到子类JavaBBinder类。根据上面的分析，最终会调到java层的Binder类的onTransact()方法
。而在Sub类类实现了Binder，因此会调用Sub类的onTransact方法。最终调到H额来咯Service的接口方法。

# 六、问题

1. Java层的ServiceManager和native的ServiceManager是什么关系？

答： 二者没有什么必然联系，都是对ServiceManager大管家的的封装。底层都是通过 ProcessState::self()->getContextObject(NULL)
来获得指向ServiceManager的BpBinder对象来完成与ServiceManager的 通信。





















