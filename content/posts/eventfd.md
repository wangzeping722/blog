---
title: "eventfd"
date: 2021-03-17T10:53:54+08:00
draft: true
---

eventfd 是 linux 在 2.6.27 版本新增的进程间通信方式，主要用于进程或者线程之间的通信。

我们知道，在 I/O 多路复用中，select，poll，epoll 都会阻塞在监听函数上，但是，有时候我们需要在监听事件还未到来之前就将线程唤醒。常用的方法是使用管道进行通信，建立一个管道，将管道的一端置于监听函数上，当我们想要唤醒线程时，向管道的另一端写入数据。

eventfd 通信调用为上述过程提供了更加方便的实现形式 。只能在父子进程中做简单的消息通知，性能上比 pipe 好一些。

``` c
#include<sys/eventfd.h> 

int eventfd(unsigned int initval, int flags); //返回对象的文件描述符
```

eventfd() 创建一个“eventfd对象”，这个对象能被用户空间应用用作一个事件等待/响应机制，靠内核去响应用户空间应用事件。这个对象包含一个由内核保持的无符号64位整型计数器。这个计数器由参数initval说明的值来初始化。
它的标记可以有以下属性：

EFD_CLOECEX，EFD_NONBLOCK，EFD_SEMAPHORE。

它返回了一个引用eventfd object的描述符。这个描述符可以支持以下操作：

**read：** 如果计数值counter的值不为0，读取成功，获得到该值，读取后count值变为0。如果counter的值为0，非阻塞模式，会直接返回失败，并把errno的值指纹EINVAL。如果为阻塞模式，一直会阻塞到counter为非0位置。

**write：** 会增加8字节的整数在计数器counter上，如果counter的值达到0xfffffffffffffffe时，就会阻塞。直到counter的值被read。阻塞和非阻塞情况同上面read一样。