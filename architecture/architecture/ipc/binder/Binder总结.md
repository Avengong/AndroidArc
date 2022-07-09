java层

c++层

binder驱动层

服务端： Binder implements IBinder。 服务端肯定是 binder对象的提供者，因此，肯定是stub extends Binder implements
IHelloService。

客户端： 只需要 Proxy 实现 IHelloService接口。Proxy创建的时候 会传入一个BinderRroxy对象，表示服务端Binder对象的代理对象。

BinderRroxy从哪里来？ 答：在创建proxy的时候，从c++创建得到。其实指向的就是c++层的 BpBinder对象。






 

