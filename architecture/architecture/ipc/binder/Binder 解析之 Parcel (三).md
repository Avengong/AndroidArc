# 背景

在继续深入理解binder前，我们很有必要对Parcel理解透彻。

# parcel 为何物？

> /frameworks/base/core/jni/android_os_Parcel.cpp

# parcel 初试

java层有parcel.java类，native也有parcel.cpp。他们之间有什么关系？

## parcel 构造方法

```3
 private Parcel(long nativePtr) {
        if (DEBUG_RECYCLE) {
            mStack = new RuntimeException();
        }
        //Log.i(TAG, "Initializing obj=0x" + Integer.toHexString(obj), mStack);
        init(nativePtr);
    }
```

```
  private void init(long nativePtr) {
        if (nativePtr != 0) {
            //没有native parcel的指针
            mNativePtr = nativePtr;
            mOwnsNativeParcelObject = false;
        } else {
            // 拥有native  parcel的指正
            mNativePtr = nativeCreate();
            mOwnsNativeParcelObject = true;
        }
    }
    
    // jni 方法
    private static native long nativeCreate();
```

## nativeCreate()

```
483  static jlong android_os_Parcel_create(JNIEnv* env, jclass clazz)
484  {
        // 创建c++ parcel对象
485      Parcel* parcel = new Parcel();
         // 返回引用给java 
486      return reinterpret_cast<jlong>(parcel);
487  }
```

## native层parcel的构造方法

```

352  Parcel::Parcel()
353  {
354      LOG_ALLOC("Parcel %p: constructing", this);
355      initState();
356  }
```

```
void Parcel::initState()
2913  {
2914      LOG_ALLOC("Parcel %p: initState", this);
2915      mError = NO_ERROR;
2916      mData = nullptr;
2917      mDataSize = 0;
2918      mDataCapacity = 0;
2919      mDataPos = 0;
2920      ALOGV("initState Setting data size of %p to %zu", this, mDataSize);
2921      ALOGV("initState Setting data pos of %p to %zu", this, mDataPos);
2922      mObjects = nullptr;
2923      mObjectsSize = 0;
2924      mObjectsCapacity = 0;
2925      mNextObjectHint = 0;
2926      mObjectsSorted = false;
2927      mHasFds = false;
2928      mFdsKnown = true;
2929      mAllowFds = true;
2930      mOwner = nullptr;
2931      mOpenAshmemSize = 0;
2932      mWorkSourceRequestHeaderPosition = 0;
2933      mRequestHeaderPresent = false;
2934  
2935      // racing multiple init leads only to multiple identical write
2936      if (gMaxFds == 0) {
2937          struct rlimit result;
2938          if (!getrlimit(RLIMIT_NOFILE, &result)) {
2939              gMaxFds = (size_t)result.rlim_cur;
2940              //ALOGI("parcel fd limit set to %zu", gMaxFds);
2941          } else {
2942              ALOGW("Unable to getrlimit: %s", strerror(errno));
2943              gMaxFds = 1024;
2944          }
2945      }
2946  }
```

## 如何往parcel容器中写入一个binder对象

```
//在parcel容器当前的位置，插入一个binder对象。若有需要，自动增加parcel的容量。
public final void writeStrongBinder(IBinder val) {
        nativeWriteStrongBinder(mNativePtr, val);
    }
```

```
  private static native void nativeWriteStrongBinder(long nativePtr, IBinder val);
```

```
static void android_os_Parcel_writeStrongBinder(JNIEnv* env, jclass clazz, jlong nativePtr, jobject object)
299  {
        // 得到native对应的 parcel对象
300      Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
301      if (parcel != NULL) {
            // 往parcel容器中存入一个binder对象
302          const status_t err = parcel->writeStrongBinder(ibinderForJavaObject(env, object));
303          if (err != NO_ERROR) {
304              signalExceptionForError(env, clazz, err);
305          }
306      }
307  }
```

```
status_t Parcel::writeStrongBinder(const sp<IBinder>& val)
1135  {
1136      return flatten_binder(ProcessState::self(), val, this);
1137  }
1138  
```

```
status_t flatten_binder(const sp<ProcessState>& /*proc*/,
205      const sp<IBinder>& binder, Parcel* out)
206  {
        //构造一个 flat_binder_object，对应一个服务
207      flat_binder_object obj;
208  
209      if (IPCThreadState::self()->backgroundSchedulingDisabled()) {
210          /* minimum priority for all nodes is nice 0 */
211          obj.flags = FLAT_BINDER_FLAG_ACCEPTS_FDS;
212      } else {
213          /* minimum priority for all nodes is MAX_NICE(19) */
214          obj.flags = 0x13 | FLAT_BINDER_FLAG_ACCEPTS_FDS;
215      }
216  
217      if (binder != nullptr) {
            // 得到服务在native层的对象，JavaBBinder(该对象是在Binder.java构造函数中初始化的)
218          BBinder *local = binder->localBinder();
219          if (!local) {
220              BpBinder *proxy = binder->remoteBinder();
221              if (proxy == nullptr) {
222                  ALOGE("null proxy");
223              }
224              const int32_t handle = proxy ? proxy->handle() : 0;
225              obj.hdr.type = BINDER_TYPE_HANDLE;
226              obj.binder = 0; /* Don't pass uninitialized stack data to a remote process */
227              obj.handle = handle;
228              obj.cookie = 0;
229          } else {
230              if (local->isRequestingSid()) {
231                  obj.flags |= FLAT_BINDER_FLAG_TXN_SECURITY_CTX;
232              }
                // 把binder的引用写入
233              obj.hdr.type = BINDER_TYPE_BINDER;
234              obj.binder = reinterpret_cast<uintptr_t>(local->getWeakRefs());
235              obj.cookie = reinterpret_cast<uintptr_t>(local);
236          }
237      } else {
238          obj.hdr.type = BINDER_TYPE_BINDER;
239          obj.binder = 0;
240          obj.cookie = 0;
241      }
242  
243      return finish_flatten_binder(binder, obj, out);
244  }
```

```
inline static status_t finish_flatten_binder(
199      const sp<IBinder>& /*binder*/, const flat_binder_object& flat, Parcel* out)
200  {  //把binder对象转换成flat_bidner_obj，再往parcel中写入一个flat_binder_obj。
        // 后续肯定是从parcel中的得到flat_binder_obj，恢复成binder对象？
201      return out->writeObject(flat, false);
202  }
```

## 如何从parcel容器中获取一个binder对象

```
//从parcel 当前位置读取一个对象
public final IBinder readStrongBinder() {
        return nativeReadStrongBinder(mNativePtr);
    }
```

```
 private static native IBinder nativeReadStrongBinder(long nativePtr);
```

native ：

```
static jobject android_os_Parcel_readStrongBinder(JNIEnv* env, jclass clazz, jlong nativePtr)
462  {
463      Parcel* parcel = reinterpret_cast<Parcel*>(nativePtr);
464      if (parcel != NULL) {
            
465          return javaObjectForIBinder(env, parcel->readStrongBinder());
466      }
467      return NULL;
468  }
```

readStrongBinder():

```
sp<IBinder> Parcel::readStrongBinder() const
2214  {
2215      sp<IBinder> val;
2216      // Note that a lot of code in Android reads binders by hand with this
2217      // method, and that code has historically been ok with getting nullptr
2218      // back (while ignoring error codes).
2219      readNullableStrongBinder(&val);
2220      return val;
2221  }
```

readNullableStrongBinder():

```
status_t Parcel::readNullableStrongBinder(sp<IBinder>* val) const
2209  {
2210      return unflatten_binder(ProcessState::self(), *this, val);
2211  }
```

从parcel容器对象中获取一个binder对象。

```
 status_t unflatten_binder(const sp<ProcessState>& proc,
303      const Parcel& in, sp<IBinder>* out)
304  {
        // 从parcel中得到一个flat_binder_object对象
305      const flat_binder_object* flat = in.readObject(false);
306  
307      if (flat) {
308          switch (flat->hdr.type) {
309              case BINDER_TYPE_BINDER:
                     // 类型是binder，根据cookies解析出来一个binder对象。
310                  *out = reinterpret_cast<IBinder*>(flat->cookie);
311                  return finish_unflatten_binder(nullptr, *flat, in);
312              case BINDER_TYPE_HANDLE: //类型是handle则计息一个BpBinder代理对象。
313                  *out = proc->getStrongProxyForHandle(flat->handle);
314                  return finish_unflatten_binder(
315                      static_cast<BpBinder*>(out->get()), *flat, in);
316          }
317      }
318      return BAD_TYPE;
319  }
```

从parcel中获取flat_binder_object对象，从flat_binder_object中获得IBinder对象。

```
inline static status_t finish_unflatten_binder(
296      BpBinder* /*proxy*/, const flat_binder_object& /*flat*/,
297      const Parcel& /*in*/)
298  {
299      return NO_ERROR;
300  }
```

# jni与java的jni映射

```
static const JNINativeMethod gParcelMethods[] = {
694      // @CriticalNative
695      {"nativeDataSize",            "(J)I", (void*)android_os_Parcel_dataSize},
696      // @CriticalNative
697      {"nativeDataAvail",           "(J)I", (void*)android_os_Parcel_dataAvail},
698      // @CriticalNative
699      {"nativeDataPosition",        "(J)I", (void*)android_os_Parcel_dataPosition},
700      // @CriticalNative
701      {"nativeDataCapacity",        "(J)I", (void*)android_os_Parcel_dataCapacity},
702      // @FastNative
703      {"nativeSetDataSize",         "(JI)J", (void*)android_os_Parcel_setDataSize},
704      // @CriticalNative
705      {"nativeSetDataPosition",     "(JI)V", (void*)android_os_Parcel_setDataPosition},
706      // @FastNative
707      {"nativeSetDataCapacity",     "(JI)V", (void*)android_os_Parcel_setDataCapacity},
708  
709      // @CriticalNative
710      {"nativePushAllowFds",        "(JZ)Z", (void*)android_os_Parcel_pushAllowFds},
711      // @CriticalNative
712      {"nativeRestoreAllowFds",     "(JZ)V", (void*)android_os_Parcel_restoreAllowFds},
713  
714      {"nativeWriteByteArray",      "(J[BII)V", (void*)android_os_Parcel_writeByteArray},
715      {"nativeWriteBlob",           "(J[BII)V", (void*)android_os_Parcel_writeBlob},
716      // @FastNative
717      {"nativeWriteInt",            "(JI)V", (void*)android_os_Parcel_writeInt},
718      // @FastNative
719      {"nativeWriteLong",           "(JJ)V", (void*)android_os_Parcel_writeLong},
720      // @FastNative
721      {"nativeWriteFloat",          "(JF)V", (void*)android_os_Parcel_writeFloat},
722      // @FastNative
723      {"nativeWriteDouble",         "(JD)V", (void*)android_os_Parcel_writeDouble},
724      {"nativeWriteString",         "(JLjava/lang/String;)V", (void*)android_os_Parcel_writeString},
725      {"nativeWriteStrongBinder",   "(JLandroid/os/IBinder;)V", (void*)android_os_Parcel_writeStrongBinder},
726      {"nativeWriteFileDescriptor", "(JLjava/io/FileDescriptor;)J", (void*)android_os_Parcel_writeFileDescriptor},
727  
728      {"nativeCreateByteArray",     "(J)[B", (void*)android_os_Parcel_createByteArray},
729      {"nativeReadByteArray",       "(J[BI)Z", (void*)android_os_Parcel_readByteArray},
730      {"nativeReadBlob",            "(J)[B", (void*)android_os_Parcel_readBlob},
731      // @CriticalNative
732      {"nativeReadInt",             "(J)I", (void*)android_os_Parcel_readInt},
733      // @CriticalNative
734      {"nativeReadLong",            "(J)J", (void*)android_os_Parcel_readLong},
735      // @CriticalNative
736      {"nativeReadFloat",           "(J)F", (void*)android_os_Parcel_readFloat},
737      // @CriticalNative
738      {"nativeReadDouble",          "(J)D", (void*)android_os_Parcel_readDouble},
739      {"nativeReadString",          "(J)Ljava/lang/String;", (void*)android_os_Parcel_readString},
740      {"nativeReadStrongBinder",    "(J)Landroid/os/IBinder;", (void*)android_os_Parcel_readStrongBinder},
741      {"nativeReadFileDescriptor",  "(J)Ljava/io/FileDescriptor;", (void*)android_os_Parcel_readFileDescriptor},
742  
743      {"nativeCreate",              "()J", (void*)android_os_Parcel_create},
744      {"nativeFreeBuffer",          "(J)J", (void*)android_os_Parcel_freeBuffer},
745      {"nativeDestroy",             "(J)V", (void*)android_os_Parcel_destroy},
746  
747      {"nativeMarshall",            "(J)[B", (void*)android_os_Parcel_marshall},
748      {"nativeUnmarshall",          "(J[BII)J", (void*)android_os_Parcel_unmarshall},
749      {"nativeCompareData",         "(JJ)I", (void*)android_os_Parcel_compareData},
750      {"nativeAppendFrom",          "(JJII)J", (void*)android_os_Parcel_appendFrom},
751      // @CriticalNative
752      {"nativeHasFileDescriptors",  "(J)Z", (void*)android_os_Parcel_hasFileDescriptors},
753      {"nativeWriteInterfaceToken", "(JLjava/lang/String;)V", (void*)android_os_Parcel_writeInterfaceToken},
754      {"nativeEnforceInterface",    "(JLjava/lang/String;)V", (void*)android_os_Parcel_enforceInterface},
755  
756      {"getGlobalAllocSize",        "()J", (void*)android_os_Parcel_getGlobalAllocSize},
757      {"getGlobalAllocCount",       "()J", (void*)android_os_Parcel_getGlobalAllocCount},
758  
759      // @CriticalNative
760      {"nativeGetBlobAshmemSize",       "(J)J", (void*)android_os_Parcel_getBlobAshmemSize},
761  
762      // @CriticalNative
763      {"nativeReadCallingWorkSourceUid", "(J)I", (void*)android_os_Parcel_readCallingWorkSourceUid},
764      // @CriticalNative
765      {"nativeReplaceCallingWorkSourceUid", "(JI)Z", (void*)android_os_Parcel_replaceCallingWorkSourceUid},
766  };
```

从以上可以得知，parcel 对象除了支持读写基本数据，如int、string、long、double等之外，还支持读写binder对象。

总结： parcel就是一个数据容器。我们可以通过把数据写入parcel，它会自动帮我们完成序列化，打包，然后解包。 按数据写入的顺序 来解析数据，否则会失败。