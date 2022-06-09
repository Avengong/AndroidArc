# gradle 理解

classpath 'com.android.tools.build:gradle:4.1.0'
这个只是编译时期的一些逻辑封装插件。跟asm的类似。而我们需要依赖的语法就是wrapper中的版本。因为wrapper依赖的是gradle
的API，所以插件只是针对Android编译做了特殊的一些逻辑封装。真正的工作则是需要依赖gradle sdk的规则和语法。 因此，如果说编译报错，一般就是wrapper的版本的问题。

同时tools.build.gradle 也需要配合使用一些gradle版本。因为每个版本的gradle SDK的API可能会存在不兼容问题。

wrapper 的缓存路径： /Users/avengong/.gradle/wrapper 其中 .gradle 是隐藏文件夹。需要通过 shift+cmd+.来显示。
 

