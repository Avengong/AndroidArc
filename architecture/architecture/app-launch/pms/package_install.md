# 安装过程

疑问： 1, 一个apk是怎么被安装的？

当 PMS 启动后，本质上所有的包都应该被扫描出来了。也就是说从静态的文件包 转变成了内存的数据结构。 那么先看下这个转变的过程是如何转变的吧。 所以，先看扫描。

> frameworks/base/core/java/android/content/pm/IPackageInstaller.aidl

```
 interface IPackageInstaller {
31      int createSession(in PackageInstaller.SessionParams params, String installerPackageName, int userId);
32  
33      void updateSessionAppIcon(int sessionId, in Bitmap appIcon);
34      void updateSessionAppLabel(int sessionId, String appLabel);
35  
36      void abandonSession(int sessionId);
37  
38      IPackageInstallerSession openSession(int sessionId);
39  
40      PackageInstaller.SessionInfo getSessionInfo(int sessionId);
41  
42      ParceledListSlice getAllSessions(int userId);
43      ParceledListSlice getMySessions(String installerPackageName, int userId);
44  
45      ParceledListSlice getStagedSessions();
46  
47      void registerCallback(IPackageInstallerCallback callback, int userId);
48      void unregisterCallback(IPackageInstallerCallback callback);
49  
50      @UnsupportedAppUsage
51      void uninstall(in VersionedPackage versionedPackage, String callerPackageName, int flags,
52              in IntentSender statusReceiver, int userId);
53  
54      void installExistingPackage(String packageName, int installFlags, int installReason,
55              in IntentSender statusReceiver, int userId, in List<String> whiteListedPermissions);
56  
57      void setPermissionsResult(int sessionId, boolean accepted);
58  }
```
















