大体方向是理清楚静态apk与内存中的apk数据结构之间的对应关系，然后在去分析流程。 为什么要这样的顺序？ 因为解析的过程本质上是得到内存中的数据结构，然后往内存中数据结构中填充数据。
我们最关心的就是内部数据结构的变化。

# 解析中的数据结构

```



```

# PMS的扫描包过程

本质上我们关心apk的扫描过程。更本质我们关心从静态apk文件 转变成内存中的数据结构是什么样的！

不管是 system/framework分区 还是 data/分区。都会调用这个方法来进行扫描。

# scanDirTracedLI()

> xxxLI结尾的方法表明需要 mInstaller 锁。 xxxLP结尾的方法表明需要 mPackages 锁。

```
  private void scanDirTracedLI(File scanDir, final int parseFlags, int scanFlags, long currentTime) {
        Trace.traceBegin(TRACE_TAG_PACKAGE_MANAGER, "scanDir [" + scanDir.getAbsolutePath() + "]");
        try {
            scanDirLI(scanDir, parseFlags, scanFlags, currentTime);
        } finally {
            Trace.traceEnd(TRACE_TAG_PACKAGE_MANAGER);
        }
    }
```

这个方法仅仅追加了trace日志。 scanDir： parseFlags： scanFlags： currentTime：

# 解析

最终通过 PackageParse 解析manifest中的所有标签。如 user-permission、application、以及application里面的activity
、service、provider、broadcast等。

```aidl

```

静态的文件形式，最终得以转变为内存中的数据结构。PMS就可以愉快的管理起来了！









