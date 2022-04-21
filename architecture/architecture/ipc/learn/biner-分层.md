native层：

我们说过client最核心的是handler server最核心的就是ptr和cookies serviceManager最核心是ptr和cookies

test_client： 类结构： bpHelloService 继承了IInterface，IInterface(模板) 继承了bpRefBase、IHelloService接口。
bpServiceManager 继承了IInterface，IInterface(模板) 继承了bpRefBase、IServiceManager接口。

```
bpRefBase 里面有一个 IBinder *mRemote指针。mRemote指针到底是个什么东西？ 
答： 它指向的是一个bpBinder对象。 BpBinder 继承了 IBinder接口。 

bpBinder里面有一个handle， 这个怎么来的？ 
答：如果是serviceManager，那么handle=0。如果是其他服务，那么就是通过bpServiceManager的getService
返回的flat_binder_obj中的handler。 


```

不管是test_client还是test_server， 都是通过defaultManager来获得serviceManager对象的。

defaultManager内部如何实现呢？ 答： 通过定义在IServiceManager.cpp中的 defaultServiceManager()方法得到：

```
sp<IServiceManager> defaultServiceManager() // 返回值为指向IServiceManager的智能指针
{
    std::call_once(gSmOnce, []() {
        sp<AidlServiceManager> sm = nullptr;
        while (sm == nullptr) {
       //1 ProcessState的getContextObject，传入的参数为ProcessState.cpp中的 getStrongProxyForHandle(0)
       ，传入的是0表示serviceManager。返回一个new BpBinder(handle, trackedUid)对象。而BpBinder继承了IBinder接口
       
       //2 interface_cast方法则会封装BpBinder对象，赋值给mRemote成员变量。得到一个BpServiceManager对象(拥有了
       远端server的接口的能力)
            sm = interface_cast<AidlServiceManager>(ProcessState::self()->getContextObject(nullptr));
            if (sm == nullptr) {
                ALOGE("Waiting 1s on context object on %s.", ProcessState::self()->getDriverName().c_str());
                sleep(1);
            }
        }

        gDefaultServiceManager = sp<ServiceManagerShim>::make(sm);
    });

    return gDefaultServiceManager;
}


```

> frameworks/native/libs/binder/ProcessState.cpp

IBinder 对象其实就是代表service端的某个服务。因此，通过interface_cast(IBinder *binder)把这个对象转换成一个 bpBinder对象。

至此，client得到了sm的远端服务代理对象，BpServiceManager对象，拥有了IServiceManager接口能力。相当于在本地可以直接 调用sm的接口啦！
可以发起请求getService。

那么BpServiceManager是如何发送数据的？ 答： 通过ProcessState 和 PCThreadState 。

内部通过mRemote.transaction()方法。 BpBinder的transaction()方法又会调用

test_server：

open mmap 等操作，因为要面向对象，所以由一个类对象来完成。processState，进程单例对象。 主线程完成open、mmap等操作，子线程可以共享这些fd。

每个线程都可以读取数据ioctl，当然这个动作也应该有一个对象来完成： IPCThreadState，也是个单例模式。 但是是针对线程来说的单例，每个线程有一个对象。

ProcessState：：self()：

```
mDriveFd(open_driver())。赋值操作。IPCThreadState 肯定有一个成员gProcess= processState::self()。

open驱动
mmap映射

//循环体

IPCThreadState::self()->startThreadPool() //创建子线程

IPCThreadState::self()->joinThreadPool() //主线程，进入循环
while{
    读数据 talkWithDrive()-->ioctl(mProcessState->mDriverFD,...)
    解析数据
    处理数据 : executeCommand(int cmd)-> case BR_TRANSACTION：

}


startThreadPool内部： 

通过 spawnPooledThread--》new poolThread()对象，执行run方法。poolThread继承了thread类是一个线程。
最终调用了 IPCThreadState::self()->joinThreadPool()。 

也就是说创建了子线程，然后执行跟主线程一样的逻辑，循环读数据，处理数据等。

addService过程：
不同的服务，构造不同的flat_binder_object。每个服务的.binder/.cookies不一样
收到服务后，会得到flat_binder_object，通过里面的.binder/.cookies来找到对应的服务。
sm->addService("hello",new BnHelloService());
parcel data,reply;

data.writeStrongBinder(service) //service=new BnHelloService();
     flatten_binder(PorcessState::self(),val,this); //val=service=new BnHelloService();
        binder=val;     //binder对象 =val=service=new BnHelloService();
        flat_binder_object obj;  
         IBinder *local=binder-> localBinder();   
        
     
     
remote()->transaction(ADD_SERVICE，data);




server 怎么知道client调用哪个服务？ 
handle？？

怎么调用server提供的函数？ 
code？？




```

```

BBinder: 是真正的binder，与驱动ioctl通信。


```

Java层：

```
IInterface: 这是个什么东西？ 跟serviceManager相关？ 
Binder: 这是binder，有什么用？
IBinder: 这是顶层binder接口，肯定是提供了某些接口，但是是谁来调用这些接口呢？ 
parcel: 容器，装跨进程通信的数据。

使用： 
通过aidl 文件生成： 
IHelloService.java
proxy 给客户端
stub 类 给服务端

test_client： 
main(){
    getService()
    //调用方法
    sayHello
    
}


test_server： 
main(){

    addService()
    //一下不用自己写
    while(){
       读数据
       解析数据
       处理数据
       返回值
    
    }
}



```

在serviceManager获取服务的过程中，java层是如何跟c++层关联的？？

```

ServiceManagerNative.asInterface(binderInternal.getContextObject())

通过binderInternal.getContextObject()；是一个native方法,调到c++层。

c++层中，通过ProcessState::self()->getContextObject(null);
继续到调用getStrongProxyForHandle（0）；内部 b=new BpBinder(0)。BpBinder中的mHandler就是0。

最终，返回一个 javaObjectForBinder(env,b); 内部使用c代码调用newOjbect来创建一个java对象 ：
BinderProxy。其中它的mObject= b=new BpBinder(0)；
这样，java层就引用到了c++层的对象。 


```

ServiceManagerNative.asInterface：

内部创建了一个 ServiceManagerProxy(BinderProxy obj) java 对象

# hello服务里的mRemote如何构造？？ 











