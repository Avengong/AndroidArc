# 混淆规则

```

一颗星表示只是保持该包下的类名，而子包下的类名还是会被混淆；

  -keep class com.thc.test.*
两颗星表示把本包和所含子包下的类名都保持；

  -keep class com.thc.test.**

既可以保持该包下的类名，又可以保持类里面的内容不被混淆;
  -keep class com.thc.test.*{*;}

既可以保持该包及子包下的类名，又可以保持类里面的内容不被混淆;
 -keep class com.thc.test.**{*;}

避免混淆某个类及内部的成员方法、变量等
 -keep class com.xlpay.sqlite.cache.BaseDaoImpl{*;}
-keep com.tcloud.core.utils.UtilsWrapper{*;}


保持某个类名不被混淆（但是内部内容会被混淆）
  -keep class com.xlpay.sqlite.cache.BaseDaoImpl

```

