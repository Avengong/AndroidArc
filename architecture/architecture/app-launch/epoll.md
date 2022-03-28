# epoll 机制

## I / O 多路复用

## eventPoll 结构

内部构成： 红黑树+双向链表。

## 三个核心API

1，epoll_create(int size)
当调用这个接口后，系统内核中会创建eventPoll对象，这个对象本身相当于一个fd，并返回这个对象的句柄fd值。

```

```

2，epoll_clt(int epfd,int op,int fd,struct epoll_event *event)
增加、修改、删除某个fd的监听。 event: 指针地址。 当该事件变化时候会回调。

3,epoll_wait(int epfd,struct epoll_event *events,int maxevnets,int timeout)
等待监听。 eventPoll中链表数据。 只要epoll被触发，那么就会通过events数组地址返回。

触发机制： 














