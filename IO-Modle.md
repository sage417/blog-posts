---
title: IO Modle
date: 2019-05-20 15:39:56
thumbnail: /images/BingWallpaper-2019-05-20.jpg
tags:
    - java
    - linux
categories:
    - Java基础
---
### 操作系统IO模型与Java IO

Java IO模型和操作系统IO模型息息相关，之前阻塞/非阻塞，同步/非同步之间的关系一直分不清，所以很有必要了解下操作系统(linux)提供了哪些接口来进行IO。目前我们只需要了解即可，使用相关可以直接查看java io教程。



#### 最基础的知识

以使用IO读取数据为例，一般操作系统分为两个独立的阶段进行操作：

1. 等待数据准备完成，可以是从磁盘拷贝到内核空间，或者是网卡接受到数据后拷贝到内核空间。
2. 从内核空间将数据拷贝至请求数据的进程。如果是java可能还需从进程拷贝至jvm堆内存。



#### Blocking I/O Model

这个是最常用的模型，望文生义就是阻塞IO，进行IO的两个阶段会都阻塞程序，直到读取到数据或者返回错误才会返回。

![blocking io](https://i.loli.net/2019/05/20/5ce23b22371a951913.png)

具体来说，通过调用系统recvfrom函数，而recvfrom函数会等到出错或者把数据拷贝到进程完成时才会返回。之后我们程序只需要处理错误或者处理数据就可以了。

阻塞模型对应java中绝大部分IO操作，比如网络请求api，io stream api，该模型优点在于简单直观，缺点在长时间阻塞很难支持大量并发IO请求。



#### Nonblocking I/O Model

该模型在java中没有对应，所以这里只做简单介绍。

![nonblocking io](https://i.loli.net/2019/05/20/5ce23ea09356737999.png)

使用轮询方式调用系统recvfrom函数，recvfrom函数在第一阶段完成前一直返回错误，直到第一阶段完成后，阻塞至第二阶段完成。

这个模型稍显鸡肋，特点是在第一阶段是非阻塞的（进程不会被切换），代码相比阻塞模型来说也更复杂。



#### I/O Multiplexing Model

非常著名的IO模型，可以支持大量并发IO。通过调用`select`或者`pull`并阻塞，而不是在实际调用系统IO时阻塞。使用select阻塞在第一阶段和Blocking I/O的阻塞不太一样，Blocking I/O阻塞在当前IO操作第一阶段，而I/O复用则可以注册多个I/O在select函数，当有一个I/O就绪时select函数就会返回，如果所有I/O处于第一阶段阻塞状态则select函数阻塞。

#### ![multiplexing io](https://i.loli.net/2019/05/20/5ce2479f396b673848.png)

相比较Blocking I/O Model和Nonblocking I/O Model，I/O Multiplexing Model明显能在短时间内处理更多的I/O。如果使用多线程+Blocking I/O Model也能达到类似的效果，但是有可能消耗过多线程资源。

I/O Multiplexing Model对应java NIO的Selector等api



#### Signal-Driven I/O Model

该模型在java中没有对应，所以这里只做简单介绍。

![Signal-Driven I/O](https://i.loli.net/2019/05/20/5ce24b4cbe2bc95227.png)

该模型特点是第一阶段调用sigaction函数非阻塞返回，在第一阶段完成后发送信号`SIGIO`至进程，之后在`signal handler`中进行第二阶段处理。相当于对Nonblocking I/O Model的一种改进。



#### Asynchronous I/O Model

Asynchronous I/O Model相比较Signal-Driven I/O Model的区别在于通知的时机不同：Asynchronous I/O Model在第一和第二阶段都完成时通过信号通知进程操作完成。

![Asynchronous I/O Model](https://i.loli.net/2019/05/20/5ce2562b51ccb91290.png)

Asynchronous I/O Model对应java中**AsynchronousSocketChannel**，**AsynchronousServerSocketChannel** 和 **AsynchronousFileChannel**等api。



#### 各个模型比较

![](https://i.loli.net/2019/05/20/5ce2570fe047836287.png)

[1]: https://notes.shichao.io/unp/ch6/	"Chapter 6. I/O Multiplexing: The select and poll Functions"


