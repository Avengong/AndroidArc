# 服务的使用过程

每个进程在驱动内核中都有一个binder_proc对象。里面有两棵红黑树：refs_by_desc，refs-by_node。用来保存目的进程服务的 binder_ref，
binder_ref中有binder_node服务，binder_node中有服务进程的binder_proc。至此，找到server进程。

```
client：
用户态： 构造数据，封装handle+bwr
--------------------
内核态：
根据handle在client对应的binder_proc中找到binder_ref，....最终找到server的 binder_proc,
copy bwr，copy数据
把数据放入到server的todo链表
唤醒server


server：
用户态：
去除数据，根据code调用函数
用返回值构造数据，发送给client，到内核态。

--------------------------------------
内核态： 被唤醒，读出数据，返回数据给用户态。

        找出要返回的进程，client
        把数据放入client的todo链表
        唤醒client进程的内核。 

```

# transaction_stack 机制

## 发给谁？？

问题： server对应一个binder_proc,同时还对应多个线程binder_thread。binder_proc和binder_thread中都有todo链表。
handle值表明是server进程， 那么client到底往哪个todo链表加入数据呢？？

```
1，一般放入binder_proc的todo链表中，唤醒等待的空闲的binder_thread线程。
2，对于双向传输，则放在binder_thread的todo里面。唤醒该线程。用transactionstack来判断
```

## 回给谁？

没handle来表明目的进程，肯定在某个地方记录发送者。 这个地方就是transactionstack。

```
    A                        B

    BC_TRANSACTION--发出-->     BR_TRANSACTION
    BC_REPLY          <--回复---BR_REPLY
    
    
    
1，BC_TRANSACTION： 
a，一开始不是双向传输，所以放在binder_proc进程的todo链表中

b，入栈。 client的 binder_thread.transaction_stack。
这个transaction_stack里面包含了from、to proc、to thread。
唤醒进程的线程。

2，BR_TRANSACTION：
从server的binder_proc中取出数据,
处理
入栈，sever的 binder_thread中有个transaction——stack, 指向了client栈中同一个 binder_stack。

同一个binder_transaction, 通过 from_parent放入发送者中栈；通过to_prarent放入接受者的栈。

3， BR_REPLY
回复给谁？ 
a，从栈中取出一个binder_transaction结构体。 回复给其中的client。

b，出栈 


。。。

  
```

还记得进程在调用binder_open的时候，驱动会为它创建一个binder_proc。
在调用ioctl的时候会创建一个线程，因此，就算client是单线程的，也会有单独线程binder_thread结构体，里面有transaction_stack.

# transaction_stack 机制 双向服务

假设：

```
P1进程提供 s1服务。
P2进程提供 s2服务。
P3进程提供 s3服务。
```

























