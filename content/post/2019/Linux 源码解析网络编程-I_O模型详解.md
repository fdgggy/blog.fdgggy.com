---
title: "Linux 源码解析网络编程-I/O模型详解"
date: 2019-07-15T12:05:08+08:00
description: "Linux 源码解析网络编程-I/O模型详解"
tags:
 - 网络IO
 - 阻塞
 - 非阻塞
 - IO复用
 - 异步
categories:
 - 源码分析
keywords:
 - 网络IO
 - 阻塞
 - 非阻塞
 - IO复用
 - 异步
url: "/2019/07/15/io_model/"
---
#### I/O模型
##### 阻塞式I/O模型
socket套接字默认是阻塞的，如果I/O条件未满足，则进程或线程就会被挂起，直到I/O条件满足才返回。常用的IO操作都是阻塞I/O，如read一个已连接的套接字时，如果没有数据，那么就会挂起进程，阻塞等待，直到有数据可读时才返回。
![0f58289a87bd790a8185d341a16d034d](/img/hugo/2019/Linux 源码解析网络编程-I_O模型详解.resources/23B1C7A4-62C8-4CBC-B38D-A71DA59FFA4C.png)
如果采用阻塞式I/O做高并发服务器，假设有1000个连接，就需要1000个线程或进程来处理1000个连接的I/O操作，缺点：
1.线程是有内存开销的，1个线程大概需要512k存放栈，那么1000个线程需要大概512M内存空间。
2.线程切换(上下文切换)是有cpu开销的，需花费大量时间在上下文切换上。
#### 进程阻塞不占用cpu资源
操作系统实现了进程调度的功能，会把进程分为运行和等待等几种状态。运行状态是进程获得cpu使用权，正在执行代码的状态。等待状态是阻塞状态。如程序运行到read/recv时，程序会从运行状态变为等待状态，接收到数据后又变回运行状态。
![e260ec41f776a8435fc5fd0a5d58ef6e](/img/hugo/2019/Linux 源码解析网络编程-I_O模型详解.resources/52C3374F-CE2C-4D81-AAF6-1D0587C0A215.png)
上图中的3个进程都被操作系统的工作队列所引用，处于运行状态，会分时执行。
![e4c4e6a534d8c3db21655d0b7ba209a4](/img/hugo/2019/Linux 源码解析网络编程-I_O模型详解.resources/57493DE9-430F-43FE-A553-70F62C0E0122.png)
当程序执行recv时，操作系统会将进程A从队列移动到改sockt的等待队列中。工作队列只剩下B和C，cpu会轮流执行这两个进程的程序，不会执行进程A的程序。所以进程A被阻塞，不会往下执行代码，也不会占用cpu资源。
![c9b6e5e167622006d85b6ffe53e03e21](/img/hugo/2019/Linux 源码解析网络编程-I_O模型详解.resources/77DFBCE4-8239-4591-8E9E-555D9BB186D3.png)
![1815eae29b1bcab2482df54a37e5ab16](/img/hugo/2019/Linux 源码解析网络编程-I_O模型详解.resources/5FCA10B2-3AFE-4B87-9382-0783C7A1E008.png)
#### 非阻塞I/O模型
如果把套接字设置成非阻塞，可以通过fcntl(posix)或ioctl(unix)设置为非阻塞模式。当调用recv时，如果有数据，就返回数据，没有数据则返回错误。这样的操作不阻塞线程，应用层需要不断的轮询读取。
![40b0c38fa94e35b3e86850ba8cd3518e](/img/hugo/2019/Linux 源码解析网络编程-I_O模型详解.resources/C47C46EA-E46A-48F0-9EC2-A96E30E962A2.png)
缺点：应用进程需要持续轮询内核，查看是否有数据，耗费大量cpu。
设置为非阻塞套接字后：
1.使用read,recv,write,send...如果无数据，立即返回错误
2.accept，如果无新连接，立即返回错误
3.connect,如果连接不能立即建立，连接能发起，有可能连接成功，有可能失败。但会返回错误。
#### I/O复用模型 (I/O multiplexing)，也称为事件驱动模型
一个进程或线程同时监视多个文件描述符(socket)的就绪状态，一旦有一个或多个文件描述符读状态就绪，则返回给应用层，应用层可以通过一个线程或多个线程读写操作，读写过程是阻塞的。
select,poll,epool都是属于I/O复用模型。
![87ee809edb1ec0a0c336dcd03f64f163](/img/hugo/2019/Linux 源码解析网络编程-I_O模型详解.resources/5BF7179E-79F2-4B0D-8E27-01B1DD781AAB.png)
这样处理1000个连接时，只需要1个线程监控就绪状态，对就绪的每个连接开一个线程处理数据，大大减少了线程数量，减少了内存开销和上下文切换的cpu开销。
#### 信号驱动式I/O模型
被监听的描述符就绪后，内核发送SIGIO信号通知应用层，应用层在阻塞读取数据
![bad68caa9f06cfa3e96ea7f371b57dca](/img/hugo/2019/Linux 源码解析网络编程-I_O模型详解.resources/ECD66808-3E7B-4DDE-809C-8D3D312FD064.png)
优点：等待数据到达期间进程不会被阻塞。
#### 异步I/O模型
内核通知I/O操作何时完成
![6cb2b17e1a5d34ac07d2970724e70b3c](/img/hugo/2019/Linux 源码解析网络编程-I_O模型详解.resources/0902F70D-FCDF-408E-825F-499E24174332.png)
通过系统调用将文件描述符，缓冲区指针，大小告知内核，当整个操作完成时通知应用层。
异步IO不需要应用负责读写，内核会把数据从内核空间拷贝到用户空间。

#### 各种I/O模型比较
同步I/O：导致请求进程阻塞，直到I/O操作完成。包括：阻塞式I/O,非阻塞式I/O,I/O复用，信号驱动I/O
异步I/O：不会导致请求进程阻塞。包括：异步I/O模型
![101c5c7ce2d88b6c078d082ecd0ecb7d](/img/hugo/2019/Linux 源码解析网络编程-I_O模型详解.resources/D52C0E5F-52E5-40AA-83F2-164007C5F24F.png)


# 参考

1.UNIX网络编程卷1:套接字联网API.     
2.https://zhuanlan.zhihu.com/p/64746509.    






