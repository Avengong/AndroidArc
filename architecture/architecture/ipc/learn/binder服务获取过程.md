binder_client:

```

用户态：
1 构造数据，binder_io
2 发送数据给serviceManager，

内核态： 
3 根据handle=0，找到serviceManager的proc对象。把数据拷贝到proc的todo链表
4，唤醒serviceManager

```

serviceManager：

```
内核态：
5，返回数据
13，得到handle
用户态：
6，取出数据，得到hello
7，在svclist链表里，根据 hello 找到对应的svcinfo，得知handle=1.（这是server注册时候的handle）
8，用ioctl 把handle发给驱动内核。
9，在refs_by_desc树中，根据handle找到binder_ref，进而找到hello服务的binder_node。在找到binder_proc。得到test_server。
10，为test_client创建binder_ref，里面也有binder_proc
11，唤醒test_client，把数据放入test_client的todo链表，
12，在test_client的binder_proc结构体中的refs_by_desc加入一个binder_ref节点。
```


