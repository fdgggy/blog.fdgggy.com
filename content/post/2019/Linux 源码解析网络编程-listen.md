---
title: "Linux 源码解析网络编程-listen"
date: 2019-07-16T12:05:08+08:00
description: "Linux 源码解析网络编程-listen"
tags:
 - Linux 网络编程
 - socket listen
 - socket 监听
 - socket 原生API
categories:
 - 源码分析
keywords:
 - Linux 网络编程
 - socket listen
 - socket 监听
 - socket 原生API
url: "/2019/07/16/listen/"
---

#### listen()
```c
int listen(int sockfd, int backlog);
```
作用：监听套接字上的链接
@sockfd 引用SOCK_STREAM/SOCK_SEQPACKET类型套接字的文件描述符
@backlog 挂起连接的队列的最大长度, ***ddos攻击有关

linux 内核源码
```c
/*
 *	Perform a listen. Basically, we allow the protocol to do anything
 *	necessary for a listen, and if that works, we mark the socket as
 *	ready for listening.
 */

SYSCALL_DEFINE2(listen, int, fd, int, backlog)
{
	struct socket *sock;
	int err, fput_needed;
	int somaxconn;
    //通过文件描述符fd查找对应的socket
	sock = sockfd_lookup_light(fd, &err, &fput_needed);
	if (sock) {
       //backlog参数不能超过系统参数somaxconn
		somaxconn = sock_net(sock->sk)->core.sysctl_somaxconn;
		if ((unsigned int)backlog > somaxconn)
			backlog = somaxconn;

		err = security_socket_listen(sock, backlog);
		if (!err)
           //socket层的操作，如果是SOCK_STREAM,接下来是inet_listen()
			err = sock->ops->listen(sock, backlog);

		fput_light(sock->file, fput_needed);
	}
	return err;
}
```
```c
struct sock {
	unsigned short sk_ack_backlog;   current listen backlog
	unsigned short sk_max_ack_backlog;   listen backlog set in listen()
}
//SOCK_STREAM 套接字的socket层操作函数实例为inet_stream_ops, 位于/net/ipv4/af_inet.c
const struct proto_ops inet_stream_ops = {
    ...
	.family		   = PF_INET,
	.bind		   = inet_bind,
	.accept		   = inet_accept,
	.listen		   = inet_listen,
    ...
    }
    
```
```c
/*
 *	Move a socket into listening state.
 */
 //net/ipv4/af_inet.c
int inet_listen(struct socket *sock, int backlog)
{
	struct sock *sk = sock->sk;
	unsigned char old_state;
	int err;

	lock_sock(sk);

	err = -EINVAL;
    //套接字状态需要为SS_UNCONNECTED,类型需要为SOCK_STREAM
	if (sock->state != SS_UNCONNECTED || sock->type != SOCK_STREAM)
		goto out;

	old_state = sk->sk_state;
	if (!((1 << old_state) & (TCPF_CLOSE | TCPF_LISTEN)))
		goto out;

	/* Really, if the socket is already in listen state
	 * we can only allow the backlog to be adjusted.
	 */
	if (old_state != TCP_LISTEN) {
		err = inet_csk_listen_start(sk, backlog);
		if (err)
			goto out;
	}
	sk->sk_max_ack_backlog = backlog;
	err = 0;

out:
	release_sock(sk);
	return err;
}
//net/ipv4/inet_connection_sock.c
int inet_csk_listen_start(struct sock *sk, const int nr_table_entries)
{
	struct inet_sock *inet = inet_sk(sk);
	struct inet_connection_sock *icsk = inet_csk(sk);
    //创建request_sock_queue请求socket队列管理实例，初始化全连接队列，创建半连接管理实例
	int rc = reqsk_queue_alloc(&icsk->icsk_accept_queue, nr_table_entries);

	if (rc != 0)
		return rc;

	sk->sk_max_ack_backlog = 0;
	sk->sk_ack_backlog = 0;
	inet_csk_delack_init(sk);

	/* There is race window here: we announce ourselves listening,
	 * but this transition is still not validated by get_port().
	 * It is OK, because this socket enters to hash table only
	 * after validation is complete.
	 */
     //设置socket状态为TCP_LISTEN
	sk->sk_state = TCP_LISTEN;
    //检测端口是否可用，防止bind后其他进程修改端口信息
	if (!sk->sk_prot->get_port(sk, inet->inet_num)) {
		inet->inet_sport = htons(inet->inet_num);

		sk_dst_reset(sk);
		sk->sk_prot->hash(sk); //把sock连接放入监听哈希表

		return 0;
	}

	sk->sk_state = TCP_CLOSE;
	__reqsk_queue_destroy(&icsk->icsk_accept_queue);
	return -EADDRINUSE;
}

/*
 * Maximum number of SYN_RECV sockets in queue per LISTEN socket.
 * One SYN_RECV socket costs about 80bytes on a 32bit machine.
 * It would be better to replace it with a global counter for all sockets
 * but then some measure against one socket starving all other sockets
 * would be needed.
 *
 * The minimum value of it is 128. Experiments with real servers show that
 * it is absolutely not enough even at 100conn/sec. 256 cures most
 * of problems.
 * This value is adjusted to 128 for low memory machines,
 * and it will increase in proportion to the memory of machine.
 * Note : Dont forget somaxconn that may limit backlog too.
 */
int sysctl_max_syn_backlog = 256;
EXPORT_SYMBOL(sysctl_max_syn_backlog);
//net/core/request_sock.c
//@queue 连接请求控制
//@nr_table_entries 半连接最大个数
int reqsk_queue_alloc(struct request_sock_queue *queue,unsigned int nr_table_entries)
{
	size_t lopt_size = sizeof(struct listen_sock);
	struct listen_sock *lopt;

    //nr_table_entries 必须在[8,sysctl_max_syn_backlog]之间，默认[8,256]
    //但是，在listen入口时确定了backlog<=sysctl_somaxconn(默认128)
    //所以此时默认区间是[8,128]
	nr_table_entries = min_t(u32, nr_table_entries, sysctl_max_syn_backlog);
	nr_table_entries = max_t(u32, nr_table_entries, 8);
    //使nr_table_entries=2^n,向上取整
	nr_table_entries = roundup_pow_of_two(nr_table_entries + 1);
	lopt_size += nr_table_entries * sizeof(struct request_sock *);
	if (lopt_size > PAGE_SIZE)
		lopt = vzalloc(lopt_size);
	else
		lopt = kzalloc(lopt_size, GFP_KERNEL);
	if (lopt == NULL)
		return -ENOMEM;

	for (lopt->max_qlen_log = 3;
	     (1 << lopt->max_qlen_log) < nr_table_entries;
	     lopt->max_qlen_log++);

	get_random_bytes(&lopt->hash_rnd, sizeof(lopt->hash_rnd));
	rwlock_init(&queue->syn_wait_lock);
	queue->rskq_accept_head = NULL;         //全连接队列置空
	lopt->nr_table_entries = nr_table_entries; //半连接队列最大长度

	write_lock_bh(&queue->syn_wait_lock);
	queue->listen_opt = lopt;
	write_unlock_bh(&queue->syn_wait_lock);

	return 0;
}
//用于保存SYN_RECV状态的连接，所以也叫半连接队列
struct listen_sock {
	u8			max_qlen_log;
	u8			synflood_warned;
	/* 2 bytes hole, try to use */
	int			qlen;
	int			qlen_young;
	int			clock_hand;
	u32			hash_rnd;
	u32			nr_table_entries;  //监听client返回syn队列最大长度
	struct request_sock	*syn_table[0];  //半连接队列
};
/** struct request_sock_queue - queue of request_socks
 *
 * @rskq_accept_head - FIFO head of established children
 * @rskq_accept_tail - FIFO tail of established children
 * @rskq_defer_accept - User waits for some data after accept()
 * @syn_wait_lock - serializer
 *
 * %syn_wait_lock is necessary only to avoid proc interface having to grab the main
 * lock sock while browsing the listening hash (otherwise it's deadlock prone).
 *
 * This lock is acquired in read mode only from listening_get_next() seq_file
 * op and it's acquired in write mode _only_ from code that is actively
 * changing rskq_accept_head. All readers that are holding the master sock lock
 * don't need to grab this lock in read mode too as rskq_accept_head. writes
 * are always protected from the main sock lock.
 */
 //sock队列描述
struct request_sock_queue { 
	struct request_sock	*rskq_accept_head; //全连接头
	struct request_sock	*rskq_accept_tail; //全连接尾
	rwlock_t		syn_wait_lock;
	u8			rskq_defer_accept;
	/* 3 bytes hole, try to pack */
	struct listen_sock	*listen_opt;
};
```

#### 内核维护监听套接字的两个队列
![3b4a7f56ba46eada042e00dee2530c8a](/img/hugo/2019/Linux 源码解析网络编程-listen.resources/56841679-0B64-43A2-8D01-42B4AB9A2F89.png)

* 1.半连接队列，装载客户端发送SYN同步分节到达服务器，服务器正在等待TCP3路握手完成的sock数据，sock的状态为SYN_RCV状态
* 2.全连接队列，装载完成TCP3路握手的sock数据，sock状态为ESTABLISHED，该sock数据从半连接队列复制到全连接队列
* 3.accept阻塞等待3路握手完成或63s超时返回，建立连接后从全连接队列取出头结点，返回新的sockfd供应用层读写。
* 4.backlog参数linux3.6源码取值范围是[8,128]，允许修改宏设置最大值，TODO 待验证。如果队列满了，再链接的客户端会收到connect refused的错误码，导致客户端重连。
* 5.SYN Flood攻击，DDOS攻击，就是客户端发送大量SYN，伪造源地址，或者直接下线，服务器会默认等待63s才会断开，这样攻击者就可以把服务器的syn连接队列耗尽，让正常连接请求不能处理。