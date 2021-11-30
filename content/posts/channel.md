---
title: "go之channel"
date: 2021-11-30T9:38:03+08:00
draft: false
---

# channel

## channel的作用

1. goroutine之间安全通信

   hchan mutex        

2. goroutine之间数据传递

3. FIFO语义

   copying into and out of hchan buffer

   - 先从 Channel 读取数据的 Goroutine 会先接收到数据；
   - 先向 Channel 发送数据的 Goroutine 会得到先发送数据的权利；

4. 可以使得goroutine阻塞和唤醒

   hchan sudog queues

   calls into the runtime scheduler(gopark, goready)

## 设计原理

![shared-memory](https://img.draveness.me/2020-01-28-15802171487042-shared-memory.png)

​                                               **多线程使用共享内存传递数据**

![channel-and-goroutines](https://img.draveness.me/2020-01-28-15802171487080-channel-and-goroutines.png)

​                                           **Goroutine 使用 Channel 传递数据**

### 先入先出

目前的 Channel 收发操作均遵循了先入先出（FIFO）的设计，具体规则如下：

- 先从 Channel 读取数据的 Goroutine 会先接收到数据；
- 先向 Channel 发送数据的 Goroutine 会得到先发送数据的权利；

## 数据结构

Channel 在运行时的内部表示是 [`runtime.hchan`](https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L32)，该结构体中包含了一个用于保护成员变量的**互斥锁**，从某种程度上说，Channel 是一个**用于同步和通信的有锁队列**

Go 语言的 Channel 在运行时使用 [`runtime.hchan`](https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L32) 结构体表示。我们在 Go 语言中创建新的 Channel 时，实际上创建的都是如下所示的结构体：

```go
type hchan struct {
    // chan 里元素数量
    qcount   uint
    // chan 底层循环数组的长度
    dataqsiz uint
    // 指向底层循环数组的指针
    // 只针对有缓冲的 channel
    buf      unsafe.Pointer
    // chan 中元素大小
    elemsize uint16
    // chan 是否被关闭的标志
    closed   uint32
    // chan 中元素类型
    elemtype *_type // element type
    // 已发送元素在循环数组中的索引
    sendx    uint   // send index
    // 已接收元素在循环数组中的索引
    recvx    uint   // receive index
    // 等待接收的 goroutine 队列
    recvq    waitq  // list of recv waiters
    // 等待发送的 goroutine 队列
    sendq    waitq  // list of send waiters

    // 保护 hchan 中所有字段
    lock mutex
}
//waitq 是 sudog 的一个双向链表，而 sudog 实际上是对 goroutine 的一个封装：
type waitq struct {
    first *sudog
    last  *sudog
}

type sudog struct {
	g *g

	next *sudog
	prev *sudog
	elem unsafe.Pointer // data element (may point to stack)
  。。。
}

```

例如，创建一个容量为 6 的，元素为 int 型的 channel 数据结构如下 ：

![img](https://user-images.githubusercontent.com/7698088/61179068-806ee080-a62d-11e9-818c-16af42025b1b.png)

`recvq` 的数据结构如下(双向循环链表，此时recvq中有两个sudog，两个goroutine被阻塞在接收操作)：

![img](https://user-images.githubusercontent.com/7698088/61179210-d3966280-a630-11e9-8c73-5a22340910a6.png)

## 创建通道

新建一个 chan 后，内存在**堆**上分配，大概长这样： ![img](https://user-images.githubusercontent.com/7698088/61337268-4d179600-a867-11e9-98ac-f979e3da00a6.png)

```go
ch:=make(chan int,3)
//make关键字 --> runtime.makechan 
//在堆上分配内存，返回一个指针，所以我们可以用ch本身传参，不需要ch的指针
```

## 发送数据

```go
// 位于 src/runtime/chan.go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    // 如果 channel 是 nil
    if c == nil {
        // 不能阻塞，直接返回 false，表示未发送成功
        if !block {
            return false
        }
        // 当前 goroutine 被挂起
        gopark(nil, nil, "chan send (nil chan)", traceEvGoStop, 2)
        throw("unreachable")
    }

    // 省略 debug 相关……

    // 针对select语句且有default的情况
    // 对于不阻塞的 send，快速检测失败场景（不用获取锁），好快速返回
    // 如果 channel 未关闭且 channel 没有多余的缓冲空间。这可能是：
    // 1. channel 是非缓冲型的，且等待接收队列里没有 goroutine
    // 2. channel 是缓冲型的，但循环数组已经装满了元素
    if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
        (c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
        return false
    }

    var t0 int64
    if blockprofilerate > 0 {
        t0 = cputicks()
    }

    // 锁住 channel，并发安全
    lock(&c.lock)        return false
        }
        // 当前 goroutine 被挂起

    // 如果 channel 关闭了
    if c.closed != 0 {
        // 解锁
        unlock(&c.lock)
        // 直接 panic
        panic(plainError("send on closed channel"))
    }

    // 如果接收队列里有 goroutine，直接将要发送的数据拷贝到接收 goroutine
    if sg := c.recvq.dequeue(); sg != nil {
        send(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true
    }

    // 对于缓冲型的 channel，如果还有缓冲空间
    if c.qcount < c.dataqsiz {
        // qp 指向 buf 的 sendx 位置
        qp := chanbuf(c, c.sendx)

        // ……

        // 将数据从 ep 处拷贝到 qp
        typedmemmove(c.elemtype, qp, ep)
        // 发送游标值加 1
        c.sendx++
        // 如果发送游标值等于容量值，游标值归 0
        if c.sendx == c.dataqsiz {
            c.sendx = 0
        }
        // 缓冲区的元素数量加一
        c.qcount++

        // 解锁
        unlock(&c.lock)
        return true
    }

    // 如果不需要阻塞，则直接返回错误
    if !block {
        unlock(&c.lock)
        return false
    }

    // channel 满了，发送方会被阻塞。接下来会构造一个 sudog

    // 获取当前 goroutine 的指针
    gp := getg()
    mysg := acquireSudog()
    mysg.releasetime = 0
    if t0 != 0 {
        mysg.releasetime = -1
    }

    mysg.elem = ep
    mysg.waitlink = nil
    mysg.g = gp
    mysg.selectdone = nil
    mysg.c = c
    gp.waiting = mysg
    gp.param = nil

    // 当前 goroutine 进入发送等待队列
    c.sendq.enqueue(mysg)

    // 当前 goroutine 被挂起
    goparkunlock(&c.lock, "chan send", traceEvGoBlockSend, 3)

    // 从这里开始被唤醒了（channel 有机会可以发送了）
    if mysg != gp.waiting {
        throw("G waiting list is corrupted")
    }
    gp.waiting = nil
    if gp.param == nil {
        if c.closed == 0 {
            throw("chansend: spurious wakeup")
        }
        // 被唤醒后，channel 关闭了。坑爹啊，panic
        panic(plainError("send on closed channel"))
    }
    gp.param = nil
    if mysg.releasetime > 0 {
        blockevent(mysg.releasetime-t0, 2)
    }
    // 去掉 mysg 上绑定的 channel
    mysg.c = nil
    releaseSudog(mysg)
    return true
}
```

在发送数据的逻辑执行之前会先为当前 Channel 加锁，防止发生竞争条件。如果 Channel 已经关闭，那么向该 Channel 发送数据时就会报`"send on closed channel"` 错误并中止程序。

因为 [`runtime.chansend`](https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L142) 函数的实现比较复杂，所以我们这里将该函数的执行过程分成以下的三个部分：

- 当存在等待的接收者时，通过 [`runtime.send`](https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L270) 直接将数据发送给阻塞的接收者；
- 当缓冲区存在空余空间时，将发送的数据写入 Channel 的缓冲区；
- 当不存在缓冲区或者缓冲区已满时，等待其他 Goroutine 从 Channel 接收数据；

### 1、直接发送

> 两种情况（前提channel没有关闭）：
>
> 1、有缓存但是此时为空且存在读等待的接收者（绕过buffer）
>
> 2、无缓存且存在读等待的接收者
>
> 两条件：1、channel没有关闭   2、存在读等待的接收者

如果目标 Channel 没有被关闭并且已经有处于读等待的 Goroutine，那么 [`runtime.chansend`](https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L142) 函数会从接收队列 `recvq` 中取出最先(先入先出原则)陷入等待的 Goroutine 并直接向它发送数据：

```go
// 如果接收队列里有 goroutine，直接将要发送的数据拷贝到接收 goroutine,绕过缓存区（如果有的话）
if sg := c.recvq.dequeue(); sg != nil {
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}
```

下图展示了 Channel 中存在等待数据的 Goroutine 时，向 Channel 发送数据的过程：

![channel-direct-send](https://img.draveness.me/2020-01-29-15802354027250-channel-direct-send.png)

​                                             **图 6-20 直接发送数据的过程**

发送数据时会调用 [`runtime.send`](https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L270)，该函数的执行可以分成两个部分：

1. 调用 [`runtime.sendDirect`](https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L313) 函数将发送的数据直接拷贝到 `x = <-c` 表达式中变量 `x` 所在的内存地址上；
2. 调用 [`runtime.goready`](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/proc.go#L313) 将等待接收数据的 Goroutine 标记成可运行状态 `Grunnable` 并把该 Goroutine 放到**发送方所在的处理器的 `runnext`** 上等待执行，该处理器在下一次调度时就会立刻唤醒数据的接收方；

```go
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if sg.elem != nil {
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	goready(gp, skip+1)
}
// 向一个非缓冲型的 channel 发送数据、从一个无元素的（非缓冲型或缓冲型但空）的 channel
// 接收数据，都会导致一个 goroutine 直接操作另一个 goroutine 的栈
// 由于 GC 假设对栈的写操作只能发生在 goroutine 正在运行中并且由当前 goroutine 来写
// 所以这里实际上违反了这个假设。可能会造成一些问题，所以需要用到写屏障来规避
func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
    // src 在当前 goroutine 的栈上，dst 是另一个 goroutine 的栈

    // 直接进行内存"搬迁"
    // 如果目标地址的栈发生了栈收缩，当我们读出了 sg.elem 后
    // 就不能修改真正的 dst 位置的值了
    // 因此需要在读和写之前加上一个屏障
    dst := sg.elem
    typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)
    memmove(dst, src, t.size)
}
//这里涉及到一个 goroutine 直接写另一个 goroutine 栈的操作，一般而言，不同 goroutine 的栈是各自独有的。而这也违反了 GC 的一些假设。为了不出问题，写的过程中增加了写屏障，保证正确地完成写操作。这样做的好处是减少了一次内存 copy：不用先拷贝到 channel 的 buf，直接由发送者到接收者，没有中间商赚差价，效率得以提高，完美。
```

需要注意的是，发送数据的过程只是将接收方的 Goroutine 放到了处理器的 `runnext` 中，程序没有立刻执行该 Goroutine。

![](/Users/gongqi/Nutstore Files/个人总结/pic/channel-直接发送.png)



直接发送的好处：

减少了一次内存 copy：不用先拷贝到 channel 的 buf，直接由发送者到接收者，没有中间商赚差价，效率得以提高，完美。

### 2、缓冲区

一种情况

> 有缓存区且channel没满

![channel-buffer-send](https://img.draveness.me/2020-01-28-15802171487104-channel-buffer-send.png)

如果当前 Channel 的缓冲区未满，向 Channel 发送的数据会存储在 Channel 中 `sendx` 索引所在的位置并将 `sendx` 索引加一，由于这里的 `buf` 是一个循环数组，所以当 `sendx` 等于 `dataqsiz` 时就会重新回到数组开始的位置。

### 3、阻塞发送

1. 调用 [`runtime.getg`](https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/stubs.go#L18) 获取发送数据使用的 Goroutine；
2. 执行 [`runtime.acquireSudog`](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/proc.go#L320) 函数获取 [`runtime.sudog`](https://github.com/golang/go/blob/895b7c85addfffe19b66d8ca71c31799d6e55990/src/runtime/runtime2.go#L342) 结构体并设置这一次阻塞发送的相关信息，例如发送的 Channel、是否在 Select 控制结构中和待发送数据的内存地址等；
3. 将刚刚创建并初始化的 [`runtime.sudog`](https://github.com/golang/go/blob/895b7c85addfffe19b66d8ca71c31799d6e55990/src/runtime/runtime2.go#L342) 加入发送等待队列，并设置到当前 Goroutine 的 `waiting` 上，表示 Goroutine 正在等待该 `sudog` 准备就绪；
4. 调用 [`runtime.goparkunlock`](https://github.com/golang/go/blob/cfe3cd903f018dec3cb5997d53b1744df4e53909/src/runtime/proc.go#L309) 函数将当前的 Goroutine 陷入沉睡等待唤醒；
5. 被调度器唤醒后会执行一些收尾工作，将一些属性置零并且释放 [`runtime.sudog`](https://github.com/golang/go/blob/895b7c85addfffe19b66d8ca71c31799d6e55990/src/runtime/runtime2.go#L342) 结构体；

### 4、小结

我们在这里可以简单梳理和总结一下使用 `ch <- i` 表达式向 Channel 发送数据时遇到的几种情况：

1. **如果当前 Channel 的 `recvq` 上存在已经被阻塞的 Goroutine，那么会直接将数据发送给当前的 Goroutine 并将其设置成下一个运行的 Goroutine；**
2. **如果 Channel 存在缓冲区并且其中还有空闲的容量，我们就会将数据直接存储到当前缓冲区 `sendx` 所在的位置上；**
3. **如果不满足上面的两种情况，就会创建一个 [`runtime.sudog`](https://github.com/golang/go/blob/895b7c85addfffe19b66d8ca71c31799d6e55990/src/runtime/runtime2.go#L342) 结构并将其加入 Channel 的 `sendq` 队列中，当前 Goroutine 也会陷入阻塞等待其他的协程从 Channel 接收数据；**

发送数据的过程中包含几个会触发 Goroutine 调度的时机：

1. 当 Channel 为nil时；

2. 发送数据时发现 Channel 上存在等待接收数据的 Goroutine，立刻设置处理器的 `runnext` 属性，但是并不会立刻触发调度；

   ```go
   func goready(gp *g, traceskip int) {
   	systemstack(func() {
   		ready(gp, traceskip, true)
   	})
   }
   ```

   

3. 发送数据时并没有找到接收方并且缓冲区已经满了，这时就会将自己加入 Channel 的 `sendq` 队列并调用 [`runtime.goparkunlock`](https://github.com/golang/go/blob/cfe3cd903f018dec3cb5997d53b1744df4e53909/src/runtime/proc.go#L309) 触发 Goroutine 的调度让出处理器的使用权；

   ```go
   func goparkunlock(lock *mutex, reason waitReason, traceEv byte, traceskip int) {
   	gopark(parkunlock_c, unsafe.Pointer(lock), reason, traceEv, traceskip)
   }
   ```

### 5、问题

对于有缓存channel，有无可能存在读等待的接收者且缓存不为空的情况，如果存在这种情况，一旦直接发送，那么数据不能保证先入先出了。

## 接收数据

```go
// entry points for <- c from compiled code
func chanrecv1(c *hchan, elem unsafe.Pointer) {
    chanrecv(c, elem, true)
}

func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
    _, received = chanrecv(c, elem, true)
    return
}
//接收操作有两种写法，一种带 "ok"，反应 channel 是否关闭；一种不带 "ok"，这种写法，当接收到相应类型的零值时无法知道是真实的发送者发送过来的值，还是 channel 被关闭后，返回给接收者的默认类型的零值。
// 位于 src/runtime/chan.go

// chanrecv 函数接收 channel c 的元素并将其写入 ep 所指向的内存地址。
// 如果 ep 是 nil，说明忽略了接收值。
// 如果 block == false，即非阻塞型接收，在没有数据可接收的情况下，返回 (false, false)
// 否则，如果 c 处于关闭状态，将 ep 指向的地址清零，返回 (true, false)
// 否则，用返回值填充 ep 指向的内存地址。返回 (true, true)
// 如果 ep 非空，则应该指向堆或者函数调用者的栈

func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
    // 省略 debug 内容 …………

    // 如果是一个 nil 的 channel
    if c == nil {
        // 如果不阻塞，直接返回 (false, false)
        if !block {
            return
        }
        // 否则，接收一个 nil 的 channel，goroutine 挂起
        gopark(nil, nil, "chan receive (nil chan)", traceEvGoStop, 2)
        // 不会执行到这里
        throw("unreachable")
    }

    // 在非阻塞模式下，快速检测到失败，不用获取锁，快速返回
    // 当我们观察到 channel 没准备好接收：
    // 1. 非缓冲型，等待发送列队 sendq 里没有 goroutine 在等待
    // 2. 缓冲型，但 buf 里没有元素
    // 之后，又观察到 closed == 0，即 channel 未关闭。
    // 因为 channel 不可能被重复打开，所以前一个观测的时候 channel 也是未关闭的，
    // 因此在这种情况下可以直接宣布接收失败，返回 (false, false)
    if !block && (c.dataqsiz == 0 && c.sendq.first == nil ||
        c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0) &&
        atomic.Load(&c.closed) == 0 {
        return
    }

    var t0 int64
    if blockprofilerate > 0 {
        t0 = cputicks()
    }

    // 加锁
    lock(&c.lock)

    // channel 已关闭，并且循环数组 buf 里没有元素
    // 这里可以处理非缓冲型关闭 和 缓冲型关闭但 buf 无元素的情况
    // 也就是说即使是关闭状态，但在缓冲型的 channel，
    // buf 里有元素的情况下还能接收到元素
    if c.closed != 0 && c.qcount == 0 {
        if raceenabled {
            raceacquire(unsafe.Pointer(c))
        }
        // 解锁
        unlock(&c.lock)
        if ep != nil {
            // 从一个已关闭的 channel 执行接收操作，且未忽略返回值
            // 那么接收的值将是一个该类型的零值
            // typedmemclr 根据类型清理相应地址的内存
            typedmemclr(c.elemtype, ep)
        }
        // 从一个已关闭的 channel 接收，selected 会返回true
        return true, false
    }

    // 等待发送队列里有 goroutine 存在，说明 buf 是满的
    // 这有可能是：
    // 1. 非缓冲型的 channel
    // 2. 缓冲型的 channel，但 buf 满了
    // 针对 1，直接进行内存拷贝（从 sender goroutine -> receiver goroutine）
    // 针对 2，接收到循环数组头部的元素，并将发送者的元素放到循环数组尾部
    if sg := c.sendq.dequeue(); sg != nil {
        // Found a waiting sender. If buffer is size 0, receive value
        // directly from sender. Otherwise, receive from head of queue
        // and add sender's value to the tail of the queue (both map to
        // the same buffer slot because the queue is full).
        recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
        return true, true
    }

    // 缓冲型，buf 里有元素，可以正常接收
    if c.qcount > 0 {
        // 直接从循环数组里找到要接收的元素
        qp := chanbuf(c, c.recvx)

        // …………

        // 代码里，没有忽略要接收的值，不是 "<- ch"，而是 "val <- ch"，ep 指向 val
        if ep != nil {
            typedmemmove(c.elemtype, ep, qp)
        }
        // 清理掉循环数组里相应位置的值
        typedmemclr(c.elemtype, qp)
        // 接收游标向前移动
        c.recvx++
        // 接收游标归零
        if c.recvx == c.dataqsiz {
            c.recvx = 0
        }
        // buf 数组里的元素个数减 1
        c.qcount--
        // 解锁
        unlock(&c.lock)
        return true, true
    }

    if !block {
        // 非阻塞接收，解锁。selected 返回 false，因为没有接收到值
        unlock(&c.lock)
        return false, false
    }

    // 接下来就是要被阻塞的情况了
    // 构造一个 sudog
    gp := getg()
    mysg := acquireSudog()
    mysg.releasetime = 0
    if t0 != 0 {
        mysg.releasetime = -1
    }

    // 待接收数据的地址保存下来
    mysg.elem = ep
    mysg.waitlink = nil
    gp.waiting = mysg
    mysg.g = gp
    mysg.selectdone = nil
    mysg.c = c
    gp.param = nil
    // 进入channel 的等待接收队列
    c.recvq.enqueue(mysg)
    // 将当前 goroutine 挂起
    goparkunlock(&c.lock, "chan receive", traceEvGoBlockRecv, 3)

    // 被唤醒了，接着从这里继续执行一些扫尾工作
    if mysg != gp.waiting {
        throw("G waiting list is corrupted")
    }
    gp.waiting = nil
    if mysg.releasetime > 0 {
        blockevent(mysg.releasetime-t0, 2)
    }
    closed := gp.param == nil
    gp.param = nil
    mysg.c = nil
    releaseSudog(mysg)
    return true, !closed
}
```

### 1、直接接收

> 两种情况（前提channel没有关闭）：
>
> 1、有缓存但是此时满了且存在写等待的发送者
>
> 2、无缓存且存在写等待的发送者
>
> 两条件：1、channel没有关闭   2、存在写等待的发送者

该函数会根据缓冲区的大小分别处理不同的情况：

- 如果 Channel 不存在缓冲区；
  1. 调用 [`runtime.recvDirect`](https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L326) 函数会将 Channel 发送队列中 Goroutine 存储的 `elem` 数据拷贝到目标内存地址中；
- 如果 Channel 存在缓冲区；
  1. 将缓存区recvx指向的数据拷贝到接收方的内存地址；
  2. 将发送队列头的数据拷贝到缓冲区中，释放一个阻塞的发送方；

无论发生哪种情况，运行时都会调用 [`runtime.goready`](https://draveness.me/golang/tree/runtime.goready) 函数将当前处理器的 `runnext` 设置成发送数据的 Goroutine，在调度器下一次调度时将阻塞的发送方唤醒。

### 2、缓存区

如果接收数据的内存地址不为空，那么就会直接使用 [`runtime.typedmemmove`](https://github.com/golang/go/blob/db16de920370892b0241d3fa0617dddff2417a4d/src/runtime/mbarrier.go#L156) 将缓冲区中的数据拷贝到内存中、清除队列中的数据并完成收尾工作。

![channel-buffer-receive](https://img.draveness.me/2020-01-28-15802171487125-channel-buffer-receive.png)

### 3、阻塞接收

当 Channel 的发送队列中**不存在等待的 Goroutine 并且缓冲区中也不存在任何数据时**，从管道中接收数据的操作会变成阻塞操作，然而不是所有的接收操作都是阻塞的，与 `select` 语句结合使用时就可能会使用到非阻塞的接收操作：

在正常的接收场景中，我们会使用 [`runtime.sudog`](https://github.com/golang/go/blob/895b7c85addfffe19b66d8ca71c31799d6e55990/src/runtime/runtime2.go#L342) 结构体将当前 Goroutine 包装成一个处于等待状态的 Goroutine 并将其加入到接收队列中。

完成入队之后，上述代码还会调用 [`runtime.goparkunlock`](https://github.com/golang/go/blob/cfe3cd903f018dec3cb5997d53b1744df4e53909/src/runtime/proc.go#L309) 函数立刻触发 Goroutine 的调度，让出处理器的使用权并等待调度器的调度。

### 4、小结

我们梳理一下从 Channel 中接收数据时可能会发生的五种情况：

1. 如果 Channel 为空，那么就会直接调用 [`runtime.gopark`](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/proc.go#L287) 挂起当前 Goroutine；
2. 如果 Channel 已经关闭并且缓冲区没有任何数据，[`runtime.chanrecv`](https://github.com/golang/go/blob/e35876ec6591768edace6c6f3b12646899fd1b11/src/runtime/chan.go#L422) 函数会直接返回；
3. 如果 Channel 的 `sendq` 队列中存在挂起的 Goroutine，就会将 `recvx` 索引所在的数据拷贝到接收变量所在的内存空间上并将 `sendq` 队列中 Goroutine 的数据拷贝到缓冲区；
4. 如果 Channel 的缓冲区中包含数据就会直接读取 `recvx` 索引对应的数据；
5. 在默认情况下会挂起当前的 Goroutine，将 [`runtime.sudog`](https://github.com/golang/go/blob/895b7c85addfffe19b66d8ca71c31799d6e55990/src/runtime/runtime2.go#L342) 结构加入 `recvq` 队列并陷入休眠等待调度器的唤醒；

我们总结一下从 Channel 接收数据时，会触发 Goroutine 调度的两个时机：

1. 当 Channel 为nil时；
2. 接收数据时发现 Channel 上存在等待发送数据的 Goroutine，立刻设置处理器的 `runnext` 属性，但是并不会立刻触发调度
3. 当缓冲区中不存在数据并且也不存在数据的发送者时；

## 关闭管道

```go
func closechan(c *hchan) {
    // 关闭一个 nil channel，panic
    if c == nil {
        panic(plainError("close of nil channel"))
    }

    // 上锁
    lock(&c.lock)
    // 如果 channel 已经关闭
    if c.closed != 0 {
        unlock(&c.lock)
        // panic
        panic(plainError("close of closed channel"))
    }

    // …………

    // 修改关闭状态
    c.closed = 1

    var glist *g

    // 将 channel 所有等待接收队列的里 sudog 释放
    for {
        // 从接收队列里出队一个 sudog
        sg := c.recvq.dequeue()
        // 出队完毕，跳出循环
        if sg == nil {
            break
        }

        // 如果 elem 不为空，说明此 receiver 未忽略接收数据
        // 给它赋一个相应类型的零值
        if sg.elem != nil {
            typedmemclr(c.elemtype, sg.elem)
            sg.elem = nil
        }
        if sg.releasetime != 0 {
            sg.releasetime = cputicks()
        }
        // 取出 goroutine
        gp := sg.g
        gp.param = nil
        if raceenabled {
            raceacquireg(gp, unsafe.Pointer(c))
        }
        // 相连，形成链表
        gp.schedlink.set(glist)
        glist = gp
    }

    // 将 channel 等待发送队列里的 sudog 释放
    // 如果存在，这些 goroutine 将会 panic
    for {
        // 从发送队列里出队一个 sudog
        sg := c.sendq.dequeue()
        if sg == nil {
            break
        }

        // 发送者会 panic
        sg.elem = nil
        if sg.releasetime != 0 {
            sg.releasetime = cputicks()
        }
        gp := sg.g
        gp.param = nil
        if raceenabled {
            raceacquireg(gp, unsafe.Pointer(c))
        }
        // 形成链表
        gp.schedlink.set(glist)
        glist = gp
    }
    // 解锁
    unlock(&c.lock)

    // Ready all Gs now that we've dropped the channel lock.
    // 遍历链表
    for glist != nil {
        // 取最后一个
        gp := glist
        // 向前走一步，下一个唤醒的 g
        glist = glist.schedlink.ptr()
        gp.schedlink = 0
        // 唤醒相应 goroutine
        goready(gp, 3)
    }
}
```

close 逻辑比较简单，对于一个 channel，recvq 和 sendq 中分别保存了阻塞的发送者和接收者。关闭 channel 后，对于等待接收者而言，会收到一个相应类型的零值。对于等待发送者，会直接 panic。所以，在不了解 channel 还有没有接收者的情况下，不能贸然关闭 channel。

close 函数先上一把大锁，接着把所有挂在这个 channel 上的 sender 和 receiver 全都连成一个 sudog 链表，再解锁。最后，再将所有的 sudog 全都唤醒。

唤醒之后，该干嘛干嘛。sender 会继续执行 chansend 函数里 goparkunlock 函数之后的代码，很不幸，检测到 channel 已经关闭了，panic。receiver 则比较幸运，进行一些扫尾工作后，返回。这里，selected 返回 true，而返回值 received 则要根据 channel 是否关闭，返回不同的值。如果 channel 关闭，received 为 false，否则为 true。这我们分析的这种情况下，received 返回 false。

主要工作就是将 `recvq` 和 `sendq` 两个队列中的数据加入到 Goroutine 列表 `gList` 中，与此同时该函数会清除所有 `sudog` 上未被处理的元素。

该函数在最后会为所有被阻塞的 Goroutine 调用 [`runtime.goready`](https://github.com/golang/go/blob/64c22b70bf00e15615bb17c29f808b55bc339682/src/runtime/proc.go#L313) 触发调度。

## 总结

### 1、无缓存channel**总是**采用**直接发送接收**的形式：

1. 存在读等待的接收者，发送者直接将数据从自己栈拷贝到接收者的栈上
2. 存在写等待的发送者，接收者直接从sudog中接收数据

### 2、简单与性能取舍:

复杂了代码，但是提高了性能

# select

## 两大特点：

1. `select` 能在 Channel 上进行非阻塞的收发操作；
2. `select` 在遇到多个 Channel 同时响应时会**随机**挑选 `case` 执行；

## 数据结构

```go
//select 控制结构中的 case使用 runtime.scase 结构体来表示
type scase struct {
	c           *hchan
	elem        unsafe.Pointer //接收或者发送数据的变量地址
	kind        uint16
	pc          uintptr
	releasetime int64
}
//kind 表示 runtime.scase 的种类，总共包含以下四种：
const (
	caseNil = iota
	caseRecv
	caseSend
	caseDefault
)
```

## 原理

`select` 语句在编译期间会被转换成 `OSELECT` 节点。每一个 `OSELECT` 节点都会持有一组 `OCASE` 节点，如果 `OCASE` 的执行条件是空，那就意味着这是一个 `default` 节点:

![golang-oselect-and-ocases](https://img.draveness.me/2020-01-18-15793463657473-golang-oselect-and-ocases.png)

​                                 **图 5-7 OSELECT 和多个 OCASE**

四中情况：

1. `select` 不存在任何的 `case`；
2. `select` 只存在一个 `case`；
3. `select` 存在两个 `case`，其中一个 `case` 是 `default`；
4. `select` 存在多个 `case`；

### 直接阻塞

`select` 不存在任何的 `case`

`select {}` 的空语句转换成调用 [`runtime.block`](https://github.com/golang/go/blob/c112289ee4141ebc31db50328c355b01278b987b/src/runtime/select.go#L104-L106) 函数：

```go
func block() {
	gopark(nil, nil, waitReasonSelectNoCases, traceEvGoStop, 1)
}
```

[`runtime.block`](https://github.com/golang/go/blob/c112289ee4141ebc31db50328c355b01278b987b/src/runtime/select.go#L104-L106) 函数的实现非常简单，它会调用 runtime.gopark 让出当前 Goroutine 对处理器的使用权，传入的等待原因是 `waitReasonSelectNoCases`。

简单总结一下，空的 `select` 语句会直接阻塞当前的 Goroutine，导致 Goroutine 进入**无法被唤醒的永久休眠状态**。

### 单一管道

`select` 只存在一个 `case`

```go
// 改写前
select {
case v, ok <-ch: // case ch <- v
    ...    
}

// 改写后
if ch == nil {
    block()
}
v, ok := <-ch // case ch <- v
...
```

[`cmd/compile/internal/gc.walkselectcases`](https://github.com/golang/go/blob/c729116332ffb66a21dd587e3ee003cb8d0b16fe/src/cmd/compile/internal/gc/select.go#L108-L370) 在处理单操作 `select` 语句时，会根据 Channel 的收发情况生成不同的语句。当 `case` 中的 Channel 是空指针时，就会直接挂起当前 Goroutine 并永久休眠。

如果channel不是空指针，阻塞等待被唤醒

### 非阻塞操作

`select` 存在两个 `case`，其中一个 `case` 是 `default`

#### 发送

```go
select {
case ch <- i:
    ...
default:
    ...
}

if selectnbsend(ch, i) {
    ...
} else {
    ...
}

func selectnbsend(c *hchan, elem unsafe.Pointer) (selected bool) {
	return chansend(c, elem, false, getcallerpc())
}
//不存在接收方或者缓冲区空间不足都不会阻塞当前 Goroutine 而是会直接返回
```

#### 接收

```go
// 改写前
select {
case v <- ch: // case v, ok <- ch:
    ......
default:
    ......
}

// 改写后
if selectnbrecv(&v, ch) { // if selectnbrecv2(&v, &ok, ch) {
    ...
} else {
    ...
}

func selectnbrecv(elem unsafe.Pointer, c *hchan) (selected bool) {
	selected, _ = chanrecv(c, elem, false)
	return
}

func selectnbrecv2(elem unsafe.Pointer, received *bool, c *hchan) (selected bool) {
	selected, *received = chanrecv(c, elem, false)
	return
}
```

### 处理流程

在默认的情况下，编译器会使用如下的流程处理 `select` 语句：

1. 将所有的 `case` 转换成包含 Channel 以及类型等信息的 [`runtime.scase`](https://github.com/golang/go/blob/d1969015b4ac29be4f518b94817d3f525380639d/src/runtime/select.go#L28-L34) 结构体；

   ```go
   //select 控制结构中的 case使用 runtime.scase 结构体来表示
   type scase struct {
   c           *hchan
   elem        unsafe.Pointer //接收或者发送数据的变量地址
   kind        uint16
   pc          uintptr
   releasetime int64
   }
   ```

2. 调用运行时函数 [`runtime.selectgo`](https://github.com/golang/go/blob/d1969015b4ac29be4f518b94817d3f525380639d/src/runtime/select.go#L118-L497) 从多个准备就绪的 Channel 中选择一个可执行的 [`runtime.scase`](https://github.com/golang/go/blob/d1969015b4ac29be4f518b94817d3f525380639d/src/runtime/select.go#L28-L34) 结构体；

3. 通过 `for` 循环生成一组 `if` 语句，在语句中判断自己是不是被选中的 `case`

```go
selv := [3]scase{}
order := [6]uint16
for i, cas := range cases {
    c := scase{}
    c.kind = ...
    c.elem = ...
    c.c = ...
}
chosen, revcOK := selectgo(selv, order, 3)
if chosen == 0 {
    ...
    break
}
if chosen == 1 {
    ...
    break
}
if chosen == 2 {
    ...
    break
}
```

[`runtime.selectgo`](https://github.com/golang/go/blob/d1969015b4ac29be4f518b94817d3f525380639d/src/runtime/select.go#L118-L497) 这里分两部分分析它的执行过程：

1. 执行一些必要的初始化操作并确定 `case` 的处理顺序；
2. 处理流程
   在默认的情况下，编译器会使用如下的流程处理 select 语句：在循环中根据 `case` 的类型做出不同的处理；

#### 初始化

[`runtime.selectgo`](https://github.com/golang/go/blob/d1969015b4ac29be4f518b94817d3f525380639d/src/runtime/select.go#L118-L497) 函数首先会进行执行必要的初始化操作并决定处理 `case` 的两个顺序 — 轮询顺序 `pollOrder` 和加锁顺序 `lockOrder`

```go
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
	cas1 := (*[1 << 16]scase)(unsafe.Pointer(cas0))
	order1 := (*[1 << 17]uint16)(unsafe.Pointer(order0))
	
	scases := cas1[:ncases:ncases]
	pollorder := order1[:ncases:ncases]
	lockorder := order1[ncases:][:ncases:ncases]
	for i := range scases {
		cas := &scases[i]
	}

	for i := 1; i < ncases; i++ {
		j := fastrandn(uint32(i + 1))
		pollorder[i] = pollorder[j]
		pollorder[j] = uint16(i)
	}

	// 根据 Channel 的地址排序确定加锁顺序
	...
	sellock(scases, lockorder)
  。。。
}
```

轮询顺序 `pollOrder` 和加锁顺序 `lockOrder` 分别是通过以下的方式确认的：

- 轮询顺序：通过 runtime.fastrandn函数引入随机性；
- 加锁顺序：按照 Channel 的地址排序后确定加锁顺序；

随机的轮询顺序可以避免 Channel 的饥饿问题，保证公平性；而根据 Channel 的地址顺序确定加锁顺序能够避免死锁的发生。这段代码最后调用的 runtime.sellock 函数会按照之前生成的加锁顺序锁定 `select` 语句中包含所有的 Channel。

#### 主循环

当我们为 `select` 语句锁定了所有 Channel 之后就会进入 [`runtime.selectgo`](https://github.com/golang/go/blob/d1969015b4ac29be4f518b94817d3f525380639d/src/runtime/select.go#L118-L497) 函数的主循环，它会分三个阶段查找或者等待某个 Channel 准备就绪：

1. 查找是否已经存在准备就绪的 Channel，即可以执行收发操作；
2. 将当前 Goroutine 加入 Channel 对应的收发队列上并等待其他 Goroutine 的唤醒；
3. 当前 Goroutine 被唤醒之后找到满足条件的 Channel 并进行处理；

第一阶段：（收发）

![golang-runtime-selectgo](https://img.draveness.me/2020-01-18-15793463657488-golang-runtime-selectgo.png)

​                                   **运行时 selectgo 函数**

第二阶段：（阻塞）

除了将当前 Goroutine 对应的 [`runtime.sudog`](https://github.com/golang/go/blob/cfe3cd903f018dec3cb5997d53b1744df4e53909/src/runtime/runtime2.go#L342-L368) 结构体加入队列之外，这些 [`runtime.sudog`](https://github.com/golang/go/blob/cfe3cd903f018dec3cb5997d53b1744df4e53909/src/runtime/runtime2.go#L342-L368) 结构体都会被串成链表附着在 Goroutine 上。在入队之后会调用 [`runtime.gopark`](https://github.com/golang/go/blob/c112289ee4141ebc31db50328c355b01278b987b/src/runtime/proc.go#L287-L305) 函数挂起当前 Goroutine 等待调度器的唤醒。

![Golang-Select-Waiting](https://img.draveness.me/2020-01-19-15794018429558-Golang-Select-Waiting.png)

​                                      **Goroutine 上等待收发的 sudog 链表**

第三阶段：（唤醒）

等到 `select` 中的一些 Channel 准备就绪之后，当前 Goroutine 就会被调度器唤醒。这时会继续执行 [`runtime.selectgo`](https://github.com/golang/go/blob/d1969015b4ac29be4f518b94817d3f525380639d/src/runtime/select.go#L118-L497) 函数的第三阶段，从 [`runtime.sudog`](https://github.com/golang/go/blob/cfe3cd903f018dec3cb5997d53b1744df4e53909/src/runtime/runtime2.go#L342-L368) 结构体中获取数据

第三次遍历全部 `case` 时，我们会先获取当前 Goroutine 接收到的参数 `sudog` 结构，我们会依次对比所有 `case` 对应的 `sudog` 结构找到被唤醒的 `case`，获取该 `case` 对应的索引并返回。

由于当前的 `select` 结构找到了一个 `case` 执行，那么剩下 `case` 中没有被用到的 `sudog` 就会被忽略并且释放掉。为了不影响 Channel 的正常使用，我们还是需要将这些废弃的 `sudog` 从 Channel 中出队。

## 总结

我们简单总结一下 `select` 结构的执行过程与实现原理，首先在编译期间，Go 语言会对 `select` 语句进行优化，它会根据 `select` 中 `case` 的不同选择不同的优化路径：

1. 空的 `select` 语句会被转换成 [`runtime.block`](https://github.com/golang/go/blob/c112289ee4141ebc31db50328c355b01278b987b/src/runtime/select.go#L104-L106) 函数的调用，直接挂起当前 Goroutine；

2. 如果select语句中只包含一个case，就会被转换成 if ch == nil { block }; n;

    表达式；

   - 首先判断操作的 Channel 是不是空的；
   - 然后执行 `case` 结构中的内容；

3. 如果 `select` 语句中只包含两个 `case` 并且其中一个是 `default`，那么会使用 [`runtime.selectnbrecv`](https://github.com/golang/go/blob/d1969015b4ac29be4f518b94817d3f525380639d/src/runtime/chan.go#L683-L686) 和 [`runtime.selectnbsend`](https://github.com/golang/go/blob/d1969015b4ac29be4f518b94817d3f525380639d/src/runtime/chan.go#L662-L664) 非阻塞地执行收发操作；

4. 在默认情况下会通过 [`runtime.selectgo`](https://github.com/golang/go/blob/d1969015b4ac29be4f518b94817d3f525380639d/src/runtime/select.go#L118-L497) 函数获取执行 `case` 的索引，并通过多个 `if` 语句执行对应 `case` 中的代码；

在编译器已经对 `select` 语句进行优化之后，Go 语言会在运行时执行编译期间展开的 [`runtime.selectgo`](https://github.com/golang/go/blob/d1969015b4ac29be4f518b94817d3f525380639d/src/runtime/select.go#L118-L497) 函数，该函数会按照以下的流程执行：

1. 随机生成一个遍历的轮询顺序 `pollOrder` 并根据 Channel 地址生成锁定顺序 `lockOrder`；
2. 根据 pollOrder 遍历所有的case查看是否有可以立刻处理的 Channel；
   1. 如果存在就直接获取 `case` 对应的索引并返回；
   2. 如果不存在就会创建 [`runtime.sudog`](https://github.com/golang/go/blob/cfe3cd903f018dec3cb5997d53b1744df4e53909/src/runtime/runtime2.go#L342-L368) 结构体，将当前 Goroutine 加入到所有相关 Channel 的收发队列，并调用 [`runtime.gopark`](https://github.com/golang/go/blob/c112289ee4141ebc31db50328c355b01278b987b/src/runtime/proc.go#L287-L305) 挂起当前 Goroutine 等待调度器的唤醒；
3. 当调度器唤醒当前 Goroutine 时就会再次按照 `lockOrder` 遍历所有的 `case`，从中查找需要被处理的 [`runtime.sudog`](https://github.com/golang/go/blob/cfe3cd903f018dec3cb5997d53b1744df4e53909/src/runtime/runtime2.go#L342-L368) 结构对应的索引；

`select` 关键字是 Go 语言特有的控制结构，它的实现原理比较复杂，需要编译器和运行时函数的通力合作

## 个人理解

select语句中加入default关键字，使得select支持非阻塞收发，调用如下两个收发函数：

```go
func selectnbsend(c *hchan, elem unsafe.Pointer) (selected bool) {
	return chansend(c, elem, false, getcallerpc())
}

func selectnbrecv(elem unsafe.Pointer, c *hchan) (selected bool) {
	selected, _ = chanrecv(c, elem, false)
	return
}
```

# 相关问题

## Go的两种并发形式

1、多线程共享内存

其实就是Java或者C++等语言中的多线程开发

2、CSP并发模型

Go语言特有的

## 什么是CSP？

不要以共享内存的方式来通信，相反，要通过通信来共享内存。

CSP 是 Communicating Sequential Process 的简称，中文可以叫做通信顺序进程，是一种并发编程模型，由 [Tony Hoare](https://en.wikipedia.org/wiki/Tony_Hoare) 于 1977 年提出。简单来说，CSP 模型由并发执行的实体（线程或者进程）所组成，实体之间通过发送消息进行通信，这里发送消息时使用的就是通道，或者叫 channel。CSP 模型的关键是关注 channel，而不关注发送消息的实体。Go 语言实现了 CSP 部分理论。

Go的CSP并发模型，是通过goroutine和channel来实现的。

goroutine 是Go语言中并发的执行单位。

channel是Go语言中各个并发结构体(goroutine)之前的通信机制。 通俗的讲，就是各个goroutine之间通信的”管道“，有点类似于Linux中的管道。

**Channel**

> Golang中使用 CSP中 channel 这个概念。channel 是被单独创建并且可以在进程之间传递，它的通信模式类似于 boss-worker 模式的，一个实体通过将消息发送到channel 中，然后又监听这个 channel 的实体处理，两个实体之间是匿名的，这个就实现实体中间的解耦，其中 channel 是同步的一个消息被发送到 channel 中，最终是一定要被另外的实体消费掉的。

**Goroutine**

> Goroutine 是实际并发执行的实体，它底层是使用协程(coroutine)实现并发，coroutine是一种运行在用户态的用户线程，类似于 greenthread，go底层选择使用coroutine的出发点是因为，它具有以下特点：
>
> 用户空间 避免了内核态和用户态的切换导致的成本
>
> 可以由语言和框架层进行调度
>
> 更小的栈空间允许创建大量的实例
>
>  用户空间线程的调度不是由操作系统来完成的，像在java 1.3中使用的greenthread的是由JVM统一调度的(后java已经改为内核线程)，还有在ruby中的fiber(半协程) 是需要在重新中自己进行调度的，而goroutine是在golang层面提供了调度器，并且对网络IO库进行了封装，屏蔽了复杂的细节，对外提供统一的语法关键字支持，简化了并发程序编写的成本。

## 进程、线程和协程的区别？

进程拥有代码和打开的文件资源、数据资源、独立的内存空间。

线程从属于进程，是程序的实际执行者。一个进程至少包含一个主线程，也可以有更多的子线程。线程拥有自己的栈空间。

同一进程的线程共享本进程的地址空间和资源，而进程之间的地址空间和资源是相互独立的

**对操作系统来说，线程是最小的执行单元，进程是最小的资源管理单元。**

无论进程还是线程，都是由操作系统所管理的。

![img](https://upload-images.jianshu.io/upload_images/1604966-2a3a741343c0d16c.png?imageMogr2/auto-orient/strip|imageView2/2/w/724/format/webp)

线程的五种状态：

![img](https://upload-images.jianshu.io/upload_images/1604966-0e8f701ed9a0eb20.png?imageMogr2/auto-orient/strip|imageView2/2/w/836/format/webp)

正如一个进程可以拥有多个线程一样，一个线程也可以拥有多个协程。协程的调度完全由用户控制。协程拥有自己的寄存器上下文和栈。协程调度切换时，将寄存器上下文和栈保存到其他地方，在切回来的时候，恢复先前保存的寄存器上下文和栈，直接操作栈则基本没有内核切换的开销，可以不加锁的访问全局变量，所以上下文的切换非常快。***协程与线程主要区别是它将不再被内核调度，而是交给了程序自己而线程是将自己交给内核调度，所以也不难理解golang中调度器的存在。***![img](https://upload-images.jianshu.io/upload_images/1604966-1e45c5e197f5b28c.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/724/format/webp)

最重要的是，**协程不是被操作系统内核所管理，而完全是由程序所控制（也就是在用户态执行）**。

这样带来的好处就是性能得到了很大的提升，不会像线程切换那样消耗资源。

### 进程与线程的区别总结

线程具有许多传统进程所具有的特征，故又称为轻型进程(Light—Weight Process)或进程元；而把传统的进程称为重型进程(Heavy—Weight Process)，它相当于只有一个线程的任务。在引入了线程的操作系统中，通常一个进程都有若干个线程，至少包含一个线程。

**根本区别**：进程是操作系统资源分配的基本单位，而线程是处理器任务调度和执行的基本单位

**资源开销**：每个进程都有独立的代码和数据空间（程序上下文），程序之间的切换会有较大的开销；线程可以看做轻量级的进程，同一类线程共享代码和数据空间，每个线程都有自己独立的运行栈和程序计数器（PC），线程之间切换的开销小。

**包含关系**：如果一个进程内有多个线程，则执行过程不是一条线的，而是多条线（线程）共同完成的；线程是进程的一部分，所以线程也被称为轻权进程或者轻量级进程。

**内存分配**：同一进程的线程共享本进程的地址空间和资源，而进程之间的地址空间和资源是相互独立的

**影响关系**：一个进程崩溃后，在保护模式下不会对其他进程产生影响，但是一个线程崩溃整个进程都死掉。所以多进程要比多线程健壮。

**执行过程**：每个独立的进程有程序运行的入口、顺序执行序列和程序出口。但是线程不能独立执行，必须依存在应用程序中，由应用程序提供多个线程执行控制，两者均可并发执行

##  从一个关闭的 channel 仍然能读出数据吗？

从一个有缓冲的 channel 里读数据，当 channel 被关闭，依然能读出有效值。只有当返回的 ok 为 false 时，读出的数据才是无效的。

## channel的使用场景？

### 退出信号

```go
go func (t *task)run{
  //任务主循环
  for{
    ...
		select {
    //收到关闭信号，退出循环
		case <-t.closeCh:
			break Loop
		case <-timer.C:
		}
	}
  //关闭前准备工作，如设置下任务状态
  t.setError(vpsapi.TaskStatus_FINISHED, nil)
  t.logger.Info("task ended")
  close(t.doneCh)
}

func(t *task)close()error{
  ...
  //我要关闭该任务了
  close(t.closeCh)
  timer := time.NewTimer(20 * time.Second)
	defer timer.Stop()
	select {
  //等待任务关闭完成
	case <-t.doneCh:
  //超时退出
	case <-timer.C:
		t.logger.Warnf("loop not exit after 20 seconds")
	}
}
```

### 数据传输

```go
//tracket-->analyser
func (t *TrackExecutor) writeResult(ctx *TrackContext, out motanalyser.Tracklet) bool {
	written := false
	if t.offline {
		timer := time.NewTimer(pcommon.DefaultOfflineChannelWriteTimeout)
		select {
		case ctx.outputCh <- out:
			written = true
		case <-timer.C:
		}
		timer.Stop()
	} else {
		select {
    //向有缓冲channel写入结果
		case ctx.outputCh <- out:
			written = true
		default:
		}
	}
}
```



### 解耦生产方和消费方

```go
func main() {
    taskCh := make(chan int, 100)
    go worker(taskCh)

    // 塞任务
    for i := 0; i < 10; i++ {
        taskCh <- i
    }

    // 等待 1 小时 
    select {
    case <-time.After(time.Hour):
    }
}

func worker(taskCh <-chan int) {
    const N = 5
    // 启动 5 个工作协程
    for i := 0; i < N; i++ {
        go func(id int) {
            for {
                task := <- taskCh
                fmt.Printf("finish task: %d by worker %d\n", task, id)
                time.Sleep(time.Second)
            }
        }(i)
    }
}
//5 个工作协程在不断地从工作队列里取任务，生产方只管往 channel 发送任务即可，解耦生产方和消费方。
```

### 控制并发数

有时需要定时执行几百个任务，例如每天定时按城市来执行一些离线计算的任务。但是并发数又不能太高，因为任务执行过程依赖第三方的一些资源，对请求的速率有限制。这时就可以通过 channel 来控制并发数。

```go
var limit = make(chan int, 3)

func main() {
    // …………
    for _, w := range work {
        go func() {
            limit <- 1
            w()
            <-limit
        }()
    }
    // …………
}
//控制同时运行的 goroutine 数最多3个。
//还有一点要注意的是，如果 w() 发生 panic，那“许可证”可能就还不回去了，因此需要使用 defer 来保证。
```

## 如何优雅的关闭channel？

> don't close a channel from the receiver side and don't close a channel if the channel has multiple concurrent senders.
>
> don't close (or send values to) closed channels.

根据 sender 和 receiver 的个数，分下面几种情况：

1. 一个 sender，一个 receiver

2. 一个 sender， M 个 receiver

   直接从sender端关闭

   以下两种不调用close主动关闭channel，等待GC来关闭

3. N 个 sender，一个 reciver

   一个传递关闭信号的 channel，receiver 通过信号 channel 下达关闭数据 channel 指令。senders 监听到关闭信号后，停止发送数据

```go
func main() {
    rand.Seed(time.Now().UnixNano())

    const Max = 100000
    const NumSenders = 1000

    dataCh := make(chan int, 100)
    stopCh := make(chan struct{})

    // senders
    for i := 0; i < NumSenders; i++ {
        go func() {
            for {
                select {
                case <- stopCh:
                    return
                case dataCh <- rand.Intn(Max):
                }
            }
        }()
    }

    // the receiver
    go func() {
        for value := range dataCh {
            if value == Max-1 {
                fmt.Println("send stop signal to senders.")
                close(stopCh)
                return
            }

            fmt.Println(value)
        }
    }()

    select {
    case <- time.After(time.Hour):
    }
}
//在 Go 语言中，对于一个 channel，如果最终没有任何 goroutine 引用它，不管 channel 有没有被关闭，最终都会被 gc 回收。所以，在这种情形下，所谓的优雅地关闭 channel 就是不关闭 channel，让 gc 代劳
```

​       4.N 个 sender， M 个 receiver

```go
func main() {
    rand.Seed(time.Now().UnixNano())

    const Max = 100000
    const NumReceivers = 10
    const NumSenders = 1000

    dataCh := make(chan int, 100)
    stopCh := make(chan struct{})

    // It must be a buffered channel.
    toStop := make(chan string, 1)

    var stoppedBy string

    // moderator
    go func() {
        stoppedBy = <-toStop
        close(stopCh)
    }()

    // senders
    for i := 0; i < NumSenders; i++ {
        go func(id string) {
            for {
                value := rand.Intn(Max)
                if value == 0 {
                    select {
                    case toStop <- "sender#" + id:
                    default:
                    }
                    return
                }

                select {
                case <- stopCh:
                    return
                case dataCh <- value:
                }
            }
        }(strconv.Itoa(i))
    }

    // receivers
    for i := 0; i < NumReceivers; i++ {
        go func(id string) {
            for {
                select {
                case <- stopCh:
                    return
                case value := <-dataCh:
                    if value == Max-1 {
                        select {
                        case toStop <- "receiver#" + id:
                        default:
                        }
                        return
                    }

                    fmt.Println(value)
                }
            }
        }(strconv.Itoa(i))
    }

    select {
    case <- time.After(time.Hour):
    }
}
//代码里 toStop 就是中间人的角色，使用它来接收 senders 和 receivers 发送过来的关闭 dataCh 请求。
//这里将 toStop 声明成了一个 缓冲型的 channel。假设 toStop 声明的是一个非缓冲型的 channel，那么第一个发送的关闭 dataCh 请求可能会丢失。因为无论是 sender 还是 receiver 都是通过 select 语句来发送请求，如果中间人所在的 goroutine 没有准备好，那 select 语句就不会选中，直接走 default 选项，什么也不做。这样，第一个关闭 dataCh 的请求就会丢失。
//如果，我们把 toStop 的容量声明成 Num(senders) + Num(receivers)，那发送 dataCh 请求的部分可以改成更简洁的形式：
...
toStop := make(chan string, NumReceivers + NumSenders)
...
            value := rand.Intn(Max)
            if value == 0 {
                toStop <- "sender#" + id
                return
            }
...
                if value == Max-1 {
                    toStop <- "receiver#" + id
                    return
                }
...
//直接向 toStop 发送请求，因为 toStop 容量足够大，所以不用担心阻塞，自然也就不用 select 语句再加一个 default case 来避免阻塞。
```

## channel 在什么情况下会引起资源泄漏？

泄漏的原因是 goroutine 操作 channel 后，处于发送或接收阻塞状态，而 channel 处于满或空的状态，一直得不到改变。同时，垃圾回收器也不会回收此类资源，进而导致 gouroutine 会一直处于等待队列中，不见天日。

另外，程序运行过程中，对于一个 channel，如果没有任何 goroutine 引用了，gc 会对其进行回收操作，不会引起内存泄漏。

## channel 发送和接收元素的本质是什么？

Channel 发送和接收元素的本质是什么？

> All transfer of value on the go channels happens with the copy of value.

就是说 channel 的发送和接收操作本质上都是 “值的拷贝”，无论是从 sender goroutine 的栈到 chan buf，还是从 chan buf 到 receiver goroutine，或者是直接从 sender goroutine 到 receiver goroutine

## 操作channel的情况

总结一下操作 channel 的结果：

| 操作     | nil channel | closed channel     | not nil, not closed channel                                  |
| -------- | ----------- | ------------------ | ------------------------------------------------------------ |
| close    | panic       | panic              | 正常关闭                                                     |
| 读 <- ch | 阻塞        | 读到对应类型的零值 | 阻塞或正常读取数据。缓冲型 channel 为空或非缓冲型 channel 没有等待发送者时会阻塞 |
| 写 ch <- | 阻塞        | panic              | 阻塞或正常写入数据。非缓冲型 channel 没有等待接收者或缓冲型 channel buf 满时会被阻塞 |

总结一下，发生 panic 的情况有三种：向一个关闭的 channel 进行写操作；关闭一个 nil 的 channel；重复关闭一个 channel。

读、写一个 nil channel 都会被阻塞。**如果为非阻塞收发，即收发操作为select的一个case（且存在default分支），不会被阻塞，直接返回。**

## 关于 channel 的 happened-before 有哪些?

关于 channel 的发送（send）、发送完成（send finished）、接收（receive）、接收完成（receive finished）的 happened-before 关系如下：

1. 第 n 个 `send` 一定 `happened before` 第 n 个 `receive finished`，无论是缓冲型还是非缓冲型的 channel。
2. 对于容量为 m 的缓冲型 channel，第 n 个 `receive` 一定 `happened before` 第 n+m 个 `send finished`。
3. 对于非缓冲型的 channel，第 n 个 `receive` 一定 `happened before` 第 n 个 `send finished`。
4. channel close 一定 `happened before` receiver 得到通知。

```go
var done = make(chan bool)
var msg string

func aGoroutine() {
    msg = "hello, world"
    done <- true
}

func main() {
    go aGoroutine()
    <-done
    println(msg)
}
//这里依赖的 happened before 就是前面讲的第一条。第一个 send 一定 happened before 第一个 receive finished，即 done <- true 先于 <-done 发生，这意味着 main 函数里执行完 <-done 后接着执行 println(msg) 这一行代码时，msg 已经被赋过值了，所以会打印出想要的结果
```

```go
var done = make(chan bool)
var msg string

func aGoroutine() {
    msg = "hello, world"
    <-done
}

func main() {
    go aGoroutine()
    done <- true
    println(msg)
}
//根据第三条规则，对于非缓冲型的 channel，第一个 receive 一定 happened before 第一个 send finished。也就是说， 在 done <- true 完成之前，<-done 就已经发生了，也就意味着 msg 已经被赋上值了，最终也会打印出 hello, world。
```

