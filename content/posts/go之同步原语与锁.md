---
title: "go之同步原语"
date: 2022-06-26T21:35:03+08:00
draft: false
author: "gq"
---

# 同步原语与锁

## Mutex

### 互斥锁的演变史

![](https://s2.loli.net/2022/06/10/IgzVObeT4rif6qc.png)

![](https://s2.loli.net/2022/06/13/j5gEfVOwHranRyF.png)

#### 初版

```go
    // CAS操作，当时还没有抽象出atomic包
    func cas(val *int32, old, new int32) bool
    func semacquire(*int32)
    func semrelease(*int32)
    // 互斥锁的结构，包含两个字段
    type Mutex struct {
        key  int32 // 锁是否被持有的标识
        sema int32 // 信号量专用，用以阻塞/唤醒goroutine
    }
    
    // 保证成功在val上增加delta的值
    func xadd(val *int32, delta int32) (new int32) {
        for {
            v := *val
            if cas(val, v, v+delta) {
                return v + delta
            }
        }
        panic("unreached")
    }
    
    // 请求锁
    func (m *Mutex) Lock() {
        if xadd(&m.key, 1) == 1 { //标识加1，如果等于1，成功获取到锁
            return
        }
        semacquire(&m.sema) // 否则阻塞等待
    }
    
    func (m *Mutex) Unlock() {
        if xadd(&m.key, -1) == 0 { // 将标识减去1，如果等于0，则没有其它等待者
            return
        }
        semrelease(&m.sema) // 唤醒其它阻塞的goroutine
    }
```

存在的问题：

如果我们能够把锁交给正在占用 CPU 时间片的 goroutine 的话，那就不需要做上下文的切换，在高并发的情况下，可能会有更好的性能

#### 给新人机会



### 数据结构

```go
type Mutex struct {
	state int32  //表示当前互斥锁的状态
	sema  uint32 //用于控制锁状态的信号量
}
```

Mutex 的零值是还没有 goroutine 等待的未加锁的状态，所以你不需要额外的初始化，直接声明变量（如 var mu sync.Mutex）即可

### 互斥锁的state

互斥锁的状态比较复杂，如下图所示，最低三位分别表示 `mutexLocked`、`mutexWoken` 和 `mutexStarving`，剩下的位置用来表示当前有多少个 Goroutine 等待互斥锁的释放：

![golang-mutex-state](https://img.draveness.me/2020-01-23-15797104328010-golang-mutex-state.png)

**图 6-6 互斥锁的状态**

在默认情况下，互斥锁的所有状态位都是 `0`，`int32` 中的不同位分别表示了不同的状态：

- `mutexLocked` — 表示互斥锁的锁定状态；
- `mutexWoken` — 表示从正常模式被从唤醒；
- `mutexStarving` — 当前的互斥锁进入饥饿状态；

### 互斥锁的两种模式

#### 正常模式

在正常模式下，锁的waiters Goroutine会按照**先进先出**的顺序获取锁。但是刚被唤起的 Goroutine 与新创建的 Goroutine 竞争时，大概率会获取不到锁（因为新创建的Goroutine正在占用 CPU 时间片），为了减少这种情况的出现，一旦 Goroutine 超过 1ms 没有获取到锁，它就会将当前互斥锁切换饥饿模式，防止部分 Goroutine 被饿死。

#### 饥饿模式

在饥饿模式中，互斥锁会直接交给等待队列最前面的 Goroutine。新的 Goroutine 在该状态下不能获取锁、也不会进入自旋状态，它们只会在队列的末尾等待。如果一个 Goroutine 获得了互斥锁并且它在队列的末尾或者它等待的时间少于 1ms，那么当前的互斥锁就会被切换回正常模式。

相比于饥饿模式，正常模式下的互斥锁能够提供更好地性能，饥饿模式的能避免 Goroutine 由于陷入等待无法获取锁而造成的高尾延时。

### CAS

#### 什么是CAS？

`CAS`即`Compare And Swap`的缩写，翻译成中文就是**比较并交换**，其作用是让CPU比较内存中某个值是否和预期的值相同，如果相同则将这个值更新为新值，不相同则不做更新，也就是CAS是**原子性**的操作(读和写两者同时具有原子性)，其实现方式是通过借助`C/C++`调用CPU指令完成的，所以效率很高。

```go
atomic.CompareAndSwapInt32(&m.state, old, new)
//更新m.state字段之前先比较m.state和old是否相等，如果相等，new写入m.state,函数返回true；如果不相等，返回函数false。
//CAS机制中的这些步骤是原子性的（从指令层面提供的原子操作），所以CAS机制可以解决多线程并发编程对共享变量读写的原子性问题
```

我们知道`CAS`操作并不会锁住共享变量，也就是一种**非阻塞**的同步机制，`CAS`就是乐观锁的实现。

1. 乐观锁 **乐观锁**总是假设最好的情况，每次去操作数据都认为不会被别的线程修改数据，**所以在每次操作数据的时候都不会给数据加锁**，即在线程对数据进行操作的时候，**别的线程不会阻塞**仍然可以对数据进行操作，只有在需要更新数据的时候才会去判断数据是否被别的线程修改过，如果数据被修改过则会拒绝操作并且返回错误信息给用户。
2. 悲观锁 **悲观锁**总是假设最坏的情况，每次去操作数据时候都认为会被的线程修改数据，**所以在每次操作数据的时候都会给数据加锁**，让别的线程无法操作这个数据，别的线程会一直阻塞直到获取到这个数据的锁。这样的话就会影响效率，比如当有个线程发生一个很耗时的操作的时候，别的线程只是想获取这个数据的值而已都要等待很久。

#### CAS的缺点

1. ABA问题

   在多线程场景下`CAS`会出现`ABA`问题，关于ABA问题这里简单科普下，例如有2个线程同时对同一个值(初始值为A)进行CAS操作，这三个线程如下

   1. 线程1，期望值为A，欲更新的值为B
   2. 线程2，期望值为A，欲更新的值为B

   线程`1`抢先获得CPU时间片，而线程`2`因为其他原因阻塞了，线程`1`取值与期望的A值比较，发现相等然后将值更新为B，然后这个时候**出现了线程`3`，期望值为B，欲更新的值为A**，线程3取值与期望的值B比较，发现相等则将值更新为A，此时线程`2`从阻塞中恢复，并且获得了CPU时间片，这时候线程`2`取值与期望的值A比较，发现相等则将值更新为B，虽然线程`2`也完成了操作，但是线程`2`并不知道值已经经过了`A->B->A`的变化过程。

   **`ABA`问题带来的危害**：
    小明在提款机，提取了50元，因为提款机问题，有两个线程，同时把余额从100变为50
    线程1（提款机）：获取当前值100，期望更新为50，
    线程2（提款机）：获取当前值100，期望更新为50，
    线程1成功执行，线程2某种原因block了，这时，某人给小明汇款50
    线程3（默认）：获取当前值50，期望更新为100，
    这时候线程3成功执行，余额变为100，
    线程2从Block中恢复，获取到的也是100，compare之后，继续更新余额为50！！！
    此时可以看到，实际余额应该为100（100-50+50），但是实际上变为了50（100-50+50-50）这就是ABA问题带来的成功提交。

   **解决方法**： 在变量前面加上版本号，每次变量更新的时候变量的**版本号都`+1`**，即`A->B->A`就变成了`1A->2B->3A`。

2. 可能会消耗较高的CPU
   看起来CAS比锁的效率高，**从阻塞机制变成了非阻塞机制**，减少了线程之间等待的时间。每个方法不能绝对的比另一个好，在线程之间竞争程度大的时候，如果使用CAS，每次都有很多的线程在**竞争**，也就是说CAS机制不能更新成功。这种情况下CAS机制会一直重试，这样就会比较耗费CPU。因此可以看出，如果线程之间竞争程度小，使用CAS是一个很好的选择；但是如果竞争很大，使用锁可能是个更好的选择。在并发量非常高的环境中，如果仍然想通过原子类来更新的话，可以使用AtomicLong的替代类：LongAdder。

   如果`CAS`操作失败，就需要循环进行`CAS`操作(循环同时将期望值更新为最新的)，如果长时间都不成功的话，那么会造成CPU极大的开销。

   > 这种循环也称为自旋

   **解决方法**： 限制自旋次数，防止进入死循环。

3. 只能保证一个共享变量的原子操作

   `CAS`的原子操作只能针对一个共享变量。

   **解决方法**： 如果需要对多个共享变量进行操作，可以使用加锁方式(悲观锁)保证原子性，或者可以把多个共享变量合并成一个共享变量进行`CAS`操作。

#### CAS的优点

可以保证变量操作的原子性；
并发量不是很高的情况下，使用CAS机制比使用锁机制效率更高；
在线程对共享资源占用时间较短的情况下，使用CAS机制效率也会较高。

### 自旋

```go
func sync_runtime_doSpin() {
	procyield(active_spin_cnt)
}

TEXT runtime·procyield(SB),NOSPLIT,$0-0
	MOVL	cycles+0(FP), AX
again:
	PAUSE
	SUBL	$1, AX
	JNZ	again
	RET
```

**自旋是一种多线程同步机制，当前的进程在进入自旋的过程中会一直保持 CPU 的占用，持续检查某个条件是否为真。在多核的 CPU 上，自旋可以避免 Goroutine 的切换，使用恰当会对性能带来很大的增益，但是使用的不恰当就会拖慢整个程序**.

####  Goroutine 进入自旋的条件

1. 互斥锁只有在普通模式才能进入自旋；
2. `sync.runtime_canSpin`需要返回true：
   1. 运行在多 CPU 的机器上；
   2. 当前 Goroutine 为了获取该锁进入自旋的次数小于四次；
   3. 当前机器上至少存在一个正在运行的处理器 P 并且处理的运行队列为空；

#### 为什么自旋时使用PAUSE指令？

当spinlock执行lock()获得锁失败后会进行busy loop，不断检测锁状态，尝试获得锁。这么做有一个缺陷：频繁的检测会让流水线上充满了读操作。另外一个线程往流水线上丢入一个锁变量写操作的时候，必须对流水线进行重排，因为CPU必须保证所有读操作读到正确的值。流水线重排十分耗时，影响lock()的性能。

为了解决这个问题，intel发明了pause指令。这个指令的本质功能：让加锁失败时cpu睡眠30个（about）clock，从而使得读操作的频率低很多。流水线重排的代价也会小很多。

Pause指令解释（from intel）：

Description

Improves the performance of spin-wait loops. When executing a “spin-wait loop,” a Pentium 4 or Intel Xeon processor suffers a severe performance penalty when exiting the loop because it detects a possible memory order violation. The PAUSE instruction provides a hint to the processor that the code sequence is a spin-wait loop. The processor uses this hint to avoid the memory order violation in most situations, which greatly improves processor performance. For this reason, it is recommended that a PAUSE instruction be placed in all spin-wait loops.

提升spin-wait-loop的性能，当执行spin-wait循环的时候，笨死和小强处理器会因为在退出循环的时候检测到memory order violation而导致严重的性能损失，pause指令就相当于提示处理器哥目前处于spin-wait中。在绝大多数情况下，处理器根据这个提示来避免violation，藉此大幅提高性能，由于这个原因，我们建议在spin-wait中加上一个pause指令。

名词解释(以下为本人猜想)：memory order violation，直译为-内存访问顺序冲突，当处理器在(out of order)乱序执行的流水线上去内存load某个内存地址的值(此处是lock)的时候，发现这个值正在被store，而且store本身就在load之前，对于处理器来说，这就是一个hazard，流水流不起来。

在本文中，具体是指当一个获得锁的工作线程W从临界区退出，在调用unlock释放锁的时候，有若干个等待线程S都在自旋检测锁是否可用，此时W线程会产生一个store指令，若干个S线程会产生很多load指令，在store之后的load指令要等待store在流水线上执行完毕才能执行，由于处理器是乱序执行，在没有store指令之前，处理器对多个没有依赖的load是可以随机乱序执行的，当有了store指令之后，需要reorder重新排序执行，此时会严重影响处理器性能，按照intel的说法，会带来25倍的性能损失。Pause指令的作用就是减少并行load的数量，从而减少reorder时所耗时间。

### 加锁和解锁

#### 加锁

互斥锁的加锁过程比较复杂，它涉及自旋、信号量以及调度等概念：

- 如果互斥锁处于初始化状态，就会直接通过置位 `mutexLocked` 加锁；
- 如果互斥锁处于 `mutexLocked` 并且在普通模式下工作，就会进入自旋，执行 30 次 `PAUSE` 指令消耗 CPU 时间等待锁的释放；
- 如果当前 Goroutine 等待锁的时间超过了 1ms，互斥锁就会切换到饥饿模式；
- 互斥锁在正常情况下会通过 [`sync.runtime_SemacquireMutex`](https://github.com/golang/go/blob/a4c579e8f7c8129b2c27779f206ebd2c9b393633/src/runtime/sema.go#L70-L72) 函数将尝试获取锁的 Goroutine 切换至休眠状态，等待锁的持有者唤醒当前 Goroutine；
- 如果当前 Goroutine 是互斥锁上的最后一个等待的协程或者等待的时间小于 1ms，当前 Goroutine 会将互斥锁切换回正常模式；

#### 解锁

- 当互斥锁已经被解锁时，那么调用 [`sync.Mutex.Unlock`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/mutex.go#L179-L192) 会直接抛出异常；
- 当互斥锁处于饥饿模式时，会直接将锁的所有权交给队列中的下一个等待者，等待者会负责设置 `mutexLocked` 标志位；
- 当互斥锁处于普通模式时，如果没有 Goroutine 等待锁的释放或者已经有被唤醒的 Goroutine 获得了锁，就会直接返回；在其他情况下会通过 [`sync.runtime_Semrelease`](https://github.com/golang/go/blob/a4c579e8f7c8129b2c27779f206ebd2c9b393633/src/runtime/sema.go#L65-L67) 唤醒对应的 Goroutine；

Unlock 方法可以被任意的 goroutine 调用释放锁，即使是没持有这个互斥锁的 goroutine，也可以进行这个操作。这是因为，Mutex 本身并没有包含持有这把锁的 goroutine 的信息，所以，Unlock 也不会对此进行检查。Mutex 的这个设计一直保持至今。

### 错误场景

Lock/Unlock 没有成对出现，就意味着会出现死锁的情况，或者是因为 Unlock 一个未加锁的 Mutex 而导致 panic。

Mutex 不是可重入的锁。

## RWMutex

![](https://s2.loli.net/2022/06/13/4sOd6nGlhQT1ZX5.png)

### 数据结构

```go
type RWMutex struct {
	w           Mutex //为 writer 的竞争锁而设计；
	writerSem   uint32 //都是为了阻塞设计的信号量。
	readerSem   uint32 //都是为了阻塞设计的信号量。
	readerCount int32 //记录当前 reader 的数量（以及是否有 writer 竞争锁）；
	readerWait  int32 //记录 writer 请求锁时需要等待 read 完成的 reader 的数量；
}
```

### Rlock

```go
func (rw *RWMutex) RLock() {
	if atomic.AddInt32(&rw.readerCount, 1) < 0 {
    // rw.readerCount是负值的时候，意味着此时有writer等待请求锁，因为writer优先级高，所以把后来的reader阻塞休眠
		runtime_SemacquireMutex(&rw.readerSem, false, 0)
	}
}
```

### RUnlock

```go
func (rw *RWMutex) RUnlock() {
	if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
    // 有等待的writer
		rw.rUnlockSlow(r)
	}
}
func (rw *RWMutex) rUnlockSlow(r int32) {
  if atomic.AddInt32(&rw.readerWait, -1) == 0 { 
    // 最后一个reader了，writer终于有机会获得锁了
    runtime_Semrelease(&rw.writerSem, false, 1) 
  }
}
```

### Lock

```go
func (rw *RWMutex) Lock() {
	// 首先解决其他writer竞争问题
	rw.w.Lock()
	// 反转readerCount，告诉reader有writer竞争锁
	r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders
	if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
    // 如果当前有reader持有锁，那么需要等待
		runtime_SemacquireMutex(&rw.writerSem, false, 0)
	}
}
```

### Unlock

```go
func (rw *RWMutex) Unlock() {
	// 告诉reader没有活跃的writer了
	r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
	if r >= rwmutexMaxReaders {
		throw("sync: Unlock of unlocked RWMutex")
	}
	for i := 0; i < int(r); i++ {
    // 唤醒阻塞的reader们
		runtime_Semrelease(&rw.readerSem, false, 0)
	}
  	rw.w.Unlock()
}
```

## WaitGroup

![](https://s2.loli.net/2022/06/13/PN2UGsImtBD5Vwe.png)

### 数据结构

```go
type WaitGroup struct {
	noCopy noCopy
	state1 [3]uint32
}
```

```go

type WaitGroup struct {
    // 避免复制使用的一个技巧，可以告诉vet工具违反了复制使用的规则
    noCopy noCopy
    // 64bit(8bytes)的值分成两段，高32bit是计数值，低32bit是waiter的计数
    // 另外32bit是用作信号量的
    // 因为64bit值的原子操作需要64bit对齐，但是32bit编译器不支持，所以数组中的元素在不同的架构中不一样，具体处理看下面的方法
    // 总之，会找到对齐的那64bit作为state，其余的32bit做信号量
    state1 [3]uint32
}


// 得到state的地址和信号量的地址
func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
    if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
        // 如果地址是64bit对齐的，数组前两个元素做state，后一个元素做信号量
        return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
    } else {
        // 如果地址是32bit对齐的，数组后两个元素用来做state，它可以用来做64bit的原子操作，第一个元素32bit用来做信号量
       //&wg.state1[0]是32位对齐的，&wg.state1[0]+4也就是&wg.state1[1]是64位对齐的
        return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
    }
}
```

如果wg初始地址64位对齐：

![](https://s2.loli.net/2022/06/09/5CAo8u9pMBOvJds.png)

如果wg初始地址32位对齐：

![](https://s2.loli.net/2022/06/09/HTbLiXlNjG3cgfY.png)

### Add

Add 方法主要操作的是 state 的计数部分。你可以为计数值增加一个 delta 值，内部通过原子操作把这个值加到计数值上。需要注意的是，这个 delta 也可以是个负数，相当于为计数值减去一个值，Done 方法内部其实就是通过 Add(-1) 实现的。

```go
func (wg *WaitGroup) Add(delta int) {
    statep, semap := wg.state()
    // 高32bit是计数值v，所以把delta左移32，增加到计数上
    state := atomic.AddUint64(statep, uint64(delta)<<32)
    v := int32(state >> 32) // 当前计数值
    w := uint32(state) // waiter count

  	if v < 0 {
		   panic("sync: negative WaitGroup counter")
	  }
    if v > 0 || w == 0 {
        return
    }
    // 执行到这里v = 0 && w >0。v = 0说明所有goroutine都完成了，w > 0说明有等待被唤醒的goroutine
    if *statep != state {
       //此刻别再调Add增加counter或者调Wait增加waiter了，防止滥用wg
		   panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	  }
    *statep = 0 // 将waiter的数量设置为0，因为计数值v也是0,所以它们俩的组合*statep直接设置为0即可
    for ; w != 0; w-- { //唤醒所有的waiter
        runtime_Semrelease(semap, false, 0)
    }
}

// Done方法实际就是计数器减1
func (wg *WaitGroup) Done() {
    wg.Add(-1)
}
```

### wait

Wait 方法的实现逻辑是：不断检查 state 的值。如果其中的计数值变为了 0，那么说明所有的任务已完成，调用者不必再等待，直接返回。如果计数值大于 0，说明此时还有任务没完成，那么调用者就变成了等待者，需要加入 waiter 队列，并且阻塞住自己。

```go
func (wg *WaitGroup) Wait() {
    statep, semap := wg.state()
    
    for {
        state := atomic.LoadUint64(statep)
        v := int32(state >> 32) // 当前计数值
        w := uint32(state) // waiter的数量
        if v == 0 {
            // 如果计数值为0, 调用这个方法的goroutine不必再等待，继续执行它后面的逻辑即可
            return
        }
        // 否则把waiter数量加1。期间可能有并发调用Wait的情况，所以最外层使用了一个for循环
        if atomic.CompareAndSwapUint64(statep, state, state+1) {
            // 阻塞休眠等待
            runtime_Semacquire(semap)
          	if *statep != 0 {
               //这轮wait还没完成，就开始重用wg。例如调用Wait后，接着又调用了Add
				       panic("sync: WaitGroup is reused before previous Wait has returned")
			      }
            // 被唤醒，不再阻塞，返回
            return
        }
    }
}
```

### 信号量

P操作:

```go
runtime_Semacquire(semap)
//semap为state中的信号量。该方法阻塞等待直到semap>0，然后执行semap-1
```

V操作:

```go
runtime_Semrelease(semap, false, 0)
//semap为state中的信号量。该方法执行semap+1
```

### 小结

- 可以同时有多个 Goroutine 等待当前 [`sync.WaitGroup`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/waitgroup.go#L20-L29) 计数器的归零，这些 Goroutine 会被同时唤醒；

- 可以wg.Add负数，但是需保证不能使计数器变成负数，否则就会导致 panic

- 等所有的 Add 方法调用之后再调用 Wait，否则就可能导致 panic 或者不期望的结果。

- WaitGroup 虽然可以重用，但是是有一个前提的，那就是必须等到上一轮的 Wait 完成之后，才能重用 WaitGroup 执行下一轮的 Add/Wait，如果你在 Wait 还没执行完的时候就调用下一轮 Add 方法，就有可能出现 panic。

  ```go
  func main() {
      var wg sync.WaitGroup
      wg.Add(1)
      go func() {
          time.Sleep(time.Millisecond)
          wg.Done() // 计数器减1
          wg.Add(1) // 计数值加1（第七行）
      }()
      wg.Wait() // 主goroutine等待，有可能和第7行并发执行
  }
  //panic: sync: WaitGroup is reused before previous Wait has returned
  ```

### 使用wg的建议

不重用 WaitGroup。新建一个 WaitGroup 不会带来多大的资源开销，重用反而更容易出错。

保证所有的 Add 方法调用都在 Wait 之前。

不传递负数给 Add 方法，只通过 Done 来给计数值减 1。

不做多余的 Done 方法调用，保证 Add 的计数值和 Done 方法调用的数量是一样的。

不遗漏 Done 方法的调用，否则会导致 Wait hang 住无法返回。

## Once

Once 常常用来初始化单例资源，或者并发访问只需初始化一次的共享资源，或者在测试的时候初始化一次测试资源。

### 数据结构

每一个 [`sync.Once`](https://github.com/golang/go/blob/bc593eac2dc63d979a575eccb16c7369a5ff81e0/src/sync/once.go#L12-L20) 结构体中都只包含一个用于标识代码块是否执行过的 `done` 以及一个互斥锁 [`sync.Mutex`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/mutex.go#L25-L28)：

```go
type Once struct {
	done uint32
	m    Mutex
}
```

### 实现

[`sync.Once.Do`](https://github.com/golang/go/blob/bc593eac2dc63d979a575eccb16c7369a5ff81e0/src/sync/once.go#L40-L59) 是 [`sync.Once`](https://github.com/golang/go/blob/bc593eac2dc63d979a575eccb16c7369a5ff81e0/src/sync/once.go#L12-L20) 结构体对外唯一暴露的方法，该方法会接收一个入参为空的函数：

- 如果传入的函数已经执行过，就会直接返回；
- 如果传入的函数没有执行过，就会调用 [`sync.Once.doSlow`](https://github.com/golang/go/blob/bc593eac2dc63d979a575eccb16c7369a5ff81e0/src/sync/once.go#L61-L68) 执行传入的函数：

**双检查的机制（double-checking）**

```go
func (o *Once) Do(f func()) {
	if atomic.LoadUint32(&o.done) == 0 { //第一次检查，f已经调用过之后的检查
		o.doSlow(f) //f没被调用过，多个Goroutine可能争夺调用权
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
  if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```

1. 为当前 Goroutine 获取互斥锁；
2. 执行传入的无入参函数；
3. 运行延迟函数调用，将成员变量 `done` 更新成 1；

[`sync.Once`](https://github.com/golang/go/blob/bc593eac2dc63d979a575eccb16c7369a5ff81e0/src/sync/once.go#L12-L20) 就会通过成员变量 `done` 确保函数不会执行第二次。

### 错误案例

#### 第一种错误：死锁

```go
func main() {
    var once sync.Once
    once.Do(func() {
        once.Do(func() {
            fmt.Println("初始化")
        })
    })
}
```

#### 第二种错误：未初始化

如果 f 方法执行的时候 panic，或者 f 执行初始化资源的时候失败了，这个时候，Once 还是会认为初次执行已经成功了，即使再次调用 Do 方法，也不会再次执行 f。

### 功能更加强大的Once

```go
// 一个功能更加强大的Once
type Once struct {
    m    sync.Mutex
    done uint32
}
// 传入的函数f有返回值error，如果初始化失败，需要返回失败的error
// Do方法会把这个error返回给调用者
func (o *Once) Do(f func() error) error {
    if atomic.LoadUint32(&o.done) == 1 { //fast path
        return nil
    }
    return o.slowDo(f)
}
// 如果还没有初始化
func (o *Once) slowDo(f func() error) error {
    o.m.Lock()
    defer o.m.Unlock()
    var err error
    if o.done == 0 { // 双检查，还没有初始化
        err = f()
        if err == nil { // 初始化成功才将标记置为已初始化
            atomic.StoreUint32(&o.done, 1)
        }
    }
    return err
}
```

### 问题

我已经分析了几个并发原语的实现，你可能注意到总是有些 slowXXXX 的方法，从 XXXX 方法中单独抽取出来，你明白为什么要这么做吗，有什么好处？

> 分离固定内容和非固定内容，使得固定的内容能被内联调用，从而优化执行过程。
>
>  fast path的一个好处是此方法可以内联

### 内联是什么？



## Cond

### 易错点

#### 调用 Wait 的时候不加锁

```go
//    c.L.Lock()
//    for !condition() {
//        c.Wait()
//    }
//    ... make use of condition ...
//    c.L.Unlock()
//
func (c *Cond) Wait() {
	c.checker.check()
	t := runtime_notifyListAdd(&c.notify)
	c.L.Unlock()
	runtime_notifyListWait(&c.notify, t)
	c.L.Lock()
}
```

如果调用 Wait 之前不加锁的话，就有可能 Unlock 一个未加锁的 Locker。所以切记，调用 cond.Wait 方法之前一定要加锁。

#### 只调用了一次 Wait，没有检查等待条件是否满足

我们一定要记住，waiter goroutine 被唤醒不等于等待条件被满足，只是有 goroutine 把它唤醒了而已，等待条件有可能已经满足了，也有可能不满足，我们需要进一步检查。你也可以理解为，等待者被唤醒，只是得到了一次检查的机会而已。

### cond与channel

- Cond 和一个 Locker 关联，可以利用这个 Locker 对相关的依赖条件更改提供保护。
- Cond 可以同时支持 Signal 和 Broadcast 方法，而 Channel 只能同时支持其中一种。
- Cond 的 Broadcast 方法可以被重复调用。等待条件再次变成不满足的状态后，我们又可以调用 Broadcast 再次唤醒等待的 goroutine。这也是 Channel 不能支持的，Channel 被 close 掉了之后不支持再 open。

## Pool

### sync.Pool

- sync.Pool 本身就是线程安全的，多个 goroutine 可以并发地调用它的方法存取对象；

- sync.Pool 不可在使用之后再复制使用。

![](https://s2.loli.net/2022/06/07/wM5nzdbcSK4FrpO.png)

#### 数据结构

```go
type Pool struct {
	noCopy noCopy

	local     unsafe.Pointer // local fixed-size per-P pool, actual type is [P]poolLocal
  //存放poolLocal
  
	localSize uintptr        // size of the local array ，localSize == runtime.GOMAXPROCS(0)

	victim     unsafe.Pointer // local from previous cycle
	victimSize uintptr        // size of victims array

	// New optionally specifies a function to generate
	// a value when Get would otherwise return nil.
	// It may not be changed concurrently with calls to Get.
	New func() any
}

	// Local per-P Pool appendix.
type poolLocalInternal struct {
	private any       // Can be used only by the respective P.private，代表一个缓存的元素，而且只能由相应的一个 P 存取。因为一个 P 同时只能执行一个 goroutine，所以不会有并发的问题。
	shared  poolChain // Local P can pushHead/popHead; any P can popTail.shared，可以由任意的 P 访问，但是只有本地的 P 才能 pushHead/popHead，其它 P 可以 popTail，相当于只有一个本地的 P 作为生产者（Producer），多个 P 作为消费者（Consumer），它是使用一个 local-free 的 queue 列表实现的。
}
```



#### Get

```go
func (p *Pool) Get() interface{} {
  // 把当前goroutine固定在当前的P上
  //pin 方法会将此 goroutine 固定在当前的 P 上，避免查找元素期间被其它的 P 执行。固定的好处就是查找元素期间直接得到跟这个 P 相关的 local。直接得到P相关的local又如何？
    l, pid := p.pin()
    x := l.private // 优先从local的private字段取，快速
    l.private = nil
    if x == nil {
        // 从当前的local.shared弹出一个，注意是从head读取并移除
        x, _ = l.shared.popHead()
        if x == nil { // 如果没有，则去偷一个
            x = p.getSlow(pid) 
        }
    }
    runtime_procUnpin()
    // 如果没有获取到，尝试使用New函数生成一个新的
    if x == nil && p.New != nil {
        x = p.New()
    }
    return x
}

func (p *Pool) getSlow(pid int) any {
	size := runtime_LoadAcquintptr(&p.localSize) // load-acquire
	locals := p.local                            // load-consume
	// Try to steal one element from other procs.
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i+1)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}

	// Try the victim cache. We do this after attempting to steal
	// from all primary caches because we want objects in the
	// victim cache to age out if at all possible.
	size = atomic.LoadUintptr(&p.victimSize)
	if uintptr(pid) >= size {
		return nil
	}
	locals = p.victim
	l := indexLocal(locals, pid)
	if x := l.private; x != nil {
		l.private = nil
		return x
	}
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}

	return nil
}
```

#### Put

```go
func (p *Pool) Put(x interface{}) {
    if x == nil { // nil值直接丢弃
        return
    }
    l, _ := p.pin() // 把当前goroutine固定在当前的P上
    if l.private == nil { // 如果本地private没有值，直接设置这个值即可
        l.private = x
        x = nil
    }
    if x != nil { // 否则加入到本地队列中
        l.shared.pushHead(x)
    }
    runtime_procUnpin()
}
```

#### pin

```go
//将当前g绑定在P上
func (p *Pool) pin() (*poolLocal, int) {
	pid := runtime_procPin()
	s := runtime_LoadAcquintptr(&p.localSize) // load-acquire
	l := p.local                          
	if uintptr(pid) < s {
		return indexLocal(l, pid), pid     // 通过pid获取当前g所在P对应的poolLocal
	}
	return p.pinSlow()
}
func (p *Pool) pinSlow() (*poolLocal, int) {
	// Retry under the mutex.
	// Can not lock the mutex while pinned.
	runtime_procUnpin()
	allPoolsMu.Lock()
	defer allPoolsMu.Unlock()
	pid := runtime_procPin()
	// poolCleanup won't be called while we are pinned.
	s := p.localSize
	l := p.local
	if uintptr(pid) < s {
		return indexLocal(l, pid), pid
	}
	if p.local == nil {
		allPools = append(allPools, p)
	}
	// If GOMAXPROCS changes between GCs, we re-allocate the array and lose the old one.
	size := runtime.GOMAXPROCS(0)
	local := make([]poolLocal, size)   //创建poolLocal切片，len=P数量
	atomic.StorePointer(&p.local, unsafe.Pointer(&local[0])) // store-release
	runtime_StoreReluintptr(&p.localSize, uintptr(size))     // store-release
	return &local[pid], pid
}
```

#### GC期间

```go
func poolCleanup() {
    // 丢弃当前victim, STW所以不用加锁
    for _, p := range oldPools {
        p.victim = nil
        p.victimSize = 0
    }

    // 将local复制给victim, 并将原local置为nil
    for _, p := range allPools {
        p.victim = p.local
        p.victimSize = p.localSize
        p.local = nil
        p.localSize = 0
    }

    oldPools, allPools = allPools, nil
}
```

发生GC时，清空victim，将local赋值给victim，清空local

例如，local中有1000个对象，发生GC的时候，1000个对象被放入victim中。GC后，如果有新请求需要这些对象，从local中找不到，可以从victim中找到，不用重新分配内存创建这些新对象。**可以防止GC前后的请求延迟波动**

![](https://s2.loli.net/2022/06/07/7gqmX2SU8jQluBi.png)

#### 内存泄漏

##### 原因

GC对于sync.Pool大对象的回收策略问题

##### 解决方法

在使用 sync.Pool 回收 buffer 的时候，一定要检查回收的对象的大小。如果 buffer 太大，就不要回收了,否则会导致内存泄漏。

一、要做到物尽其用，尽可能不浪费的话，我们可以将 buffer 池分成几层。

```go
//标准库http
var (
	http2dataChunkSizeClasses = []int{
		1 << 10,
		2 << 10,
		4 << 10,
		8 << 10,
		16 << 10,
	}
	http2dataChunkPools = [...]sync.Pool{
		{New: func() interface{} { return make([]byte, 1<<10) }},
		{New: func() interface{} { return make([]byte, 2<<10) }},
		{New: func() interface{} { return make([]byte, 4<<10) }},
		{New: func() interface{} { return make([]byte, 8<<10) }},
		{New: func() interface{} { return make([]byte, 16<<10) }},
	}
)

func http2getDataBufferChunk(size int64) []byte {
	i := 0
	for ; i < len(http2dataChunkSizeClasses)-1; i++ {
		if size <= int64(http2dataChunkSizeClasses[i]) {
			break
		}
	}
	return http2dataChunkPools[i].Get().([]byte)
}

func http2putDataBufferChunk(p []byte) {
	for i, n := range http2dataChunkSizeClasses {
		if len(p) == n {
			http2dataChunkPools[i].Put(p)
			return
		}
	}
	panic(fmt.Sprintf("unexpected buffer len=%v", len(p)))
}
```

二、限制放回的大小

```go
//fmt包
func (p *pp) free() {
	// Proper usage of a sync.Pool requires each entry to have approximately
	// the same memory cost. To obtain this property when the stored type
	// contains a variably-sized buffer, we add a hard limit on the maximum buffer
	// to place back in the pool.
	//
	// See https://golang.org/issue/23199
  //限制放回的大小
	if cap(p.buf) > 64<<10 {
		return
	}

	p.buf = p.buf[:0]
	p.arg = nil
	p.value = reflect.Value{}
	p.wrappedErr = nil
	ppFree.Put(p)
}
```



#### 总结

事实上，我们很少会使用 sync.Pool 去池化连接对象，原因就在于，sync.Pool 会无通知地在某个时候就把连接移除垃圾回收掉了，而我们的场景是需要长久保持这个连接，所以，我们一般会使用其它方法来池化连接

### workPool

#### fasthttp

```go
type workerPool struct {
	// Function for serving server connections.
	// It must leave c unclosed.
	WorkerFunc ServeHandler

	MaxWorkersCount int

	LogAllErrors bool

	MaxIdleWorkerDuration time.Duration

	Logger Logger

	lock         sync.Mutex
	workersCount int

	ready []*workerChan //就绪的worker

	stopCh chan struct{}

	workerChanPool sync.Pool
}

type workerChan struct {
	lastUseTime time.Time
	ch          chan net.Conn
```

```go
func (s *Server) Serve(ln net.Listener) error {
  ...
  //初始化workpool
  	wp := &workerPool{
		WorkerFunc:            s.serveConn,
		MaxWorkersCount:       maxWorkersCount,
		LogAllErrors:          s.LogAllErrors,
		MaxIdleWorkerDuration: s.MaxIdleWorkerDuration,
		Logger:                s.logger(),
		connState:             s.setState,
	}
	wp.Start()
  for {
    //Accept拿到net.conn
		if c, err = acceptConn(s, ln, &lastPerIPErrorTime); err != nil {
			wp.Stop()
			if err == io.EOF {
				return nil
			}
			return err
		}
    //将Accept到的net.conn放入channel中,无缓冲或缓冲为1
    if !wp.Serve(c) {
      ...
    }
  }
}
func (wp *workerPool) Serve(c net.Conn) bool {
  //获取ready worker的ch，没有则创建
	ch := wp.getCh()
	if ch == nil {
		return false
	}
  //将net.conn放入channel
	ch.ch <- c
	return true
}
```



```go
func (wp *workerPool) getCh() *workerChan {
   var ch *workerChan
   createWorker := false

   wp.lock.Lock()
   ready := wp.ready
   n := len(ready) - 1
   if n < 0 {
      if wp.workersCount < wp.MaxWorkersCount { //默认最多256 * 1024个worker
         createWorker = true
         wp.workersCount++
      }
   } else {
      ch = ready[n]
      ready[n] = nil
      wp.ready = ready[:n]
   }
   wp.lock.Unlock()

   if ch == nil {
      if !createWorker {
         return nil
      }
      //没有可用worker，创建
      vch := wp.workerChanPool.Get()
      ch = vch.(*workerChan)
      go func() {
         wp.workerFunc(ch) //从channel中拿取net.conn,并处理
         wp.workerChanPool.Put(vch)
      }()
   }
   return ch
}
```

## Semaphore

信号量（Semaphore）是用来控制**多个 goroutine 同时访问多个资源**的并发原语

```
function P(semaphore S, integer I):
    repeat:
        [if S ≥ I: //等待直到资源数>=I
        S ← S − I
        break]
        
function V(semaphore S, integer I):
    [S ← S + I]
```

使用信号量遵循的原则就是请求多少资源，就释放多少资源

### 死锁的四个条件

- **禁止抢占**（no preemption）：系统资源不能被强制从一个线程中退出。如果哲学家可以抢夺，那么大家都去抢别人的筷子，也会打破死锁的局面，但这是有风险的，因为可能孔子还没吃饭就被老子抢走了。计算机资源如果不是主动释放而是被抢夺有可能出现意想不到的现象。
- **持有和等待**（hold and wait）：一个线程在等待时持有并发资源。持有并发资源并还等待其它资源，也就是吃着碗里的望着锅里的。
- **互斥**（mutual exclusion）：资源只能同时分配给一个线程，无法多个线程共享。资源具有排他性，孔子和老子的关系再好，也不允许他们俩一起拿着一根筷同时吃。
- **循环等待**（circular waiting）：一系列线程互相持有其他进程所需要的资源。必须有一个循环依赖的关系。

死锁只有在四个条件同时满足时发生，预防死锁必须至少破坏其中一项。

### 哲学家就餐问题

#### 解法一: 限制就餐人数

我们把这个信号量初始值设置为4，代表最多允许同时4位哲学家就餐。把这个信号量传给哲学家对象，哲学家想就餐时就请求这个信号量，如果能得到一个许可，就可以就餐，吃完把许可释放回给信号量。

#### 解法二：奇偶资源

我们给每一位哲学家编号，从1到5, 如果我们规定奇数号的哲学家首先拿左手边的筷子，再拿右手边的筷子，偶数号的哲学家先拿右手边的筷子，再拿左手边的筷子， 释放筷子的时候按照相反的顺序，这样也可以避免出现循环依赖的情况。

#### 解法三：资源分级

另一个简单的解法是为资源（这里是筷子）分配一个偏序或者分级的关系，并约定所有资源都按照这种顺序获取，按相反顺序释放，而且保证不会有两个无关资源同时被同一项工作所需要。在哲学家就餐问题中，筷子按照某种规则编号为1至5，每一个工作单元（哲学家）总是先拿起左右两边编号较低的筷子，再拿编号较高的。用完筷子后，他总是先放下编号较高的筷子，再放下编号较低的。在这种情况下，当四位哲学家同时拿起他们手边编号较低的筷子时，只有编号最高的筷子留在桌上，从而第五位哲学家就不能使用任何一只筷子了。而且，只有一位哲学家能使用最高编号的筷子，所以他能使用两只筷子用餐。当他吃完后，他会先放下编号最高的筷子，再放下编号较低的筷子，从而让另一位哲学家拿起后边的这只开始吃东西。

```go
/ 无休止的进餐和冥想.
// 吃完睡(冥想、打坐), 睡完吃.
// 可以调整吃睡的时间来增加或者减少抢夺叉子的机会.
func (p *Philosopher) dine() {
    for {
        mark(p, "冥想")
        randomPause(10)

        mark(p, "饿了")
        if p.ID == 5 { //
            p.rightChopstick.Lock() // 先尝试拿起第1只筷子
            mark(p, "拿起左手筷子")
            p.leftChopstick.Lock() // 再尝试拿起第5只筷子

            mark(p, "用膳")
            randomPause(10)

            p.leftChopstick.Unlock()  // 先尝试放下第5只的筷子
            p.rightChopstick.Unlock() // 再尝试放下第1只的筷子
        } else {
            p.leftChopstick.Lock() // 先尝试拿起左手边的筷子(第n只)
            mark(p, "拿起右手筷子")
            p.rightChopstick.Lock() // 再尝试拿起右手边的筷子(第n+1只)

            mark(p, "用膳")
            randomPause(10)

            p.rightChopstick.Unlock() // 先尝试放下右手边的筷子
            p.leftChopstick.Unlock()  // 再尝试拿起左手边的筷子
        }

    }
}
```

#### 解法四：引入服务生

如果我们引入一个服务生，比如韩非子，由韩非子负责分配筷子，这样我们就可以将拿左手筷子和右手筷子看成一个原子操作，要么拿到筷子，要么等待，就可以破坏死锁的第二个条件(持有和等待)。

```go
type Philosopher struct {
    // 哲学家的名字
    name string
    // 左手一只和右手一只筷子
    leftChopstick, rightChopstick *Chopstick
    status                        string

    mu *sync.Mutex
}

// 无休止的进餐和冥想.
// 吃完睡(冥想、打坐), 睡完吃.
// 可以调整吃睡的时间来增加或者减少抢夺叉子的机会.
func (p *Philosopher) dine() {
    for {
        mark(p, "冥想")
        randomPause(10)

        mark(p, "饿了")
        p.mu.Lock() // 服务生控制
        p.leftChopstick.Lock() // 先尝试拿起左手边的筷子
        mark(p, "拿起左手筷子")
        p.rightChopstick.Lock() // 再尝试拿起右手边的筷子
        p.mu.Unlock() 

        mark(p, "用膳")
        randomPause(10)

        p.rightChopstick.Unlock() // 先尝试放下右手边的筷子
        p.leftChopstick.Unlock()  // 再尝试拿起左手边的筷子
    }
}
```

### go内置Semaphore

<img src="https://s2.loli.net/2022/06/13/PUcfTBowlXGpQi8.png" style="zoom:20%;" />

#### 数据结构

```go
type semaRoot struct {
	lock  mutex
	treap *sudog // root of balanced tree of unique waiters.
	nwait uint32 // Number of waiters. Read w/o the lock.
}

const semTabSize = 251
var semtable [semTabSize]struct {
   root semaRoot //树
   pad  [cpu.CacheLinePadSize - unsafe.Sizeof(semaRoot{})]byte
}

func semroot(addr *uint32) *semaRoot {
  //通过计算(信号量地址>>3%251),选择251棵树中其中一颗
	return &semtable[(uintptr(unsafe.Pointer(addr))>>3)%semTabSize].root
}
```
