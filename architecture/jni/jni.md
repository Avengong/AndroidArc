# jni 原理

java native interface，java本地接口。链接java和c++的桥梁，并不是Android特有的。

疑问： jni到底是怎么做到的java和native互相通信的？ 通过一个映射，再去虚拟机内部寻找， 内部的原理是什么？

Android系统本身哪些地方用到了jni？

app应用如何使用？ system.loadLibrary

## jni 组成类

```
frameworks/base/core/jni/AndroidRuntime.cpp 
该类注册了所有的jni的native方法，在c++这边就是external方法。如下： 

/*
 * JNI-based registration functions.  Note these are properly contained in
 * namespace android.
 */
extern int register_android_app_admin_SecurityLog(JNIEnv* env);
extern int register_android_content_AssetManager(JNIEnv* env);
extern int register_android_util_CharsetUtils(JNIEnv* env);
extern int register_android_util_EventLog(JNIEnv* env);
extern int register_android_util_Log(JNIEnv* env);
extern int register_android_util_MemoryIntArray(JNIEnv* env);
extern int register_android_content_StringBlock(JNIEnv* env);
extern int register_android_content_XmlBlock(JNIEnv* env);
extern int register_android_content_res_ApkAssets(JNIEnv* env);
extern int register_android_graphics_BLASTBufferQueue(JNIEnv* env);
extern int register_android_graphics_SurfaceTexture(JNIEnv* env);
extern int register_android_view_DisplayEventReceiver(JNIEnv* env);
extern int register_android_view_InputApplicationHandle(JNIEnv* env);
extern int register_android_view_InputWindowHandle(JNIEnv* env);
extern int register_android_view_Surface(JNIEnv* env);
extern int register_android_view_SurfaceControl(JNIEnv* env);
extern int register_android_view_SurfaceControlFpsListener(JNIEnv* env);
extern int register_android_view_SurfaceControlHdrLayerInfoListener(JNIEnv* env);
extern int register_android_view_SurfaceSession(JNIEnv* env);
extern int register_android_view_CompositionSamplingListener(JNIEnv* env);
extern int register_android_view_TextureView(JNIEnv* env);
extern int register_android_view_TunnelModeEnabledListener(JNIEnv* env);
extern int register_android_database_CursorWindow(JNIEnv* env);
extern int register_android_database_SQLiteConnection(JNIEnv* env);
extern int register_android_database_SQLiteGlobal(JNIEnv* env);
extern int register_android_database_SQLiteDebug(JNIEnv* env);
extern int register_android_media_MediaMetrics(JNIEnv *env);
extern int register_android_os_Debug(JNIEnv* env);
extern int register_android_os_GraphicsEnvironment(JNIEnv* env);
extern int register_android_os_HidlSupport(JNIEnv* env);
extern int register_android_os_HwBinder(JNIEnv *env);
extern int register_android_os_HwBlob(JNIEnv *env);
extern int register_android_os_HwParcel(JNIEnv *env);
extern int register_android_os_HwRemoteBinder(JNIEnv *env);
extern int register_android_os_NativeHandle(JNIEnv *env);
extern int register_android_os_ServiceManager(JNIEnv *env);
extern int register_android_os_MessageQueue(JNIEnv* env);
extern int register_android_os_Parcel(JNIEnv* env);
extern int register_android_os_PerformanceHintManager(JNIEnv* env);
extern int register_android_os_SELinux(JNIEnv* env);
extern int register_android_os_VintfObject(JNIEnv *env);
extern int register_android_os_VintfRuntimeInfo(JNIEnv *env);
extern int register_android_os_storage_StorageManager(JNIEnv* env);
extern int register_android_os_SystemProperties(JNIEnv *env);
extern int register_android_os_SystemClock(JNIEnv* env);
extern int register_android_os_Trace(JNIEnv* env);
extern int register_android_os_FileObserver(JNIEnv *env);
extern int register_android_os_UEventObserver(JNIEnv* env);
extern int register_android_os_HidlMemory(JNIEnv* env);
extern int register_android_os_MemoryFile(JNIEnv* env);
extern int register_android_os_SharedMemory(JNIEnv* env);
extern int register_android_service_DataLoaderService(JNIEnv* env);
extern int register_android_os_incremental_IncrementalManager(JNIEnv* env);
extern int register_android_net_LocalSocketImpl(JNIEnv* env);
extern int register_android_text_AndroidCharacter(JNIEnv *env);
extern int register_android_text_Hyphenator(JNIEnv *env);
extern int register_android_opengl_classes(JNIEnv *env);
extern int register_android_ddm_DdmHandleNativeHeap(JNIEnv *env);
extern int register_android_backup_BackupDataInput(JNIEnv *env);
extern int register_android_backup_BackupDataOutput(JNIEnv *env);
extern int register_android_backup_FileBackupHelperBase(JNIEnv *env);
extern int register_android_backup_BackupHelperDispatcher(JNIEnv *env);
extern int register_android_app_backup_FullBackup(JNIEnv *env);
extern int register_android_app_Activity(JNIEnv *env);
extern int register_android_app_ActivityThread(JNIEnv *env);
extern int register_android_app_NativeActivity(JNIEnv *env);
extern int register_android_media_RemoteDisplay(JNIEnv *env);
extern int register_android_util_jar_StrictJarFile(JNIEnv* env);
extern int register_android_view_InputChannel(JNIEnv* env);
extern int register_android_view_InputDevice(JNIEnv* env);
extern int register_android_view_InputEventReceiver(JNIEnv* env);
extern int register_android_view_InputEventSender(JNIEnv* env);
extern int register_android_view_InputQueue(JNIEnv* env);
extern int register_android_view_KeyCharacterMap(JNIEnv *env);
extern int register_android_view_KeyEvent(JNIEnv* env);
extern int register_android_view_MotionEvent(JNIEnv* env);
extern int register_android_view_PointerIcon(JNIEnv* env);
extern int register_android_view_VelocityTracker(JNIEnv* env);
extern int register_android_view_VerifiedKeyEvent(JNIEnv* env);
extern int register_android_view_VerifiedMotionEvent(JNIEnv* env);
extern int register_android_content_res_ObbScanner(JNIEnv* env);
extern int register_android_content_res_Configuration(JNIEnv* env);
extern int register_android_animation_PropertyValuesHolder(JNIEnv *env);
extern int register_android_security_Scrypt(JNIEnv *env);
extern int register_com_android_internal_content_F2fsUtils(JNIEnv* env);
extern int register_com_android_internal_content_NativeLibraryHelper(JNIEnv *env);
extern int register_com_android_internal_content_om_OverlayConfig(JNIEnv *env);
extern int register_com_android_internal_net_NetworkUtilsInternal(JNIEnv* env);
extern int register_com_android_internal_os_ClassLoaderFactory(JNIEnv* env);
extern int register_com_android_internal_os_DmabufInfoReader(JNIEnv* env);
extern int register_com_android_internal_os_FuseAppLoop(JNIEnv* env);
extern int register_com_android_internal_os_KernelCpuBpfTracking(JNIEnv* env);
extern int register_com_android_internal_os_KernelCpuTotalBpfMapReader(JNIEnv* env);
extern int register_com_android_internal_os_KernelCpuUidBpfMapReader(JNIEnv *env);
extern int register_com_android_internal_os_KernelSingleProcessCpuThreadReader(JNIEnv* env);
extern int register_com_android_internal_os_KernelSingleUidTimeReader(JNIEnv *env);
extern int register_com_android_internal_os_Zygote(JNIEnv *env);
extern int register_com_android_internal_os_ZygoteCommandBuffer(JNIEnv *env);
extern int register_com_android_internal_os_ZygoteInit(JNIEnv *env);
extern int register_com_android_internal_security_VerityUtils(JNIEnv* env);
extern int register_com_android_internal_util_VirtualRefBasePtr(JNIEnv *env);
extern int register_android_window_WindowInfosListener(JNIEnv* env);

```




















