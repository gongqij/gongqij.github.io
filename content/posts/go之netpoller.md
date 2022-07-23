---
title: "go之netpoller"
date: 2022-07-23T10:53:03+08:00
draft: false
author: "gq"
---

#  I/O 模型

现代的网络服务的主流已经完成从 CPU 密集型到 IO 密集型的转变，所以服务端程序对 I/O 的处理必不可少，而一旦操作 I/O 则必定要在用户态和内核态之间来回切换。

- 阻塞 I/O (Blocking I/O)
- 非阻塞 I/O (Nonblocking I/O)
- I/O 多路复用 (I/O multiplexing)
- 信号驱动 I/O (Signal driven I/O)
- 异步 I/O (Asynchronous I/O)

**操作系统上的 I/O 是用户空间和内核空间的数据交互，因此 I/O 操作通常包含以下两个步骤**：

1. 等待网络数据到达网卡(读就绪)/等待网卡可写(写就绪) –> 读取/写入到内核缓冲区

   > 网卡<-->内核缓冲区

2. 从内核缓冲区复制数据 –> 用户空间(读)/从用户空间复制数据 -> 内核缓冲区(写)

   > 内核缓冲区<-->用户空间

而判定一个 I/O 模型是同步还是异步，主要看第二步：数据在用户和内核空间之间复制的时候是不是会阻塞当前进程，如果会，则是同步 I/O，否则，就是异步 I/O。基于这个原则，这 5 种 I/O 模型中只有一种异步 I/O 模型：Asynchronous I/O，其余都是同步 I/O 模型。

由于使用 epoll 的 I/O 多路复用需要用户进程自己负责 I/O 读写，从用户进程的角度看，读写过程是阻塞的，所以 select&poll&epoll 本质上都是同步 I/O 模型，而像 Windows 的 IOCP 这一类的异步 I/O，只需要在调用 WSARecv 或 WSASend 方法读写数据的时候把**用户空间的内存 buffer** 提交给 kernel，kernel 负责数据在用户空间和内核空间拷贝，完成之后就会通知用户进程，整个过程不需要用户进程参与，所以是真正的异步 I/O。

## 非阻塞 I/O(Nonblocking I/O)

![](https://s2.loli.net/2022/07/23/ipm61UzEA9GMtQe.png)

Non-blocking I/O 的特点是用户进程需要不断的主动询问 kernel 数据好了没有.

```go
//系统调用fcntl，将socket设置成 Non-blocking
func SetNonblock(fd int, nonblocking bool) (err error) {
   flag, err := fcntl(fd, F_GETFL, 0)
   if err != nil {
      return err
   }
   if nonblocking {
      flag |= O_NONBLOCK
   } else {
      flag &^= O_NONBLOCK
   }
   _, err = fcntl(fd, F_SETFL, flag)
   return err
}
```

## I/O 多路复用 (I/O multiplexing)

所谓 I/O 多路复用指的就是 **select/poll/epoll** 这一系列的多路选择器：支持单一线程同时监听多个文件描述符（I/O 事件），阻塞等待，并在其中某个文件描述符**可读写**时收到通知。 I/O 复用其实复用的不是 I/O 连接，而是**复用线程**，让一个 thread of control 能够处理**多个连接（I/O 事件）**。

**fd==文件描述符=I/O事件==连接**

### Select/poll

缺点：

1. 最大并发数限制：使用 32 个整数的 32 位，即 32*32=1024 来标识 fd，虽然可修改，但是有以下第 2, 3 点的瓶颈
2. 每次调用 select，都需要把 fd 集合从用户态拷贝到内核态，这个开销在 fd 很多时会很大
3. 性能衰减严重：每次 kernel 都需要线性扫描整个 fd_set，所以随着监控的描述符 fd 数量增长，其 I/O 性能会线性下降

**返回的活跃连接 == select(全部待监控的连接)**

**全部待监控连接**是数以十万计的，返回的只是数百个**活跃连接**，这本身就是无效率的表现。被放大后就会发现，处理并发上万个连接时，select&poll 就完全力不从心了

### epoll

```c
#include <sys/epoll.h>  
int epoll_create(int size); // int epoll_create1(int flags);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

```go
//epoll_create
static int do_epoll_create(int flags)
{
	error = ep_alloc(&ep); //开辟红黑树等内存空间
	if (error < 0)
		return error;
	return error;
}
```

```c
//epoll_ctl
SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,
		struct epoll_event __user *, event)
{
	epi = ep_find(ep, tf.file, fd);
	error = -EINVAL;
	switch (op) {
	case EPOLL_CTL_ADD: //向红黑树添加
		if (!epi) {
			epds.events |= EPOLLERR | EPOLLHUP;
			error = ep_insert(ep, &epds, tf.file, fd, full_check);
		} else
			error = -EEXIST;
		if (full_check)
			clear_tfile_check_list();
		break;
	case EPOLL_CTL_DEL://删除
		if (epi)
			error = ep_remove(ep, epi);
		else
			error = -ENOENT;
		break;
	case EPOLL_CTL_MOD://修改
		if (epi) {
			if (!(epi->event.events & EPOLLEXCLUSIVE)) {
				epds.events |= EPOLLERR | EPOLLHUP;
				error = ep_modify(ep, epi, &epds);
			}
		} else
			error = -ENOENT;
		break;
	}
}

static int ep_insert(struct eventpoll *ep, const struct epoll_event *event,
		     struct file *tfile, int fd, int full_check)
{
	ep_rbtree_insert(ep, epi);//加入红黑树
}

static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead,
				 poll_table *pt)
{
	if (epi->nwait >= 0 && (pwq = kmem_cache_alloc(pwq_cache, GFP_KERNEL))) {
		init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);//为新加的fd注册回调函数ep_poll_callback，在 fd 相应的事件触发（中断）之后（设备就绪了），内核就会调用ep_poll_callback，将fd 添加到 rdllist 这个双向链表（就绪链表）中
	} else {
		/* We have to signal that an error occurred */
		epi->nwait = -1;
	}
}
```

```go
//epoll_wait
static int do_epoll_wait(int epfd, struct epoll_event __user *events,
			 int maxevents, int timeout)
{
	error = ep_poll(ep, events, maxevents, timeout); //检查 rdllist 双向链表中是否有就绪的 fds
error_fput:
	fdput(f);
	return error;
}

static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,
		   int maxevents, long timeout)
{
	if (timeout > 0) {         //截止timeout获取rdllist
		struct timespec64 end_time = ep_set_mstimeout(timeout);
		slack = select_estimate_accuracy(&end_time);
		to = &expires;
		*to = timespec64_to_ktime(end_time);
	} else if (timeout == 0) { //非阻塞获取rdllist
		timed_out = 1;
		write_lock_irq(&ep->lock);
		eavail = ep_events_available(ep);//判断rdllist是否为空
		write_unlock_irq(&ep->lock);
		goto send_events;
	}
fetch_events://持续获取
  	for (;;) {
		eavail = ep_events_available(ep); //rdllist不为空，退出
		if (eavail)
			break;
		if (!schedule_hrtimeout_range(to, slack, HRTIMER_MODE_ABS)) {//timeout退出
			timed_out = 1;
			break;
		}
	}
send_events:
	/*
	 * Try to transfer events to user space. In case we get 0 events and
	 * there's still timeout left over, we go trying again in search of
	 * more luck.
	 */
	if (!res && eavail &&
	    !(res = ep_send_events(ep, events, maxevents)) && !timed_out)
		goto fetch_events;
	return res;
}
```

![](https://s2.loli.net/2022/07/23/ltkHhgRuYaLEcdO.png)

epoll_create 创建一个 epoll 实例并返回 epollfd；

epoll_ctl 注册 file descriptor 等待的 I/O 事件(比如 EPOLLIN、EPOLLOUT 等) 到 epoll 实例上；

epoll_wait 则是阻塞监听 epoll 实例上所有的 file descriptor 的 I/O 事件，它接收一个用户空间上的一块内存地址 (events 数组)，kernel 会在有 I/O 事件发生的时候把文件描述符列表复制到这块内存地址上，然后 epoll_wait 解除阻塞并返回，最后用户空间上的程序就可以对相应的 fd 进行读写了

在实现上 epoll 采用红黑树来存储所有监听的 fd，而**红黑树**本身插入和删除性能比较稳定，时间复杂度 O(logN)。通过 epoll_ctl 函数添加进来的 fd 都会被放在红黑树的某个节点内，所以，重复添加是没有用的。当把 fd 添加进来的时候时候会完成关键的一步：**该 fd 会与相应的设备（网卡）驱动程序建立回调关系**，也就是在内核中断处理程序为它注册一个回调函数，在 fd 相应的事件触发（中断）之后（设备就绪了），内核就会调用这个回调函数，该回调函数在内核中被称为： `ep_poll_callback` ，**这个回调函数其实就是把这个 fd 添加到 rdllist 这个双向链表（就绪链表）中**。epoll_wait 实际上就是去检查 rdllist 双向链表中是否有就绪的 fd，当 rdllist 为空（无就绪 fd）时挂起当前进程，直到 rdllist 非空时进程才被唤醒并返回。

相比于 select&poll 调用时会将全部监听的 fd **从用户态空间拷贝至内核态空间**并**线性扫描**一遍找出就绪的 fd 再返回到用户态，epoll_wait 则是**直接返回已就绪 fd**，因此 epoll 的 I/O 性能不会像 select&poll 那样随着监听的 fd 数量增加而出现线性衰减，是一个非常高效的 I/O 事件驱动技术。

# Go scheduler

## CPU-Bound、IO-Bound

CPU-Bound是一种从来没有产生过Goroutines自然进入和退出等待状态的情况的工作负载。这是一个不断进行计算的工作。一个计算Pi到第N位的线程就是CPU绑定的。

对于CPU-Bound的工作负载，你需要**并行性(多核)**来利用并发性。一个处理多个Goroutine的操作系统/硬件线程并不高效，因为这些Goroutine并没有作为其工作负载的一部分进出等待状态。由于将Goroutine移入和移出操作系统线程的延迟成本（所需时间），拥有比操作系统/硬件线程更多的Goroutine会减慢工作负载的执行。上下文切换为你的工作负载创造了一个 "停止世界 "的事件，因为你的工作负载在切换期间没有被执行，而它本来是可以被执行的。

IO-Bound是一个导致Goroutine自然进入等待状态的工作负载。这种工作包括通过网络请求访问资源，或对操作系统进行系统调用，或等待一个事件的发生。一个需要读取文件的Goroutine将是IO-Bound。我将包括同步事件（mutexes, atomic），这些事件导致Goroutine等待，也是这个类别的一部分。

对于IO-Bound的工作负载，你不需要**并行性(多核**)来使用并发性。一个操作系统/硬件线程可以高效地处理多个Goroutines，因为Goroutines作为其工作负载的一部分，自然会在等待状态中移动。拥有比操作系统/硬件线程更多的Goroutines可以加速工作负载的执行，因为将Goroutines移入和移出操作系统线程的延迟成本不会产生一个 "停止世界 "事件。你的工作负载自然会停止，这使得不同的Goroutine可以有效地利用同一个操作系统/硬件线程，而不是让操作系统/硬件线程闲置。

## 上下文切换

上下文切换被认为是昂贵的，因为它需要时间来交换core上的线程和关闭。在上下文切换过程中发生的延迟量取决于不同的因素，但它需要1000到1500纳秒的时间也不是不合理的。考虑到硬件应该能够合理地执行（平均）每纳秒12条指令，一个上下文切换会使你损失12000到18000条指令的延迟。从本质上讲，你的程序在上下文切换期间失去了执行大量指令的能力。

如果你有一个专注于IO-Bound工作的程序，那么上下文切换将是一个优势。一旦一个线程进入等待状态，另一个处于可运行状态的线程就会接替它的位置。这允许core一直在做工作。这是调度的一个最重要的方面。如果有工作（处于可运行状态的线程）要做，就不要让一个core闲置下来。

如果你的程序专注于CPU绑定的工作，那么上下文切换将是一个性能噩梦。因为Thread总是有工作要做，上下文切换是在阻止工作的进行。这种情况与IO-Bound工作负载的情况截然不同

## Go scheduler设计哲学

**就像操作系统线程是在core上进行上下文切换，Goroutines是在M上进行上下文切换**

**压榨更多CPU的性能**

从本质上讲，Go将 IO/Blocking work变成了操作系统层面上的CPU-bound work。由于所有的上下文切换都发生在**应用层面**，我们不会像使用OS Threads时那样，每次上下文切换损失约12k指令（平均）。在Go中，这些相同的上下文切换会让你损失约200纳秒或约2.4k指令。调度器也有助于提高缓存线的效率和NUMA。这就是为什么我们不需要比我们的虚拟核心更多的线程。在Go中，随着时间的推移，我们有可能完成更多的工作，因为Go调度器试图使用更少的线程，在每个线程上做更多的工作，这有助于减少操作系统和硬件的负载。

## Go netpoller

### 基本原理

Go netpoller 通过在底层对 epoll/kqueue/iocp 的封装，从而实现了**使用同步编程模式达到异步执行的效果**（socket读写未就绪时，conn.read、conn.write阻塞。阻塞时goroutine被gopark，netpoller检测socket读写是否就绪，若就绪，conn.read、conn.write正常执行）。总结来说，所有的网络操作都以网络描述符 netFD 为中心实现。netFD 与底层 PollDesc 结构绑定，当在一个 netFD 上读写遇到 EAGAIN 错误时，就将当前 goroutine 存储到这个 netFD 对应的 PollDesc 中，同时调用 gopark 把当前 goroutine 给 park 住，直到这个 netFD 上再次发生读写事件，才将此 goroutine 给 ready 激活重新运行。显然，在底层通知 goroutine 再次发生读写等事件的方式就是 epoll/kqueue/iocp 等事件驱动机制。

### 1、主要流程

![](https://s2.loli.net/2022/07/23/QedOnMCSWA498i2.png)

1. **新连接建立**。client 连接 server 的时候，listener 通过 accept 调用接收新 connection，每一个新 connection 都启动一个 goroutine 处理，accept 调用会把该 connection 的 fd 连带所在的 goroutine 上下文信息封装注册到 epoll 的监听列表里去。(**系统调用epoll_ctl，epoll采用红黑树来存储所有监听的 fd**)
2. **阻塞**。当 goroutine 调用 `conn.Read` 或者 `conn.Write` 等需要阻塞等待的函数时，会被 `gopark` 给封存起来并使之休眠，让 P 去执行本地调度队列里的下一个可执行的 goroutine，往后 Go scheduler 会在循环调度的 `runtime.schedule()` 函数以及 sysmon 监控线程中调用 `runtime.netpoll` 以获取可运行的 goroutine 列表并通过调用 injectglist 把剩下的 g 放入全局调度队列或者当前 P 本地调度队列去重新执行。（goroutine**调度**）
3. **唤醒**。netpoller 通过 `runtime.netpoll`唤醒那些在 I/O wait 的 goroutine 的。（系统调用epoll_wait，检查并唤醒就绪的FD）

Go 在多种场景下都可能会调用 `netpoll` 检查FD状态，`netpoll` 里会调用 `epoll_wait` 从 epoll 的 `eventpoll.rdllist` **就绪双向链表**返回，从而得到 I/O 就绪的 socket fd 列表，并根据取出最初调用 `epoll_ctl` 时保存的上下文信息，恢复 `g`。所以执行完`netpoll` 之后，会返回一个就绪 fd 列表对应的 goroutine 链表，接下来将就绪的 goroutine 通过调用 `injectglist` 加入到**全局调度队列**或者 P 的**本地调度队列**中，启动 M 绑定 P 去执行。

### 2、代码实现

#### net.Listen

1. 是调用 `epollcreate1` 创建一个新的 `epoll` 文件描述符，这个文件描述符会在整个程序的生命周期中使用；
2. 通过 [`runtime.nonblockingPipe`](https://draveness.me/golang/tree/runtime.nonblockingPipe) 创建一个用于通信的管道；
3. 使用 `epollctl` 将用于读取数据的文件描述符打包成 `epollevent` 事件加入监听；

```go
// 调用 linux 系统调用 socket 创建 listener fd 并设置为为阻塞 I/O
s, err := socketFunc(family, sotype|syscall.SOCK_NONBLOCK|syscall.SOCK_CLOEXEC, proto)
// On Linux the SOCK_NONBLOCK and SOCK_CLOEXEC flags were
// introduced in 2.6.27 kernel and on FreeBSD both flags were
// introduced in 10 kernel. If we get an EINVAL error on Linux
// or EPROTONOSUPPORT error on FreeBSD, fall back to using
// socket without them.
 
socketFunc        func(int, int, int) (int, error)  = syscall.Socket
 
// 用上面创建的 listener fd 初始化 listener netFD
if fd, err = newFD(s, family, sotype, net); err != nil {
	poll.CloseFunc(s)
	return nil, err
}
 
// 对 listener fd 进行 bind&listen 操作，并且调用 init 方法完成初始化
func (fd *netFD) listenStream(laddr sockaddr, backlog int, ctrlFn func(string, string, syscall.RawConn) error) error {
	...
  
	// 完成绑定操作
	if err = syscall.Bind(fd.pfd.Sysfd, lsa); err != nil {
		return os.NewSyscallError("bind", err)
	}
  
	// 完成监听操作
	if err = listenFunc(fd.pfd.Sysfd, backlog); err != nil {
		return os.NewSyscallError("listen", err)
	}
  
	// 调用 init，内部会调用 poll.FD.Init，最后调用 pollDesc.init
	if err = fd.init(); err != nil {
		return err
	}
	lsa, _ = syscall.Getsockname(fd.pfd.Sysfd)
	fd.setAddr(fd.addrFunc()(lsa), nil)
	return nil
}
 
// 使用 sync.Once 来确保一个 listener 只持有一个 epoll 实例
var serverInit sync.Once
 
// netFD.init 会调用 poll.FD.Init 并最终调用到 pollDesc.init，
// 它会创建 epoll 实例并把 listener fd 加入监听队列
func (pd *pollDesc) init(fd *FD) error {
	// runtime_pollServerInit 通过 `go:linkname` 链接到具体的实现函数 poll_runtime_pollServerInit，
	// 接着再调用 netpollGenericInit，然后会根据不同的系统平台去调用特定的 netpollinit 来创建 epoll 实例
	serverInit.Do(runtime_pollServerInit)
  
	// runtime_pollOpen 内部调用了 netpollopen 来将 listener fd 注册到 
	// epoll 实例中，另外，它会初始化一个 pollDesc 并返回
	ctx, errno := runtime_pollOpen(uintptr(fd.Sysfd))
	if errno != 0 {
		if ctx != 0 {
			runtime_pollUnblock(ctx)
			runtime_pollClose(ctx)
		}
		return syscall.Errno(errno)
	}
	// 把真正初始化完成的 pollDesc 实例赋值给当前的 pollDesc 代表自身的指针，
	// 后续使用直接通过该指针操作
	pd.runtimeCtx = ctx
	return nil
}
 
var (
	// 全局唯一的 epoll fd，只在 listener fd 初始化之时被指定一次
	epfd int32 = -1 // epoll descriptor
)
 
// netpollinit 会创建一个 epoll 实例，然后把 epoll fd 赋值给 epfd，
// 后续 listener 以及它 accept 的所有 sockets 有关 epoll 的操作都是基于这个全局的 epfd
func netpollinit() {
	epfd = epollcreate1(_EPOLL_CLOEXEC)
	if epfd < 0 {
		epfd = epollcreate(1024)
		if epfd < 0 {
			println("runtime: epollcreate failed with", -epfd)
			throw("runtime: netpollinit failed")
		}
		closeonexec(epfd)
	}
	r, w, errno := nonblockingPipe()
	if errno != 0 {
		println("runtime: pipe failed with", -errno)
		throw("runtime: pipe failed")
	}
	ev := epollevent{
		events: _EPOLLIN,
	}
	*(**uintptr)(unsafe.Pointer(&ev.data)) = &netpollBreakRd
	errno = epollctl(epfd, _EPOLL_CTL_ADD, r, &ev)
	if errno != 0 {
		println("runtime: epollctl failed with", -errno)
		throw("runtime: epollctl failed")
	}
	netpollBreakRd = uintptr(r)
	netpollBreakWr = uintptr(w)
}
 
// netpollopen 会被 runtime_pollOpen 调用，注册 fd 到 epoll 实例，
// 注意这里使用的是 epoll 的 ET 模式，同时会利用万能指针把 pollDesc 保存到 epollevent 的一个 8 位的字节数组 data 里
func netpollopen(fd uintptr, pd *pollDesc) int32 {
	var ev epollevent
	ev.events = _EPOLLIN | _EPOLLOUT | _EPOLLRDHUP | _EPOLLET
	*(**pollDesc)(unsafe.Pointer(&ev.data)) = pd
	return -epollctl(epfd, _EPOLL_CTL_ADD, int32(fd), &ev)
}
```

#### Listener.Accept()

> gopark后等待被唤醒

```go
func (fd *FD) Accept() (int, syscall.Sockaddr, string, error) {
	...
	for {
		// 使用 linux 系统调用 accept 接收新连接，创建对应的 socket
		s, rsa, errcall, err := accept(fd.Sysfd)
		// 因为 listener fd 在创建的时候已经设置成非阻塞的了，所以 accept 方法会直接返回，不管有没有新连接到   来；如果 err == nil 则表示正常建立新连接，直接返回
		if err == nil {
			return s, rsa, "", err
		}
		switch err {
   // 如果 err != nil，则判断 err == syscall.EAGAIN，符合条件则进入 pollDesc.waitRead 方法
		case syscall.EAGAIN:
			if fd.pd.pollable() {
				// 如果当前没有发生期待的 I/O 事件，那么 waitRead 会通过 park goroutine 让逻辑 block 在这里
				if err = fd.pd.waitRead(fd.isFile); err == nil {
					continue
				}
			}
		case syscall.ECONNABORTED:
			// This means that a socket on the listen
			// queue was closed before we Accept()ed it;
			// it's a silly error, so try again.
			continue
		}
		return -1, nil, errcall, err
	}
}
```

1. 服务端的 netFD 在 `listen` 时会创建 epoll 的实例，并将 listenerFD 加入 epoll 的事件队列
2. netFD 在 `accept` 时将返回的 connFD 也加入 epoll 的事件队列。
3. netFD 在读写时出现 `syscall.EAGAIN` 错误，通过 pollDesc 的 `waitRead` 方法将当前的 goroutine park 住，直到 ready，从 pollDesc 的 `waitRead` 中返回

#### Conn.Read/Conn.Write

> gopark后等待被唤醒

```go
func (fd *FD) Read(p []byte) (int, error) {
  ...
  for {
		// 尝试从该 socket 读取数据，因为 socket 在被 listener accept 的时候设置成
		// 了非阻塞 I/O，所以这里同样也是直接返回，不管有没有可读的数据
		n, err := ignoringEINTRIO(syscall.Read, fd.Sysfd, p)
		if err != nil {
			n = 0
			// err == syscall.EAGAIN 表示当前没有期待的 I/O 事件发生，也就是 socket 不可读
			if err == syscall.EAGAIN && fd.pd.pollable() {
				// 如果当前没有发生期待的 I/O 事件，那么 waitRead
				// 会通过 park goroutine 让逻辑 block在这里
				if err = fd.pd.waitRead(fd.isFile); err == nil {
					continue
				}
			}
		}
	}
}


// Write implements io.Writer.
func (fd *FD) Write(p []byte) (int, error) {
 ...
	for {
    ...
    //syscall.write
    n, err := ignoringEINTRIO(syscall.Write, fd.Sysfd, p[nn:max])
		if err == syscall.EAGAIN && fd.pd.pollable() {
      //waitWrite
			if err = fd.pd.waitWrite(fd.isFile); err == nil {
				continue
			}
		}
    ...
	}
}

// accept、read
func (pd *pollDesc) waitRead(isFile bool) error {
	return pd.wait('r', isFile)
}
// write
func (pd *pollDesc) waitWrite(isFile bool) error {
	return pd.wait('w', isFile)
}
func (pd *pollDesc) wait(mode int, isFile bool) error {
	if pd.runtimeCtx == 0 {
		return errors.New("waiting for unsupported file type")
	}
	res := runtime_pollWait(pd.runtimeCtx, mode)
	return convertErr(res, isFile)
}

//runtime_pollWait
func poll_runtime_pollWait(pd *pollDesc, mode int) int {
 ...
	for !netpollblock(pd, int32(mode), false) {
		errcode = netpollcheckerr(pd, int32(mode))
		if errcode != pollNoError {
			return errcode
		}
	}
	return pollNoError
}

//netpollblock
func netpollblock(pd *pollDesc, mode int32, waitio bool) bool {
 ...
	if waitio || netpollcheckerr(pd, mode) == 0 {
    //gopark
		gopark(netpollblockcommit, unsafe.Pointer(gpp), waitReasonIOWait, traceEvGoBlockNet, 5)
	}
  ...
}
```

netFD的`Read`操作在系统调用Read后，当有**syscall.EAGAIN**错误发生时，`WaitRead`将当前读这个connFD的goroutine给**park**住，直到这个connFD上的读事件再次发生为止，`waitRead`调用返回，继续for循环的执行。netFD的Write方法和Read的实现原理是一样的，都是在碰到EAGAIN错误的时候将当前goroutine给park住直到socket再次可写为止。

#### netpoll

**netpoll** 就是从epoll wait得到所有发生事件的fd，并将每个fd对应的goroutine通过链表返回。这个操作是在goroutine调度器中使用的，用来将因为IO wait而阻塞的goroutine重新调度。

```go
func netpoll(block bool) *g {
    if epfd == -1 {
        return nil
    }
     := int32(-1)
    if !block {
        waitms = 0
    }
    var events [128]epollevent
retry:
    //系统调用epoll_wait
    n := epollwait(epfd, &events[0], int32(len(events)), waitms)
    if n < 0 {
        goto retry
    }
    var gp guintptr
    for i := int32(0); i < n; i++ {
        ev := &events[i]
        var mode int32
        if ev.events&(_EPOLLIN|_EPOLLRDHUP|_EPOLLHUP|_EPOLLERR) != 0 {
            mode += 'r'
        }
        if ev.events&(_EPOLLOUT|_EPOLLHUP|_EPOLLERR) != 0 {
            mode += 'w'
        }
        if mode != 0 {
            pd := *(**pollDesc)(unsafe.Pointer(&ev.data))
            netpollready(&gp, pd, mode)
        }
    }
    if block && gp == 0 {
        goto retry
    }
    return gp.ptr()
}
```

##### 调用时机

- runtime.schedule()中的runtime.findrunable()

  ```go
  func findrunnable() (gp *g, inheritTime bool) {
    ...
    //// Poll network.
     if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Load64(&sched.lastpoll) != 0 {
        if list := netpoll(0); !list.empty() { // non-blocking
           gp := list.pop()
           injectglist(&list)
           casgstatus(gp, _Gwaiting, _Grunnable)
           if trace.enabled {
              traceGoUnpark(gp, 0)
           }
           return gp, false
        }
     }
    
  }
  ```

- sysmon监控线程中

  ```go
  // poll network if not polled for more than 10ms
  if netpollinited() && lastpoll != 0 && lastpoll+10*1000*1000 < now {
     atomic.Cas64(&sched.lastpoll, uint64(lastpoll), uint64(now))
     list := netpoll(0) // non-blocking - returns list of goroutines
        incidlelocked(-1)
        injectglist(&list)
        incidlelocked(1)
     }
  }
  ```

## 系统调用

<img src="https://s2.loli.net/2022/06/26/5ZxlEtCMRrhXisW.png" style="zoom:40%;" />

#### 分级保护域 (Protection ring)

<img src="https://s2.loli.net/2022/06/26/3abmMlQkcGXA7gy.png" style="zoom:50%;" />

In x86 protected mode, the CPU is always in one of 4 rings. The Linux kernel only uses 0 and 3:

- 0 for kernel
- 3 for users

This is the most hard and fast definition of kernel vs userland.

内核可以访问 Ring0，内核是大多数操作系统的核心部分，可以访问所有内容。这里运行的代码是在内核模式下运行的。在内核模式下运行的进程会影响整个系统；如果此处出现任何故障，则可能会导致系统关闭。Ring0 这个环可以直接访问 CPU 和系统内存，因此任何需要使用其中任何一个的指令都将在此处执行。

Ring3 是特权最少的环，运行在用户模式下的用户进程可以访问该环。这是您计算机上运行的大多数应用程序所在的位置。该环无法直接访问 CPU 或内存，因此必须将涉及这些的任何指令传递给Ring0

#### 系统调用上下文切换

Ring0<-->Ring3

<img src="https://s2.loli.net/2022/06/26/A1DiXNJxrjeadYC.png" style="zoom:50%;" />

#### 异步系统调用(不会阻塞M)

网络相关系统调用

![](https://s2.loli.net/2022/07/23/pWPxHFAvtmki2bC.png)

![](https://s2.loli.net/2022/07/23/FdYLtG9ojS2Bf34.png)

![](https://s2.loli.net/2022/07/23/S4GAmIb9R5Y1jfl.png)

#### 同步系统调用(会阻塞M)

动态演示：https://www.figma.com/proto/ounOboEYjlzBwcOhPgE2Z5/syscall?page-id=11%3A2&node-id=17%3A219&viewport=35%2C425%2C0.3724002540111542&scaling=contain

如操作文件相关的系统调用为同步系统调用、使用CGO的相关场景

> Windows操作系统确实具有异步进行基于文件的系统调用的能力。从技术上讲，在Windows上运行时，可以使用网络轮询

![](https://s2.loli.net/2022/07/23/nsMe2NhrykJAODg.png)

![](https://s2.loli.net/2022/07/23/eZLpMYtK61Fw2XW.png)

![](https://s2.loli.net/2022/07/23/zNCwbfPaenpDLW8.png)

## 存在的问题

goroutine 虽然非常轻量，它的自定义栈内存初始值仅为 2KB，后面按需扩容；海量连接的业务场景下， `goroutine-per-connection` ，此时 goroutine 数量以及消耗的资源就会呈线性趋势暴涨，虽然 Go scheduler 内部做了 g 的缓存链表，可以一定程度上缓解高频创建销毁 goroutine 的压力，但是对于瞬时性暴涨的长连接场景就无能为力了，**大量的 goroutines 会被不断创建出来**，从而对 Go runtime scheduler 造成极大的调度压力和侵占系统资源，然后资源被侵占又反过来影响 Go scheduler 的调度，进而导致性能下降。

# Reactors

## 两种模型

### 主从多 Reactors

![](https://s2.loli.net/2022/07/23/Yha1LNXTDJjfCwP.png)

时序图：

![](https://s2.loli.net/2022/07/23/BgyhoRefLz9XP8W.png)

### 主从多 Reactors + 线程/Go 程池

可以避免业务代码阻塞event-loop

![](https://s2.loli.net/2022/07/23/FHV6uLgSIR2Eo9k.png)

时序图：

![](https://s2.loli.net/2022/07/23/A5hieR3fCLUbVNx.png)

`gnet` 内部集成了 `ants` 以及提供了 `pool.goroutine.Default()` 方法来初始化一个 `ants` goroutine 池，然后你可以把 `EventHandler.React` 中阻塞的业务逻辑提交到 goroutine 池里执行，最后在 goroutine 池里的代码调用 `gnet.Conn.AsyncWrite([]byte)` 方法把处理完阻塞逻辑之后得到的输出数据异步写回客户端，这样就可以避免阻塞 event-loop 线程。

## reactor工作流程

- Server 端完成在 `bind&listen` 之后，将 listenfd 注册到 epollfd 中，最后进入 event-loop 事件循环。循环过程中会调用 `select/poll/epoll_wait` 阻塞等待，若有在 listenfd 上的新连接事件则解除阻塞返回，并调用 `socket.accept` 接收新连接 connfd，并将 connfd 加入到 epollfd 的 I/O 复用（监听）队列。
- 当 connfd 上发生可读/可写事件也会解除 `select/poll/epoll_wait` 的阻塞等待，然后进行 I/O 读写操作，这里读写 I/O 都是非阻塞 I/O，这样才不会阻塞 event-loop 的下一个循环。然而，这样容易割裂业务逻辑，不易理解和维护。
- 调用 `read` 读取数据之后进行解码并放入队列中，等待工作线程处理。
- 工作线程处理完数据之后，返回到 event-loop 线程，由这个线程负责调用 `write` 把数据写回 client。

## main reactor

> 对应一个goroutine，event-loop事件循环

```go
func (el *eventloop) activateMainReactor(lockOSThread bool) {
   if lockOSThread {
      runtime.LockOSThread()
      defer runtime.UnlockOSThread()
   }

   defer el.engine.signalShutdown()

   err := el.poller.Polling(func(fd int, filter int16) error { return el.engine.accept(fd, filter) })
   if err == errors.ErrEngineShutdown {
      el.engine.opts.Logger.Debugf("main reactor is exiting in terms of the demand from user, %v", err)
   } else if err != nil {
      el.engine.opts.Logger.Errorf("main reactor is exiting due to error: %v", err)
   }
}
```

## sub reactor

> 对应多个goroutine（逻辑核心数）

```go
func (el *eventloop) activateSubReactor(lockOSThread bool) {
   if lockOSThread {
      runtime.LockOSThread()
      defer runtime.UnlockOSThread()
   }

   defer func() {
      el.closeAllSockets()
      el.engine.signalShutdown()
   }()

   err := el.poller.Polling(func(fd int, filter int16) (err error) {
      if c, ack := el.connections[fd]; ack {
         switch filter {
         case netpoll.EVFilterSock:
            err = el.closeConn(c, unix.ECONNRESET)
         case netpoll.EVFilterWrite: //I/O可写事件
            if !c.outboundBuffer.IsEmpty() { 
               err = el.write(c)
            }
         case netpoll.EVFilterRead: //I/O可读事件
            err = el.read(c)
         }
      }
      return
   })
   ...
}

// Polling blocks the current goroutine, waiting for network-events.
func (p *Poller) Polling(callback func(fd int, ev uint32) error) error {
 ...
	for {
    //系统调用epoll_wait
		n, err := unix.EpollWait(p.fd, el.events, msec)
		if n == 0 || (n < 0 && err == unix.EINTR) {
			msec = -1
			runtime.Gosched()
			continue
		} else if err != nil {
			logging.Errorf("error occurs in epoll: %v", os.NewSyscallError("epoll_wait", err))
			return err
		}
		msec = 0
  }
  ...
}

func (el *eventloop) read(c *conn) error {
	n, err := unix.Read(c.fd, el.buffer)
	if err != nil {
		if err == unix.EAGAIN {
			return nil
		}
		return el.closeConn(c, os.NewSyscallError("read", err))
	}
	if n == 0 {
		return el.closeConn(c, os.NewSyscallError("read", unix.ECONNRESET))
	}

	c.buffer = el.buffer[:n]
  //业务处理代码
	action := el.eventHandler.OnTraffic(c)
	switch action {
	case None:
	case Close:
		return el.closeConn(c, nil)
	case Shutdown:
		return gerrors.ErrEngineShutdown
	}
	_, _ = c.inboundBuffer.Write(c.buffer)

	return nil
}
```

# 相关问题

### 为什么需要缓存区(用户空间)

**数据——流缓存区（用户态）—— 内核缓存区（内核态）——磁盘**

减少调用read和write的次数(**减少用户态和内核态来回切换**)，提高了磁盘的I/O效率

### 用户态和内核态切换的时机

1）**系统调用**。

2）异常事件： 当CPU正在执行运行在用户态的程序时，突然发生某些预先不可知的异常事件，这个时候就会触发从当前用户态执行的进程转向内核态执行相关的异常事件，典型的如**缺页异常**。

3）外围设备的**中断**：当外围设备完成用户的请求操作后，会像CPU发出中断信号，此时，CPU就会暂停执行下一条即将要执行的指令，转而去执行中断信号对应的处理程序，如果先前执行的指令是在用户态下，则自然就发生从用户态到内核态的转换。

### 用户态和内核态切换的代价

当程序中有系统调用语句，程序执行到系统调用时，首先使用类似`int 80H`的**软中断指令**，保存现场，去的系统调用号，在内核态执行，然后恢复现场，每个进程都会有两个栈，一个内核态栈和一个用户态栈。当执行int中断执行时就会由用户态，栈转向内核栈。系统调用时需要进行栈的切换。而且内核代码对用户不信任，需要进行额外的检查。系统调用的返回过程有很多额外工作，比如检查是否需要调度等。

