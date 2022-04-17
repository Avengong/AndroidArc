第6课第四节

ipc和rpc：

a，ipc：进程间的通信 场景： A进程把数据发送给B进程。 源： A 进程 数据： 数据 目的： 向哪个进程发送数据呢？

b，rpc： 远程调用

```
场景： A进程调用B进程的方法
因为A进程不能直接操作驱动。所以只能：
封装数据 发送给B。
|
v
怎么发送？ 通过ipc的方式。
B进程收到数据后
解析数据。
```

三者做了什么事情

serviceManager： 1， open 驱动 2，告诉驱动程序，我就是大管家。我就是SM 4， while循环 读驱动获取数据 5， 解析数据 调用：

```
a，addService，在链表中记录名字；
b，查询服务 getService，从链表中查找服务；返回 "server进程"的handle,是一个整数号，对应某个进程。
```

server： 1，open驱动 2，注册服务。 向sm发送服务名字 3，while循环，读驱动。没数据就休眠 4，解析数据，调用 对应的函数。

client： 1，open驱动 2，查询服务。把数据发给谁呢？本质上是在找目的进程。 向SM查询服务，得到server进程的handle整数号。 3，向handle发送数据

总结： 每个server进程(包括serviceManager)内部对应一个handle，在循环从binder驱动中读取数据后，经过解析，最后让handle来
处理数据。因此，client只要获取到server的handle，向它发送数据即可。

serviceManager的handle是0。

# 注册服务的过程

a. binder_open b. svcmgr_publish--> binder_call(bs,&msg,&reply,0,SVC_MGR_ADD_SERVICE)
bs：驱动的fd msg：含有服务的名字信息 reply：SM会回复的信息 target=0。表示是SM。 code：函数编号，addService

# 获取服务的过程

a. binder_open b. svcmgr_lookup--> binder_call(bs,&msg,&reply,0,SVC_MGR_CHECK_SERVICE); bs：驱动的fd
msg：含有服务的名字信息 reply：SM会回复的信息 target=0。表示是SM。 code：函数编号，getService函数

# binder_call 在binder.c中

# binder.c分析：

```
binder_call(struct binder_state *bs,struct binder_io *msg,struct binder_io *reply,uint_t target,
uint_t code)
向谁发数据  target
调用哪个函数 code
提供什么参数 msg
返回值是什么 reply
```

binder_call怎么实现？ a. 构造数据 ，存在buffer，不可能直接把buffer发送出去。应该要有一个结构体来描述这个buffer，如binder_io。 转换的过程：怎么转换？
binder_io： | v bwr： b. 调用某个ioctl来发送数据

```
res=ioctl(bs->fd,BINDER_WRITE_READ,&bwr)
bwr： struct binder_write_read bwr。  bwr具体是怎么定义的？ 在内核中定义。
```

问题来了： 我们构造的数据是binder_io，而驱动内核需要的是bwr结构体。中间肯定有个转换的过程！

c. ioctl也会收数据， 收到bwr。 也要转换为 binder_io ，应用程序才会认识内部的数据。

=====理论结束，下面开始看代码，来验证这个流程==========

# 代码深入

要使用binder_call，肯定要构造数据binder_io： binder_io指向一个缓冲区，这个缓冲区就是iodata。 实质上是对buffer缓冲区的管理： 1，首先要初始化
2，可以通过binder_io使用缓冲区

```
uint32_t svcmgr_lookup(){

    uint32_t handle;
    unsigned iodata[512/4];
    struct binder_io msg, reply;
    
    bio_init(&msg,iodata,sizeof(iodata),4); //初始化binder_io，指向缓冲区！！
    bio_put_uint32(&msg,0);
    bio_put_string16_x(&msg,SVC_MGR_NAME);
    bio_put_string16_x(&msg,name);
    
}



```

//binder_io构造完毕，因此binder_call内部：

```
   binder_call(struct binder_state *bs,struct binder_io *msg,struct binder_io *reply,uint_t target,
uint_t code){
   1 把binder_io、target、code三者转成=》 txn! 
   2 struct {
    uint_t 32;
    struct binder_transaction_data txn;
   
   } writebuf;
   
   3 如：
   writebuf.txn.target.handle=target;
   writebuf.txn.target.code=code;
   writebuf.txn.data_size = msg->data - msg->data0;
   writebuf.txn.data.ptr.buffer=(uintptr_t)msg->data0; 其实地址
   writebuf.txn.data.ptr.offsets=(uintptr_t)msg->offs0;
   ...
   4, 转换成bwr， 驱动只认识bwr
  bwr.write_size = sizeof(writebuf);
  bwr.write_consumed = 0;
  bwr.write_buffer = (uintptr_t) &writebuf;
   
}
```

# 如何写应用程序？

client： a,binder_open b,获取服务进程handle c，构造binder_io d,调用binder_call e，binder_call会返回server的给的
binder_io，解析返回值

server： a，binder_open驱动 b，注册服务 c，循环的ioctl，从驱动读取 client发送过来的数据 d，得到bwr结构体，解析里面的writebuf
e，在转回去为binder_io f,从binder_io中取出code，调用具体函数 g，把返回值封装成binder_io 发送给客户端。

## 开始代码

test_server.c： a，提供hello服务 b，函数： say_hello say_hello_to

test_client.c 1 open驱动 2， 获取服务进程对应的handler。一个进程可以有多个服务，每个服务都有一个handler。通过target中的ptr来区分
是哪一个服务的handler。 3，构造binder_io对象，通过binder_call()调用对应的code函数

数据结构的流程： client：构造binder_io--> writebuffer对象，通过里面的txn字段，--->bwr对象，调用binder_call。
返回一个bwr，解析成biner_io，得到里面的返回值。

server： 构造binder_io---writebuffer--binder_call调用注册服务；
从驱动读取数据bwr，解析出来readbuf，--得到binder_io，得到里面的字段code和target，调用具体函数，返回值放入到reply中。

## 驱动层 ,内核态

其实所谓的内核必须要从某个进程的用户空间进入，如线程A进入内核空间后，代码就不能自由写了，只能调用某些系统提供的函数，
如open、fork、mmap、iotctl等。此时的执行环境还是在线程A中的,只是进入了系统层的线程A的内核态，执行了某些系统函数。 这些函数的逻辑是写好了的，或者说会调到驱动程序中(
驱动可以由程序员编写，算是对系统函数的扩展)，然后中断返回到线程A的用户空间 ，至此，内核调用完毕。
一段系统函数，进程A可以调用，进程B当然也可以调用，因此，系统会管理每个进程对应的数据。这就是操作系统。

B 调用 A的服务：

handler是进程A对进程B提供的服务service的引用,是个整数。 表示的是服务 而非进程A。
"服务"： 向实现该""服务"的 进程 发送数据。因此，handle是""服务"的引用。 我们打开某个服务，就是对某个服务的引用。在不同的进程里面，handle的值有可能不一样。 什么意思？？

```
     serviceManager         test_server        test_client
引用：   1                   hello服务             2
        2                   googbye服务           1
说明：

```

hello服务注册的时候，desc/handle是1。googbye服务注册的时候，desc/handle是2。
test_client，获取hello服务的时候desc/handle则是2，获取goodbye服务则是1。

驱动内部用一个binder_ref来描述这个引用，表示对某个服务(binder_node)的引用。

```


binder_ref{
   struct binder_node *node;
   int desc;
}

binder_node{
   struct binder_proc *proc;
}
```

流程： 通过handle找到binder_ref，通过binder_ref找到binder_node，找到进程B. 那么进程B怎么表示？？ binder_proc!!

```
binder_proc{
    struct rb_root threads； 红黑树，用来挂载线程

}
```

找到进程B后，就把数据放到proc描述的进程B中。

现实中会有个client程序向进程B获取服务，那么进程B就会创建多个线程，一个线程用来响应一个client的binder服务请求。
因此，进程B中肯定有结构体用来管理这些线程，这个结构体就是struct rb_root threads。这是一个红黑树，用来存储线程。 每个线程都会有 binder_thread
表示binder线程。

```
struct binder_thread{
}
```

binder节点总结：

```
1，server从用户空间传入一个flat_binder_object给驱动， 在内核态的驱动程序中为每个服务创建binder_node，
里面binder_node.proc= server进程。
2，serviceManager会在内核中为创建binder_ref，引用binder_node。 binder_ref.desc=1,2,3...。
    在用户态创建服务链表，节点包含：name；handle（就是对应内核中的binder_ref.desc）
3，client程序向serviceManager查询服务，传入name。会通过驱动到达serviceManager的用户空间。
4，serviceManager在用户空间的服务链表中找到后，返回handle给内核态的驱动程序，
5，驱动程序在serviceManager的binder_ref红黑树中根据handle找到binder_ref结构
    在根据binder_ref.node找到binder_node-被指向，最后给client创建新的binder_ref，指向这个binder_node，
它的binder_ref.desc从1开始。desc表示对应的进程。驱动返回desc给client，他即位handle
6，client程序在发数据给handle：
    驱动根据handle找到binder_ref
    根据binder_ref找到binder_node
    根据binder_node找到server进程
```

# 重点

数据读写的传输(进程切换过程)：

数据如何复制？ 一般方法：两次复制。

```
        -用户态：构造数据
client：
        -内核态：驱动copy_from_user
        
|  唤醒server进程的内核态      
v       
        -用户态：处理数据
server：
        -内核态：驱动copy_to_user
```

binder的方法：只需一次拷贝。 利用mmap技术，也就是用户态可以直接操作内核态的内存。

1，server mmap，用户态可以直接访问内核态驱动中某块物理内存。 2，client 构造数据，调用驱动发送数据：copy_from_user， 3，server
可以在用户态直接使用内核中的数据。 这会涉及到一个结构体，binder_buffer。后面在说。

注意： 数据复制一次说的是数据本身，而数据头还是要复制两次的。 client： binder_write_read
bwr，bwr中的binder_buffer指向了某块数据，通过内核的ioctl发送给server。 内核：copy_from_user，拷贝bwr到内核。
copy_from_user拷贝数据到内核内存。 唤醒server的内核态， copy_from_user拷贝bwr到用户空间，用户空间直接访问内核内存中的数据。

可以这么认为：serviceManager是针对每个服务的，而不是进程的。它管理的是服务。 疑问：
serviceManager在用户态维护了一个服务链表，在内核中维护了一个binder_ref的红黑树？
链表是用来根据name找handle，内核中根据handle找binder_ref，根据binder_ref找到binder_node， 根据binder_node找到
binder_proc，进而找到server进程。

# 骚操作

学到一招骚操作：

```
if(code==001){
    call_001();
}else if(code==002){
    call_002();
}
优化： 
注册的时候直接使用一个函数的指针，而非code来区分。
那么在调用的时候就是这个样子： 
定义一个函数指针： 
*func.call()
```

# 情景分析

binder驱动内核是多线的。

binder_ioctl(); BC_XXX命令code： 表示用户空间调用内核的码 BR_XXXcode： 表示内核返回给用户空间的码

# 应用程序如何访问驱动

APP: open、read、write、ioctl

驱动： briver_open，drv_read, ...

驱动程序怎么写？？ 1，构造一个file_oprations ..open=drv_open ..write=drv_writer ...

2,告诉内核： 注册驱动。调用register_chrdev,放入file_oprations结构体来注册驱动。

3, 驱动程序的入口函数，调用register_chardev()。驱动被安装的时候，系统自动调用register_chardev。

binder.c的入口是device_initcall：

```
binder_init(){

    //杂项设备的注册 ，入口函数
    ret=misc_register(struct miscdevice * misc)

}


```

misc也是一个驱动程序，但是是一个虚拟的驱动程序。

```
misc_init(){  //misc入口函数

    

}
```

# 总结：

binder驱动框架： 1，misc设备驱动：确实有： a file_operations:
b 调用register_chadev(会创建一些class，用来让系统在虚拟文件中注册一些设备节点信息)
c 有入口函数

2，binder驱动： a，misc_device ..fps=fileopration ..open=binder_open. b，通过misc_register 注册misc_device。
所谓注册本质上把 misc_device放入某个链表。 c,入口

APP： open--misc_open==> 根据次设备好从链表中找到misc_device，取出它的fileoprations，调用binder_open函数。

## 服务的注册的过程：

serviceManager binder_thread_writer， BC_enter_loop serviceManager binder_thread_read， BR_NOOP？？//
因为对于所有的读操作，数据头部都是BR_NOOP。 serviceManager 休眠...

补充知识：

```
    A                        B

    BC_TRANSACTION--发出-->     BR_TRANSACTION
    BC_REPLY          <--回复---BR_REPLY
```

只有以上四个命令会涉及到进程 A B两个进程。 其他的所有cmd命令BC_XXX,BR_XXX都是app与驱动的交互，用来改变或者修改状态。

具体过程 1，内核函数 binder_open的实现

```
创建binder_proc，表示当前进程。 

```

2，binder_ioctl 会创建binder_thread,里面有todo链表。

```
    binder_proc 表示一个进程
    binder_thread, //表示进程中的线程结构体

```

当app读数据时候，第一次会始终读取br_noop(4个字节)，后面才是数据。

```
buffer：   |BR_NOOP|cmd+数据|cmd+数据|....
```

## test_server：

APP用户空间： 1，构造数据 如：

```

int svcmgr_publish(struct binder_state *bs, uint32_t target, const char *name, void *ptr)
{
    int status;
    unsigned iodata[512/4];
    struct binder_io msg, reply;

    bio_init(&msg, iodata, sizeof(iodata), 4);
    bio_put_uint32(&msg, 0);  // strict mode header
    bio_put_string16_x(&msg, SVC_MGR_NAME);
    bio_put_string16_x(&msg, name);
    bio_put_obj(&msg, ptr);

    if (binder_call(bs, &msg, &reply, target, SVC_MGR_ADD_SERVICE))
        return -1;

    status = bio_get_uint32(&reply);

    binder_done(bs, &msg, &reply);

    return status;
}

```

```
    int status;
    unsigned iodata[512/4]; buffer缓冲区。
    struct binder_io msg, reply;

    //对缓冲区的头进行初始化16个字节offsets 用来指向flat_binder_obj，留出来
    bio_init(&msg, iodata, sizeof(iodata), 4);
    // 在后面的4个字节放入0
    bio_put_uint32(&msg, 0);  // strict mode header
    // 在后面的放入字长(2个字节)，先放入字符串的长度，再放入字符串"android.os.IServiceManager"
    bio_put_string16_x(&msg, SVC_MGR_NAME);
    // 在后面继续放入服务的名字，先放入长度，在放入字符串如 hello、ams
    bio_put_string16_x(&msg, name);
    // 传入flat_binder_obj
    bio_put_obj(&msg, ptr);
```

之前讲过，server会在内核中创建一个binder_node结构体，里面的东西到底是什么？ APP应该会传入一个东西给内核。 转换成binder_transaction_data,
存入bwr结构体传给内核： 2，发送数据调用ioctl 进入内核 APP内核空间： binder_ioctl
把数据放入serviceManager进程的todo链表中，并唤醒serviceManager。 首先找到目的进程 然后在拷贝用户内存的数据到驱动内核的内存空间 唤醒







