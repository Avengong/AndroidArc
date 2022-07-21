# windows 开发

flutter 通过通道调用原生端的API，那么 windows端也是通过通道的吧？
是的。
## 如何在 visual studio 中打开项目，便于开发
打开项目/解决方案 路径：
D:\flutter_sdk\frog_zim_plugin\example\build\windows\xxx.sln

## 联系人部分如何选择？ 
是在dart层写逻辑，然后利用第三方的数据库来完成数据的存储？

还是直接适配通道接口，然后在c++层去完成联系人逻辑呢？ 
第一种方案简单，第二种复杂一些，但是可以学习更多东西。
暂定第一种，等完成发送消息后，再去看看有没有时间写。

# 接入腾讯windows的lib
1. 下载库
2. 引入依赖
在哪个路径下引入？ 
- 新建 windows/imsdk目录
- 添加包含目录 在visual studio 的对应的插件目录，右键-属性-c++- 附件包含目录-新建路径：D:\flutter_sdk\frog_zim_plugin\windows\imsdk\include。
- 添加库文件  右键-属性-c++
- 拷贝 DLL 到执行目录 

3. 调用接口 

# 通过CMake引入第三方库 
动态库 dll，静态库 .lib 。

##业务逻辑
1. 初始化
2. 开始登录
3. 开始发送文字消息 

# c++ 语法




















