角色角度： binder有五个部分组成： 客户端、服务端、ServiceManager、binder驱动。

分层角度：

    client                              server      java应用层
    -----------------------------------------------

                    ServiceManager                  native层

用户空间 分割线 ==============================================
内核空间                                               
binder/dev驱动                     
================================================

注册addService 流程

获取getService 流程：

java:
通过aidl 生成IBookManager类： 里面有stub抽象类，给服务程序用。实现改接口，实现里面的方法。 proxy类，客户端程序用。

c++： serviceManager BpBinder BBbinder

系统驱动： binder_open binder_mmap binder_ioctrl























 

