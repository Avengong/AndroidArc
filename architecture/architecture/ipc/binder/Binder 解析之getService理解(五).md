Binder 解析之getService理解(四)

# 背景

上篇文章从服务端角度出发，分析了addService的过程。 这次从客户端角度出发继续分析如何获取服务，即getService的过程。

client作为客户端， ServiceManager作为服务端。

client从SM中那里能获取到的只有一个在client内核中的一个binder_ref。ref包含了两个字段一个是handle用来构造BpBinder，
一个是node指向了server。因此，这个handle也是代表了server进程。 所以在client从sm中的reply中读取数据，解析应该就是BpBinder。

# client想SM 发起获取服务binder本地代理的请求

根据名字发起。

# Java层getService

ServiceManager.java 中：

```
  public static IBinder getService(String name) {
        try {
            IBinder service = sCache.get(name);
            if (service != null) {
                return service;
            } else {
                return Binder.allowBlocking(getIServiceManager().getService(name));
            }
        } catch (RemoteException e) {
            Log.e(TAG, "error in getService", e);
        }
        return null;
    }
```

getIServiceManager()方法：

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

经过上篇文章分析，我们知道肯定调用的是ServiceManagerProxy类的addService方法：

```
   public IBinder getService(String name) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IServiceManager.descriptor);
        data.writeString(name);// 写入服务的名字 
        mRemote.transact(GET_SERVICE_TRANSACTION, data, reply, 0);
        
        //从reply中读取 readStrongBinder得到binder
        IBinder binder = reply.readStrongBinder();
        reply.recycle(); // 回收parcel
        data.recycle();// 回收parcel
        return binder;
    }
```

mRemote 其实是BinderProxy对象，最终会调用transactNative()。

至此，java层的调用基本结束。 transactNative()最终会调用到native层的BpBinder()。

# native层 getService

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

先看 IPCThreadState::self()：

```

IPCThreadState* IPCThreadState::self()
287  {
288      if (gHaveTLS) {
289  restart:
290          const pthread_key_t k = gTLS;
291          IPCThreadState* st = (IPCThreadState*)pthread_getspecific(k);
              // 如果TLS中有，则返回，没有就创建
292          if (st) return st;
293          return new IPCThreadState;
294      }
295  
296      if (gShutdown) {
297          ALOGW("Calling IPCThreadState::self() during shutdown is dangerous, expect a crash.\n");
298          return nullptr;
299      }
300  
301      pthread_mutex_lock(&gTLSMutex);
302      if (!gHaveTLS) {
303          int key_create_value = pthread_key_create(&gTLS, threadDestructor);
304          if (key_create_value != 0) {
305              pthread_mutex_unlock(&gTLSMutex);
306              ALOGW("IPCThreadState::self() unable to create TLS key, expect a crash: %s\n",
307                      strerror(key_create_value));
308              return nullptr;
309          }
310          gHaveTLS = true;
311      }
312      pthread_mutex_unlock(&gTLSMutex);
313      goto restart;
314  }
315  
```

IPCThreadState会把自己存储在TLS线程私有内存空间中，因此是线程私有的。

再看 IPCThreadState的构造方法：

```
 IPCThreadState::IPCThreadState()
803      : mProcess(ProcessState::self()),
804        mWorkSource(kUnsetWorkSource),
805        mPropagateWorkSource(false),
806        mStrictModePolicy(0),
807        mLastTransactionBinderFlags(0),
808        mCallRestriction(mProcess->mCallRestriction)
809  {
        // 存入到TLS中
810      pthread_setspecific(gTLS, this);
811      clearCaller();
812      mIn.setDataCapacity(256); //输入结构体
813      mOut.setDataCapacity(256); // 输出结构体 
814      mIPCThreadStateBase = IPCThreadStateBase::self();
815  }
```

ProcessState::self() 获取进程单例对象。 我们来看ProcessState的构造方法：

```
ProcessState::ProcessState(const char *driver)
425      : mDriverName(String8(driver))
426      , mDriverFD(open_driver(driver)) //打开驱动
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
              // 进行映射mmap，BINDER_VM_SIZE=1M-8k
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
454  
```

其中我们需要注意：BINDER_VM_SIZE： BINDER_VM_SIZE ((1 * 1024 * 1024) - sysconf(_SC_PAGE_SIZE) * 2)
也就是1M-8K 缓冲区大小。 因此，在跨进程调用的的时候，数据不能超过这个大小。

至此，IPCThreadState中会持有 ProcessState的引用。 我们回到IPCThreadState的transact()方法：

```
status_t IPCThreadState::transact(int32_t handle,
                                uint32_t code, const Parcel& data,
652                                    Parcel* reply, uint32_t flags)
653  {
653  {
654      status_t err;
655  
656      flags |= TF_ACCEPT_FDS;
664  
665      LOG_ONEWAY(">>>> SEND from pid %d uid %d %s", getpid(), getuid(),
666          (flags & TF_ONE_WAY) == 0 ? "READ REPLY" : "ONE WAY");
          // 把输入赋值给mOut
667      err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, nullptr);
668  
669      if (err != NO_ERROR) {
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
                    //等待返回
693              err = waitForResponse(reply);
694          } else {
695              Parcel fakeReply;
696              err = waitForResponse(&fakeReply);
697          }
698        
713      } else {
714          err = waitForResponse(nullptr, nullptr);
715      }
717      return err;
718  }
```

1. writeTransactionData() 把数据写到mOut对象中。
2. waitForResponse():

```
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
832  {
833      uint32_t cmd;
834      int32_t err;
835     // 循环 
836      while (1) {
             // 不断的从驱动 读写数据
837          if ((err=talkWithDriver()) < NO_ERROR) break;
838          err = mIn.errorCheck();
839          if (err < NO_ERROR) break;
840          if (mIn.dataAvail() == 0) continue;
841  
842          cmd = (uint32_t)mIn.readInt32();
843  
844          IF_LOG_COMMANDS() {
845              alog << "Processing waitForResponse Command: "
846                  << getReturnString(cmd) << endl;
847          }
848  
849          switch (cmd) {
850          case BR_TRANSACTION_COMPLETE:  ...
854          case BR_DEAD_REPLY:   ...
858          case BR_FAILED_REPLY:  ...
862          case BR_ACQUIRE_RESULT: ...
871          case BR_REPLY:
872              {
873                 // 构造对象，用来接收数据 
                    binder_transaction_data tr;
                     //从parcel mIn中读取数据
874                  err = mIn.read(&tr, sizeof(tr));
875                  ALOG_ASSERT(err == NO_ERROR, "Not enough command data for brREPLY");
876                  if (err != NO_ERROR) goto finish;
877  
878                  if (reply) {
879                      if ((tr.flags & TF_STATUS_CODE) == 0) {
880                          reply->ipcSetDataReference(
881                              reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
882                              tr.data_size,
883                              reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
884                              tr.offsets_size/sizeof(binder_size_t),
885                              freeBuffer, this);
886                      } else {
887                          err = *reinterpret_cast<const status_t*>(tr.data.ptr.buffer);
888                          freeBuffer(nullptr,
889                              reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
890                              tr.data_size,
891                              reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
892                              tr.offsets_size/sizeof(binder_size_t), this);
893                      }
894                  } else {
895                      freeBuffer(nullptr,
896                          reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
897                          tr.data_size,
898                          reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
899                          tr.offsets_size/sizeof(binder_size_t), this);
900                      continue;
901                  }
902              }
903              goto finish;
904  
905          default:
906              err = executeCommand(cmd);
907              if (err != NO_ERROR) goto finish;
908              break;
909          }
910      }
911  
912  finish:
913      if (err != NO_ERROR) {
914          if (acquireResult) *acquireResult = err;
915          if (reply) reply->setError(err);
916          mLastError = err;
917      }
918  
919      return err;
920  }
```

talkWithDriver():

```
status_t IPCThreadState::talkWithDriver(bool doReceive)
923  {
924      if (mProcess->mDriverFD <= 0) {
925          return -EBADF;
926      }
927  
928      binder_write_read bwr;
929  
930      // Is the read buffer empty?
931      const bool needRead = mIn.dataPosition() >= mIn.dataSize();
932  
933      // We don't want to write anything if we are still reading
934      // from data left in the input buffer and the caller
935      // has requested to read the next data.
936      const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;
937  
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
949       ...
967      bwr.write_consumed = 0;
968      bwr.read_consumed = 0;
969      status_t err;
970      do {
                // ioctl 不断的从驱动读写数据     
975          if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
976              err = NO_ERROR;
977          else
978              err = -errno;
979  #else
980          err = INVALID_OPERATION;
981  #endif
982          if (mProcess->mDriverFD <= 0) {
983              err = -EBADF;
984          }
985          IF_LOG_COMMANDS() {
986              alog << "Finished read/write, write size = " << mOut.dataSize() << endl;
987          }
988      } while (err == -EINTR);
989  
994      }
995  
996      if (err >= NO_ERROR) {
997          if (bwr.write_consumed > 0) {
998              if (bwr.write_consumed < mOut.dataSize())
999                  mOut.remove(0, bwr.write_consumed);
1000              else {
1001                  mOut.setDataSize(0);
1002                  processPostWriteDerefs();
1003              }
1004          }
1005          if (bwr.read_consumed > 0) {
                  // 读到了数据
1006              mIn.setDataSize(bwr.read_consumed);
1007              mIn.setDataPosition(0);
1008          }
1009       
1019          return NO_ERROR;
1020      }
1021  
1022      return err;
1023  }
```

最终，getService() 通过talkWitchDriver()不断的往驱动中读写数据。

# ServiceManager进程的响应

client 的getService()通过往binder驱动写入数据后， 最终达到ServiceManager进程。

1. 在SM的用户空间根据name，从svcInfos链表中找到该服务对应的desc。
2. 进入内核根据desc找到对应的binder_ref，获取里面的node节点(就是我们要找的服务)，该node节点中的proc字段就指向了Server进程。
3. 为client新建binder_ref节点，其中的node字段指向上一步找到的node节点，同时分配desc
4. 唤醒client进程，SM进入休眠
5. client进程被唤醒，用户空间得到刚才的desc，也就是handle，对应要找的服务service。
6. 后续向该服务发起请求，则根据handle值构建BpBinder对象，最终通过IPCThreadState来完成通信。





