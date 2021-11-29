---
title: "go之同步原语"
date: 2021-11-29T11:14:03+08:00
draft: false
---

# 同步原语与锁

## Mutex

### 数据结构

```go
type Mutex struct {
	state int32  //表示当前互斥锁的状态
	sema  uint32 //用于控制锁状态的信号量
}
```

### 互斥锁的状态

互斥锁的状态比较复杂，如下图所示，最低三位分别表示 `mutexLocked`、`mutexWoken` 和 `mutexStarving`，剩下的位置用来表示当前有多少个 Goroutine 等待互斥锁的释放：

![golang-mutex-state](https://img.draveness.me/2020-01-23-15797104328010-golang-mutex-state.png)

**图 6-6 互斥锁的状态**

在默认情况下，互斥锁的所有状态位都是 `0`，`int32` 中的不同位分别表示了不同的状态：

- `mutexLocked` — 表示互斥锁的锁定状态；
- `mutexWoken` — 表示从正常模式被从唤醒；
- `mutexStarving` — 当前的互斥锁进入饥饿状态；

### 互斥锁的正常模式和饥饿模式

[`sync.Mutex`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/mutex.go#L25-L28) 有两种模式 — 正常模式和饥饿模式。我们需要在这里先了解正常模式和饥饿模式都是什么，它们有什么样的关系。

在正常模式下，锁的等待者会按照先进先出的顺序获取锁。但是刚被唤起的 Goroutine 与新创建的 Goroutine 竞争时，大概率会获取不到锁，为了减少这种情况的出现，一旦 Goroutine 超过 1ms 没有获取到锁，它就会将当前互斥锁切换饥饿模式，防止部分 Goroutine 被『饿死』。

![golang-mutex-mode](https://img.draveness.me/2020-01-23-15797104328020-golang-mutex-mode.png)

**图 6-7 互斥锁的正常模式与饥饿模式**

饥饿模式是在 Go 语言 [1.9](https://github.com/golang/go/commit/0556e26273f704db73df9e7c4c3d2e8434dec7be) 版本引入的优化[1](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-sync-primitives/#fn:1)，引入的目的是保证互斥锁的公平性（Fairness）。

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

## RWMutex

### 数据结构

```go
type RWMutex struct {
	w           Mutex
	writerSem   uint32
	readerSem   uint32
	readerCount int32
	readerWait  int32
}
//w — 复用互斥锁提供的能力；
//writerSem 和 readerSem — 分别用于写等待读和读等待写：
//readerCount 存储了当前正在执行的读操作的数量；
//readerWait 表示当写操作被阻塞时等待的读操作个数；
```

### 写锁

### 读锁

读锁的加锁方法 [`sync.RWMutex.RLock`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/rwmutex.go#L43-L56) 很简单，该方法会通过 [`atomic.AddInt32`](https://github.com/golang/go/blob/ca7c12d4c9eb4a19ca5103ec5763537cccbcc13b/src/sync/atomic/doc.go#L93) 将 `readerCount` 加一：

```go
func (rw *RWMutex) RLock() {
	if atomic.AddInt32(&rw.readerCount, 1) < 0 {
		runtime_SemacquireMutex(&rw.readerSem, false, 0)
	}
}
```

1. 如果该方法返回负数 — 其他 Goroutine 获得了写锁，当前 Goroutine 就会调用 [`sync.runtime_SemacquireMutex`](https://github.com/golang/go/blob/a4c579e8f7c8129b2c27779f206ebd2c9b393633/src/runtime/sema.go#L70-L72) 陷入休眠等待锁的释放；
2. 如果该方法的结果为非负数 — 没有 Goroutine 获得写锁，当前方法就会成功返回；

当 Goroutine 想要释放读锁时，会调用如下所示的 [`sync.RWMutex.RUnlock`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/rwmutex.go#L62-L75) 方法：

```go
func (rw *RWMutex) RUnlock() {
	if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
		rw.rUnlockSlow(r)
	}
}
```

该方法会先减少正在读资源的 `readerCount` 整数，根据 [`atomic.AddInt32`](https://github.com/golang/go/blob/ca7c12d4c9eb4a19ca5103ec5763537cccbcc13b/src/sync/atomic/doc.go#L93) 的返回值不同会分别进行处理：

- 如果返回值大于等于零 — 读锁直接解锁成功；
- 如果返回值小于零 — 有一个正在执行的写操作，在这时会调用[`sync.RWMutex.rUnlockSlow`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/rwmutex.go#L77-L87) 方法；

```go
func (rw *RWMutex) rUnlockSlow(r int32) {
	if r+1 == 0 || r+1 == -rwmutexMaxReaders {
		throw("sync: RUnlock of unlocked RWMutex")
	}
	if atomic.AddInt32(&rw.readerWait, -1) == 0 {
		runtime_Semrelease(&rw.writerSem, false, 1)
	}
}
```

[`sync.RWMutex.rUnlockSlow`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/rwmutex.go#L77-L87) 会减少获取锁的写操作等待的读操作数 `readerWait` 并在所有读操作都被释放之后触发写操作的信号量 `writerSem`，该信号量被触发时，调度器就会唤醒尝试获取写锁的 Goroutine。



读写互斥锁 [`sync.RWMutex`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/rwmutex.go#L28-L34) 虽然提供的功能非常复杂，不过因为它建立在 [`sync.Mutex`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/mutex.go#L25-L28) 上，所以整体的实现上会简单很多。我们总结一下读锁和写锁的关系：

- 调用`sync.RWMutex.Lock`尝试获取写锁时；
  - 每次 [`sync.RWMutex.RUnlock`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/rwmutex.go#L62-L75) 都会将 `readerCount` 其减一，当它归零时该 Goroutine 就会获得写锁；
  - 将 `readerCount` 减少 `rwmutexMaxReaders` 个数以阻塞后续的读操作；
- 调用 [`sync.RWMutex.Unlock`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/rwmutex.go#L118-L140) 释放写锁时，会先通知所有的读操作，然后才会释放持有的互斥锁；

读写互斥锁在互斥锁之上提供了额外的更细粒度的控制，能够在读操作远远多于写操作时提升性能。

## WaitGroup

[`sync.WaitGroup`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/waitgroup.go#L20-L29) 可以等待一组 Goroutine 的返回，一个比较常见的使用场景是批量发出 RPC 或者 HTTP 请求：

```go
requests := []*Request{...}
wg := &sync.WaitGroup{}
wg.Add(len(requests))

for _, request := range requests {
    go func(r *Request) {
        defer wg.Done()
        // res, err := service.call(r)
    }(request)
}
wg.Wait()
```

我们可以通过 [`sync.WaitGroup`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/waitgroup.go#L20-L29) 将原本顺序执行的代码在多个 Goroutine 中并发执行，加快程序处理的速度。

![golang-syncgroup](https://img.draveness.me/2020-01-23-15797104328028-golang-syncgroup.png)

**图 6-8 WaitGroup 等待多个 Goroutine**

### 数据结构

```go
type WaitGroup struct {
	noCopy noCopy
	state1 [3]uint32
}
```

### 小结

- [`sync.WaitGroup`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/waitgroup.go#L20-L29) 必须在 [`sync.WaitGroup.Wait`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/waitgroup.go#L103-L141) 方法返回之后才能被重新使用；
- [`sync.WaitGroup.Done`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/waitgroup.go#L98-L100) 只是对 [`sync.WaitGroup.Add`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/waitgroup.go#L53-L95) 方法的简单封装，我们可以向 [`sync.WaitGroup.Add`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/waitgroup.go#L53-L95) 方法传入任意负数（需要保证计数器非负）快速将计数器归零以唤醒其他等待的 Goroutine；
- 可以同时有多个 Goroutine 等待当前 [`sync.WaitGroup`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/waitgroup.go#L20-L29) 计数器的归零，这些 Goroutine 会被同时唤醒；

## Once

Go 语言标准库中 [`sync.Once`](https://github.com/golang/go/blob/bc593eac2dc63d979a575eccb16c7369a5ff81e0/src/sync/once.go#L12-L20) 可以保证在 Go 程序运行期间的某段代码只会执行一次。在运行如下所示的代码时，我们会看到如下所示的运行结果：

```go
func main() {
    o := &sync.Once{}
    for i := 0; i < 10; i++ {
        o.Do(func() {
            fmt.Println("only once")
        })
    }
}

$ go run main.go
only once
```

### 结构体

每一个 [`sync.Once`](https://github.com/golang/go/blob/bc593eac2dc63d979a575eccb16c7369a5ff81e0/src/sync/once.go#L12-L20) 结构体中都只包含一个用于标识代码块是否执行过的 `done` 以及一个互斥锁 [`sync.Mutex`](https://github.com/golang/go/blob/71239b4f491698397149868c88d2c851de2cd49b/src/sync/mutex.go#L25-L28)：

```go
type Once struct {
	done uint32
	m    Mutex
}
```

### 接口

[`sync.Once.Do`](https://github.com/golang/go/blob/bc593eac2dc63d979a575eccb16c7369a5ff81e0/src/sync/once.go#L40-L59) 是 [`sync.Once`](https://github.com/golang/go/blob/bc593eac2dc63d979a575eccb16c7369a5ff81e0/src/sync/once.go#L12-L20) 结构体对外唯一暴露的方法，该方法会接收一个入参为空的函数：

- 如果传入的函数已经执行过，就会直接返回；
- 如果传入的函数没有执行过，就会调用 [`sync.Once.doSlow`](https://github.com/golang/go/blob/bc593eac2dc63d979a575eccb16c7369a5ff81e0/src/sync/once.go#L61-L68) 执行传入的函数：

```go
func (o *Once) Do(f func()) {
	if atomic.LoadUint32(&o.done) == 0 {
		o.doSlow(f)
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

### 小结

作为用于保证函数执行次数的 [`sync.Once`](https://github.com/golang/go/blob/bc593eac2dc63d979a575eccb16c7369a5ff81e0/src/sync/once.go#L12-L20) 结构体，它使用互斥锁和 [`sync/atomic`](https://github.com/golang/go/tree/master/src/sync/atomic) 包提供的方法实现了某个函数在程序运行期间只能执行一次的语义。在使用该结构体时，我们也需要注意以下的问题：

- [`sync.Once.Do`](https://github.com/golang/go/blob/bc593eac2dc63d979a575eccb16c7369a5ff81e0/src/sync/once.go#L40-L59) 方法中传入的函数只会被执行一次，哪怕函数中发生了 `panic`；
- 两次调用 [`sync.Once.Do`](https://github.com/golang/go/blob/bc593eac2dc63d979a575eccb16c7369a5ff81e0/src/sync/once.go#L40-L59) 方法传入不同的函数也只会执行第一次调用的函数；

## Cond

Go 语言标准库中的 [`sync.Cond`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L21-L29) 一个条件变量，它可以让一系列的 Goroutine 都在满足特定条件时被唤醒。每一个 [`sync.Cond`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L21-L29) 结构体在初始化时都需要传入一个互斥锁，我们可以通过下面的例子了解它的使用方法：

```go
var status int64

func main() {
	c := sync.NewCond(&sync.Mutex{})
	for i := 0; i < 10; i++ {
		go listen(c)
	}
	time.Sleep(1 * time.Second)
	go broadcast(c)

	ch := make(chan os.Signal, 1)
	signal.Notify(ch, os.Interrupt)
	<-ch
}

func broadcast(c *sync.Cond) {
	c.L.Lock()
	atomic.StoreInt64(&status, 1)
	c.Broadcast()
	c.L.Unlock()
}

func listen(c *sync.Cond) {
	c.L.Lock()
	for atomic.LoadInt64(&status) != 1 {
		c.Wait()
	}
	fmt.Println("listen")
	c.L.Unlock()
}

$ go run main.go
listen
...
listen
```

上述代码同时运行了 11 个 Goroutine，这 11 个 Goroutine 分别做了不同事情：

- 10 个 Goroutine 通过 [`sync.Cond.Wait`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L52-L58) 等待特定条件的满足；
- 1 个 Goroutine 会调用 [`sync.Cond.Broadcast`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L73-L76) 方法通知所有陷入等待的 Goroutine；

调用 [`sync.Cond.Broadcast`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L73-L76) 方法后，上述代码会打印出 10 次 “listen” 并结束调用。

![golang-cond-broadcast](https://img.draveness.me/2020-01-23-15797104328042-golang-cond-broadcast.png)

**图 6-10 Cond 条件广播**

### 结构体

[`sync.Cond`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L21-L29) 的结构体中包含以下 4 个字段：

```go
type Cond struct {
	noCopy  noCopy
	L       Locker
	notify  notifyList
	checker copyChecker
}
```

- `noCopy` — 用于保证结构体不会在编译期间拷贝；
- `copyChecker` — 用于禁止运行期间发生的拷贝；
- `L` — 用于保护内部的 `notify` 字段，`Locker` 接口类型的变量；
- `notify` — 一个 Goroutine 的链表，它是实现同步机制的核心结构；

```go
type notifyList struct {
	wait uint32
	notify uint32

	lock mutex
	head *sudog
	tail *sudog
}
```

在 [`sync.notifyList`](https://github.com/golang/go/blob/41cb0aedffdf4c5087de82710c4d016a3634b4ac/src/sync/runtime.go#L33-L39) 结构体中，`head` 和 `tail` 分别指向的链表的头和尾，`wait` 和 `notify` 分别表示当前正在等待的和已经通知到的 Goroutine，我们通过这两个变量就能确认当前待通知和已通知的 Goroutine。

### 接口

[`sync.Cond`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L21-L29) 对外暴露的 [`sync.Cond.Wait`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L52-L58) 方法会将当前 Goroutine 陷入休眠状态，它的执行过程分成以下两个步骤：

1. 调用 [`runtime.notifyListAdd`](https://github.com/golang/go/blob/a4c579e8f7c8129b2c27779f206ebd2c9b393633/src/runtime/sema.go#L479-L483) 将等待计数器加一并解锁；
2. 调用 [`runtime.notifyListWait`](https://github.com/golang/go/blob/a4c579e8f7c8129b2c27779f206ebd2c9b393633/src/runtime/sema.go#L488-L518) 等待其他 Goroutine 的唤醒并加锁：

```go
func (c *Cond) Wait() {
	c.checker.check()
	t := runtime_notifyListAdd(&c.notify) // runtime.notifyListAdd 的链接名
	c.L.Unlock()
	runtime_notifyListWait(&c.notify, t) // runtime.notifyListWait 的链接名
	c.L.Lock()
}

func notifyListAdd(l *notifyList) uint32 {
	return atomic.Xadd(&l.wait, 1) - 1
}
```

[`runtime.notifyListWait`](https://github.com/golang/go/blob/a4c579e8f7c8129b2c27779f206ebd2c9b393633/src/runtime/sema.go#L488-L518) 函数会获取当前 Goroutine 并将它追加到 Goroutine 通知链表的最末端：

```go
func notifyListWait(l *notifyList, t uint32) {
	s := acquireSudog()
	s.g = getg()
	s.ticket = t
	if l.tail == nil {
		l.head = s
	} else {
		l.tail.next = s
	}
	l.tail = s
	goparkunlock(&l.lock, waitReasonSyncCondWait, traceEvGoBlockCond, 3)
	releaseSudog(s)
}
```

除了将当前 Goroutine 追加到链表的末端之外，我们还会调用 [`runtime.goparkunlock`](https://github.com/golang/go/blob/cfe3cd903f018dec3cb5997d53b1744df4e53909/src/runtime/proc.go#L309-L311) 将当前 Goroutine 陷入休眠状态，该函数也是在 Go 语言切换 Goroutine 时经常会使用的方法，它会直接让出当前处理器的使用权并等待调度器的唤醒。

![golang-cond-notifylist](https://img.draveness.me/2020-01-23-15797104328049-golang-cond-notifylist.png)

**图 6-11 Cond 条件通知列表**

[`sync.Cond.Signal`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L64-L67) 和 [`sync.Cond.Broadcast`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L73-L76) 方法就是用来唤醒调用 [`sync.Cond.Wait`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L52-L58) 陷入休眠的 Goroutine，它们两个的实现有一些细微差别：

- [`sync.Cond.Signal`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L64-L67) 方法会唤醒队列最前面的 Goroutine；
- [`sync.Cond.Broadcast`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L73-L76) 方法会唤醒队列中全部的 Goroutine；

```go
func (c *Cond) Signal() {
	c.checker.check()
	runtime_notifyListNotifyOne(&c.notify)
}

func (c *Cond) Broadcast() {
	c.checker.check()
	runtime_notifyListNotifyAll(&c.notify)
}
```

[`runtime.notifyListNotifyOne`](https://github.com/golang/go/blob/a4c579e8f7c8129b2c27779f206ebd2c9b393633/src/runtime/sema.go#L554-L604) 函数只会从 [`sync.notifyList`](https://github.com/golang/go/blob/41cb0aedffdf4c5087de82710c4d016a3634b4ac/src/sync/runtime.go#L33-L39) 链表中找到满足 `sudog.ticket == l.notify` 条件的 Goroutine 并通过 `readyWithTime` 唤醒：

```go
func notifyListNotifyOne(l *notifyList) {
	t := l.notify
	atomic.Store(&l.notify, t+1)

	for p, s := (*sudog)(nil), l.head; s != nil; p, s = s, s.next {
		if s.ticket == t {
			n := s.next
			if p != nil {
				p.next = n
			} else {
				l.head = n
			}
			if n == nil {
				l.tail = p
			}
			s.next = nil
			readyWithTime(s, 4)
			return
		}
	}
}
```

[`runtime.notifyListNotifyAll`](https://github.com/golang/go/blob/a4c579e8f7c8129b2c27779f206ebd2c9b393633/src/runtime/sema.go#L522-L550) 会依次通过 [`runtime.readyWithTime`](https://github.com/golang/go/blob/a4c579e8f7c8129b2c27779f206ebd2c9b393633/src/runtime/sema.go#L79-L84) 函数唤醒链表中 Goroutine：

```go
func notifyListNotifyAll(l *notifyList) {
	s := l.head
	l.head = nil
	l.tail = nil

	atomic.Store(&l.notify, atomic.Load(&l.wait))

	for s != nil {
		next := s.next
		s.next = nil
		readyWithTime(s, 4)
		s = next
	}
}
```

Goroutine 的唤醒顺序也是按照加入队列的先后顺序，先加入的会先被唤醒，而后加入的 Goroutine 需要等待调度器的调度。

在一般情况下，我们都会先调用 [`sync.Cond.Wait`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L52-L58) 陷入休眠等待满足期望条件，当满足唤醒条件时，就可以选择使用 [`sync.Cond.Signal`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L64-L67) 或者 [`sync.Cond.Broadcast`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L73-L76) 唤醒一个或者全部的 Goroutine。

### 小结

[`sync.Cond`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L21-L29) 不是一个常用的同步机制，在遇到长时间条件无法满足时，与使用 `for {}` 进行忙碌等待相比，[`sync.Cond`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L21-L29) 能够让出处理器的使用权。在使用的过程中我们需要注意以下问题：

- [`sync.Cond.Wait`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L52-L58) 方法在调用之前一定要使用获取互斥锁，否则会触发程序崩溃；
- [`sync.Cond.Signal`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L64-L67) 方法唤醒的 Goroutine 都是队列最前面、等待最久的 Goroutine；
- [`sync.Cond.Broadcast`](https://github.com/golang/go/blob/71bbffbc48d03b447c73da1f54ac57350fc9b36a/src/sync/cond.go#L73-L76) 会按照一定顺序广播通知等待的全部 Goroutine；

















