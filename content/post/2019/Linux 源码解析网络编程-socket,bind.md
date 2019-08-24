---
title: "Linux 源码解析网络编程--socket,bind"
date: 2019-07-16T12:05:08+08:00
description: "Linux 源码解析网络编程-socket,bind"
tags:
 - Linux 网络编程
 - socket 创建
 - socket 绑定
 - socket 原生API
categories:
 - 源码分析
keywords:
 - Linux 网络编程
 - socket 创建
 - socket 绑定
 - socket 原生API
url: "/2019/07/16/socket/"
---

#### scoket()
作用：创建一个通信端点并返回一个引用该端点的文件描述符
```c
int socket(int domain, int type, int protocol);
```
参数：
@domain(协议域),用于通信的协议簇,如:
![1ca822e20f6ed9322139e55937bd3a5b](/img/hugo/2019/Linux 源码解析网络编程-socket,bind.resources/309F66CB-D4A8-44B8-9253-7340EDFEE1AE.png)

@type 套接字类型，制定通信的语义,如:
![305846c3f76d8e62726f7b4e97741d50](/img/hugo/2019/Linux 源码解析网络编程-socket,bind.resources/0764153E-2BEC-4506-A7B5-73B229C3792C.png)
SOCK_STREAM 有序，可靠，双向，基于连接的字节流   TCP
SOCK_DGRAM 数据报,无连接，不可靠，具有固定最大长度  UDP
@protocol 协议类型常值,一般设为0，系统设定domain和type组合的默认值
![6947ee8ed215f9ca563e9a756d90561a](/img/hugo/2019/Linux 源码解析网络编程-socket,bind.resources/891E5C8B-095C-4D09-986C-2F93C6227A42.png)
返回值：
成功时返回一个非负整数，与文件描述符类似，称为套接字描述符。
失败时返回-1。
说明：
此函数指定了协议族和套接字类型，并没有指定本地协议地址或远程协议地址,创建好后的套接字保存在net名字空间。


linux 内核源码
socket层总入口
```c
/*
 *	System call vectors.
 *
 *	Argument checking cleaned up. Saved 20% in size.
 *  This function doesn't need to set the kernel lock because
 *  it is set by the callees.
 */

SYSCALL_DEFINE2(socketcall, int, call, unsigned long __user *, args)
{
	switch (call) {
	case SYS_SOCKET:
		err = sys_socket(a0, a1, a[2]);
		break;
	case SYS_BIND:
		err = sys_bind(a0, (struct sockaddr __user *)a1, a[2]);
		break;
	case SYS_CONNECT:
		err = sys_connect(a0, (struct sockaddr __user *)a1, a[2]);
		break;
	case SYS_LISTEN:
		err = sys_listen(a0, a1);
		break;
	case SYS_ACCEPT:
		err = sys_accept4(a0, (struct sockaddr __user *)a1,
				  (int __user *)a[2], 0);
		break;
  }
  return err;
}
```
```c
SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
{
	int retval;
	struct socket *sock;
	int flags;

	flags = type & ~SOCK_TYPE_MASK;
	if (flags & ~(SOCK_CLOEXEC | SOCK_NONBLOCK))
		return -EINVAL;
	type &= SOCK_TYPE_MASK;

	if (SOCK_NONBLOCK != O_NONBLOCK && (flags & SOCK_NONBLOCK))
		flags = (flags & ~SOCK_NONBLOCK) | O_NONBLOCK;

    //socket创建
	retval = sock_create(family, type, protocol, &sock);
	if (retval < 0)
		goto out;
    //socket映射文件描述符
	retval = sock_map_fd(sock, flags & (O_CLOEXEC | O_NONBLOCK));
	if (retval < 0)
		goto out_release;

out:
	/* It may be already another descriptor 8) Not kernel problem. */
	return retval;

out_release:
	sock_release(sock);
	return retval;
}

/**
 *	__sock_create - creates a socket
 *	@net: net namespace   
 *	@family: protocol family (AF_INET, ...)
 *	@type: communication type (SOCK_STREAM, ...)
 *	@protocol: protocol (0, ...)
 *	@res: new socket
 *	@kern: boolean for kernel space sockets
 *
 *	Creates a new socket and assigns it to @res, passing through LSM.
 *	Returns 0 or an error. On failure @res is set to %NULL. @kern must
 *	be set to true if the socket resides in kernel space.
 *	This function internally uses GFP_KERNEL.
 */
int __sock_create(struct net *net, int family, int type, int protocol,
			 struct socket **res, int kern)
{
}

int sock_map_fd(struct socket *sock, int flags)
{
	struct file *newfile;
	int fd = sock_alloc_file(sock, &newfile, flags);

	if (likely(fd >= 0))
		fd_install(fd, newfile);

	return fd;
}

```
#### bind
作用：为套接字设置一个本地协议地址。套接字创建后保存在net名字空间但并没有地址分配给他，bind通过addr参数分配地址给sockfd所引用的套接字
```c
int bind(int sockfd, const struct sockaddr *addr,socklen_t addrlen); 
```
@sockaddr 指向特定协议的地址结构指针
@addrlen 改地址结构长度
```c
    struct sockaddr_in servaddr; //地址格式描述
    memset(&servaddr, 0, sizeof(struct sockaddr_in));
    servaddr.sin_family = AF_INET;  //IPV4
    servaddr.sin_port = htons(123); //端口号
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);//INADDR_ANY一般为0，内核选择IP地址

    if (bind(listenfd, (struct sockaddr *)&servaddr, sizeof(struct sockaddr_in)) == -1) {
        printf("bind socket addr failed!\n");
        return 0;
    }
```
linux 内核源码
```c
/*
 *	Bind a name to a socket. Nothing much to do here since it's
 *	the protocol's responsibility to handle the local address.
 *
 *	We move the socket address to kernel space before we call
 *	the protocol layer (having also checked the address is ok).
 */

SYSCALL_DEFINE3(bind, int, fd, struct sockaddr __user *, umyaddr, int, addrlen)
{
	struct socket *sock;
	struct sockaddr_storage address;
	int err, fput_needed;
    //通过文件描述符fd查找对应的socket
	sock = sockfd_lookup_light(fd, &err, &fput_needed);
	if (sock) {
       //把用户空间的地址复制到内核空间
		err = move_addr_to_kernel(umyaddr, addrlen, &address);
		if (err >= 0) {
			err = security_socket_bind(sock,(struct sockaddr *)&address,addrlen);
			if (!err)
				err = sock->ops->bind(sock,(struct sockaddr *)&address, addrlen);
		}
		fput_light(sock->file, fput_needed);
	}
	return err;
}

/**
 *	move_addr_to_kernel	-	copy a socket address into kernel space
 *	@uaddr: Address in user space
 *	@kaddr: Address in kernel space
 *	@ulen: Length in user space
 *
 *	The address is copied into kernel space. If the provided address is
 *	too long an error code of -EINVAL is returned. If the copy gives
 *	invalid addresses -EFAULT is returned. On a success 0 is returned.
 */

int move_addr_to_kernel(void __user *uaddr, int ulen, struct sockaddr_storage *kaddr)
{
	if (ulen < 0 || ulen > sizeof(struct sockaddr_storage))
		return -EINVAL;
	if (ulen == 0)
		return 0;
	if (copy_from_user(kaddr, uaddr, ulen))
		return -EFAULT;
	return audit_sockaddr(ulen, kaddr);
}

```




