title: 【Go 语言设计与实现】 笔记 — Channel 源码分析
date: 2024-06-03 21:06:55
tags:
	- Golang
	- Golang Internal
	- Channel
categories: Golang
thumbnailImage: gopher_channel.png
---

{% asset_img "gopher_channel.png" %}

左书祺老师的[《Go语言设计与实现》](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/)写得很好，不过在阅读过程中发现不少部分还是要结合阅读源码才能够理解其中细节。

与上篇计时器类似，本篇将围绕 [6.4 章 Channel](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/) 通过查看资料和阅读源码补充的部分，以免自己回头忘记。

也欢迎熟悉这部分源码或是感兴趣的老板参与讨论。

基于阅读的 Golang 源码版本是 1.16 

<!-- toc -->

# 1. 数据结构

这部分针对书中 6.4.2 的章节部分，其中介绍了 Channel 的关键结构体 `runtime.hchan` ：

```golang
type hchan struct {
	qcount   uint           // total data in the queue
	dataqsiz uint           // size of the circular queue
	buf      unsafe.Pointer // points to an array of dataqsiz elements
	elemsize uint16
	closed   uint32
	elemtype *_type // element type
	sendx    uint   // send index
	recvx    uint   // receive index
	recvq    waitq  // list of recv waiters
	sendq    waitq  // list of send waiters

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}
```

书中大部分的字段都介绍到了，不过要明确的一点是，`dataqsiz` 事实上就是 `buf` 指向的数组的元素数量，亦即缓冲区的长度，对应 `make(chan int, size)` 的 `size`，在后面源码中也有所体现。

而 `buf` 指向的是个连续内存分配的数组，但在实现缓冲区的过程中，它利用 `sendx` 与 `recvx` 字段实现了基于数组的循环队列数据结构， `sendx` 与 `recvx` 分别维护了对应队列写入与获取的下标，在后文分析 `chansend` 与 `chanrecv` 代码时就能看到对应的体现。这么设计的目的应该就是书中提到的，为了实现一个高效的 FIFO 缓冲区队列，来保证先发送进入缓冲区的能够被先接收到。

而 `recvq` 与 `sendq` 对应的 `waitq` 也是 FIFO 的队列结构，只不过它是用链表的结构来维护的。




# 2. 创建 channel

对应书中章节 6.4.3 。Channel 的关键函数就是 `runtimme.makechan` 。 书中对于 `makechan` 部分展示了核心代码（`src/runtime/chan.go`），去掉调试代码和一些异常检查的代码：

```golang
func makechan(t *chantype, size int) *hchan {
	elem := t.elem
	mem, _ := math.MulUintptr(elem.size, uintptr(size))

	var c *hchan
	switch {
	case mem == 0:
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		c.buf = c.raceaddr()
	case elem.kind&kindNoPointers != 0:
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}
	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	return c
}
```

这里的 `math.MulUintptr` 顾名思义就是针对 `uintptr` 类型的乘法。

```golang
// MulUintptr returns a * b and whether the multiplication overflowed.
// On supported platforms this is an intrinsic lowered by the compiler.
func MulUintptr(a, b uintptr) (uintptr, bool) {
	if a|b < 1<<(4*sys.PtrSize) || a == 0 {
		return a * b, false
	}
	overflow := b > MaxUintptr/a
	return a * b, overflow
}
```

这里看到第二个返回值是代表是否溢出的布尔值，事实上真正的 `makechan` 源码对于这个 `overflow` 标志是接收并处理异常情况的。而书中的代码因为这部分不是重点所以省略了。

回到 `makechan` ，这里的 `math.MulUintpre(elem.size, uintptr(size))` 就相当于 `elem.size * size`，计算了缓冲区部分所需要的内存大小。

再看 `switch...case...` 部分，其中当 `mem == 0` 时， `c.buf = c.raceaddr()` 。书中并没有解释这行代码，只是说这段代码只为 `runtime.hchan` 分配了一块内存空间。

让我们看下 `raceaddr` 方法的源码（`src/unsafe/unsafe.go`）：

```golang
func (c *hchan) raceaddr() unsafe.Pointer {
	// Treat read-like and write-like operations on the channel to
	// happen at this address. Avoid using the address of qcount
	// or dataqsiz, because the len() and cap() builtins read
	// those addresses, and we don't want them racing with
	// operations like close().
	return unsafe.Pointer(&c.buf)
}
```

这个函数本身的作用其实是竞争检测时将对于通道的读操作与写操作视为对 `c.buf` 地址上的操作，用在作为 `raceacquire` 函数的参数，在`makechan`这里这个函数只是为了返回 `c.buf` 自身的地址。 要注意到在前面的 `mallocgc` 调用中由于是给整个结构体分配了内存，所以 `c.buf` 本身作为 `unsafe.Pointer` 类型也是被分配了内存的（虽然对应值是零值），所以返回 `&c.buf` 是会有一个内存地址的，虽然指向的就是 `c.buf` 本身。

后面一个 `case` 与 `default` 对应了 `elem` 是否是指针类型的两种情况，之所以要这么区分，其实源码中这段代码的注释给出了解释：

```golang
	// Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
	// buf points into the same allocation, elemtype is persistent.
	// SudoG's are referenced from their owning thread so they can't be collected.
    // 当存储在 buf 中的元素不包含指针时，Hchan 不包含对垃圾回收器（GC）有用的指针。
    // buf 指向对应同一内存分配，elemtype 是持久化的。
    // SudoG 被其所属线程引用，因此不会被回收。
```

也就是说，当 `elem` 不是指针类型的时候，`buf` 会指向与创建 `hchan` 时分配的同一内存块，然后其他指针都是不应会被当前 GC 关心的指针。

所以这部分代码就会有 `c = (* hchan)(mallocgc(hchanSize + mem, nil ,true))` ，一次性给 hchan 和 buf 创建连续的内存块，并且不需要指明这次分配内存的 `type`，对应 `mallocgc` 的第二个参数为 `nil`，也对应了 `mallocgc` 内部使用 `tiny allocator` 的方案来分配内存；然后 `c.buf = add(unsafe.Pointer(c), hchanSize)` 在分配完之后直接让 `c.buf` 的指针跳到这块内存 `hchan` 之后，指向分配给 `c.buf` 的部分。

而 `default` 的代码块代表对应 channel 的元素数据类型是指针类型的，因此 `hchan` 与 `buf` 的内存是分开分配的。同时在给 `buf` 调用 `mallocgc` 时指明了元素的类型 `elem`。

值得一提的是，`new` 关键字在书中 [5.5.2 一节](https://draveness.me/golang/docs/part2-foundation/ch05-keyword/golang-make-and-new/#552-new) 已经进行了介绍，`new` 会编译器转换成 `newobject` 函数，而 `newobject` 本质上还是调用 `mallocgc`: 

```golang
// implementation of new builtin
// compiler (both frontend and SSA backend) knows the signature
// of this function
func newobject(typ *_type) unsafe.Pointer {
	return mallocgc(typ.size, typ, true)
}
```

至于 `mallocgc` 是 Go 内存管理中分配内存的关键函数，后面分析内存管理的时候会专门整理一下。

# 3. 发送数据

对应章节 6.4.4 ， 介绍了 `ch <- i` 对应通道发送数据语句实际底层的运作机制，主要介绍的就是 channel 发送数据的核心函数 `chansend`。

```golang
/*
 * generic single channel send/recv
 * If block is not nil,
 * then the protocol will not
 * sleep but return if it could
 * not complete.
 *
 * sleep can wake up with g.param == nil
 * when a channel involved in the sleep has
 * been closed.  it is easiest to loop and re-run
 * the operation; we'll see that it's now closed.
 */
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	// ...
}
```

对于函数参数，有一个参数书中没有解释 —— `callerpc`，这个参数通过 `getcallerpc()` 获得当前所在函数的调用代码的 `pc` 程序计数器对应的地址，在竞争检测对应函数 `racereadpc` 中会用到，如果不开启竞争检测这个参数就无所谓了。

另外值得注意的是书中提到当 `c.closed != 0` 时，代表 channel 已经关闭，而程序调用 `chansend` 说明仍想给这个通道发送数据，因此直接 `panic` 直接报错，这段代码也呼应了关于 channel 的一个知识点：**向关闭通道发送数据会直接报错**


书中将 `chansend` 的源码分成了三部分来讲。在这三部分之前，除去前面竞争检测代码以及异常处理代码，有个函数中前面的代码可以提一下：

```golang
	// Fast path: check for failed non-blocking operation without acquiring the lock.
	//
	// After observing that the channel is not closed, we observe that the channel is
	// not ready for sending. Each of these observations is a single word-sized read
	// (first c.closed and second full()).
	// Because a closed channel cannot transition from 'ready for sending' to
	// 'not ready for sending', even if the channel is closed between the two observations,
	// they imply a moment between the two when the channel was both not yet closed
	// and not ready for sending. We behave as if we observed the channel at that moment,
	// and report that the send cannot proceed.
	//
	// It is okay if the reads are reordered here: if we observe that the channel is not
	// ready for sending and then observe that it is not closed, that implies that the
	// channel wasn't closed during the first observation. However, nothing here
	// guarantees forward progress. We rely on the side effects of lock release in
	// chanrecv() and closechan() to update this thread's view of c.closed and full().
	if !block && c.closed == 0 && full(c) {
		return false
	}
```

这个判断在获取锁之前进行，也就是如果这次 send 非阻塞、 通道未关闭、且通道缓冲区已满，则不获取锁直接返回 false。`full` 函数本质是返回了 `c.qcount == c.dataqsiz` 的比较值

注释解释了这么做可行的原因，因为关闭了的通道已经无法从“准备好发送”状态切换到“没准备好发送”的状态，当依次判定 `c.closed == 0` 与 `full(c)` 都为 false 时，证明这段时间内在某个瞬间这个通道没有就绪同时已经关闭了，我们认为自己行为上就是在这个瞬间看到了通道的状态，并且直接返回 false 代表数据无法发送。

下面我们也按照书中的划分，分别看下 `chansend` 三部分代码：

## 3.1 直接发送

```golang
	if sg := c.recvq.dequeue(); sg != nil {
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}
```

这段代码说明在给 channel 发送数据的时候，是优先检查接收者 goroutine 等待队列的，如果队列里有接收者就直接发送数据，不管 channel buffer。

不过理论上当 `recvq` 有 `sudog` 时，应该就是缓冲区队列为空的时，不然当时接收方应该会直接从缓冲区里拿数据而不是在等待队列里等待，所以 `chansend` 直接数据发送给接收方协程而不看 buffer 是 ok 的。

`c.recvq` 对应 `waitq`，是自己实现的 FIFO 队列，`dequeue` 方法会把队列队首元素弹出，对应 `enqueue` 方法则把元素插到队尾。

从 `recvq` 获取到 `sg` 后，代码就会调用 `send` （`src/runtime/chan.go`）函数：

```golang
// send processes a send operation on an empty channel c.
// The value ep sent by the sender is copied to the receiver sg.
// The receiver is then woken up to go on its merry way.
// Channel c must be empty and locked.  send unlocks c with unlockf.
// sg must already be dequeued from c.
// ep must be non-nil and point to the heap or the caller's stack.
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	// 竞争检测相关代码，可忽略
	if raceenabled {
		if c.dataqsiz == 0 {
			racesync(c, sg)
		} else {
			// Pretend we go through the buffer, even though
			// we copy directly. Note that we need to increment
			// the head/tail locations only when raceenabled.
			qp := chanbuf(c, c.recvx)
			racereleaseacquire(qp)
			racereleaseacquireg(sg.g, qp)
			c.recvx++
			if c.recvx == c.dataqsiz {
				c.recvx = 0
			}
			c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
		}
	}
	if sg.elem != nil {
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	gp := sg.g
	unlockf() // 利用传入的闭包函数解锁
	gp.param = unsafe.Pointer(sg)
	sg.success = true
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()        // 获取一个 cpu 频率时钟作为当前时间
	}
	goready(gp, skip+1)
}
```

书中代码除了去掉了竞争检测的代码部分以及设置 `sg.releasetime` 的代码部分，基本和源码一致。

要进一步理解 `send` 函数，就要看两个重要的调用， `sendDirect` 与 `goready`

### 3.1.1 sendDirect



```golang
// Sends and receives on unbuffered or empty-buffered channels are the
// only operations where one running goroutine writes to the stack of
// another running goroutine. The GC assumes that stack writes only
// happen when the goroutine is running and are only done by that
// goroutine. Using a write barrier is sufficient to make up for
// violating that assumption, but the write barrier has to work.
// typedmemmove will call bulkBarrierPreWrite, but the target bytes
// are not in the heap, so that will not help. We arrange to call
// memmove and typeBitsBulkBarrier instead.

func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
	// src is on our stack, dst is a slot on another stack.

	// Once we read sg.elem out of sg, it will no longer
	// be updated if the destination's stack gets copied (shrunk).
	// So make sure that no preemption points can happen between read & use.
	dst := sg.elem
	typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)
	// No need for cgo write barrier checks because dst is always
	// Go memory.
	memmove(dst, src, t.size)
}

```

这里的 `sendDirect` （`src/runtime/chan.go`） 可以看到调用了 `typeBitsBulkBarrier` 与 `memmove` ，这是因为 Go 的 GC 一般假定堆栈写入只会在 goroutine 运行时发生，且仅由该 goroutine 完成。

但 `sendDirect` 操作违反了这一假设，因此这里先调用了写屏障（write barrier， 对应 `typeBitsBulkBarrier`），再调用 `memmove` 将发送方元素地址（`ep`）上的数据复制到了接收者 goroutine 对应元素的地址（`sg.elem`）上。


### 3.1.2 goready 与 wakep

`goready` 涉及到 Go 调度器机制的重要函数。主要的作用是将目标 `goroutine` 标记为就绪状态（ready）。书中交代了部分后续，实际调度的情况是更复杂一点的。

先看下 `goready` （`src/runtime/proc.go`）以及对应内部调用 `ready` （`src/runtime/proc.go`）的源码：

```golang
func goready(gp *g, traceskip int) {
	systemstack(func() {
		ready(gp, traceskip, true)
	})
}

// Mark gp ready to run.
func ready(gp *g, traceskip int, next bool) {
	if trace.enabled {
		traceGoUnpark(gp, traceskip)
	}

	status := readgstatus(gp)

	// Mark runnable.
	_g_ := getg()
	mp := acquirem() // disable preemption because it can be holding p in a local var
	if status&^_Gscan != _Gwaiting {
		dumpgstatus(gp)
		throw("bad g->status in ready")
	}

	// status is Gwaiting or Gscanwaiting, make Grunnable and put on runq
	casgstatus(gp, _Gwaiting, _Grunnable)
	runqput(_g_.m.p.ptr(), gp, next)
	wakep()
	releasem(mp)
}



```

其中，`systemstack` 是会将栈切换到操作系统为线程分配的栈（而不是原本 goroutine 的“小栈”），再执行传入的函数，执行完之后再切回 goroutine 的栈。

我们再看下 `ready` 函数，首先参数 `traceskip` 是 trace 跟踪性能调试时用的参数，有关代码在本次分析时可忽略。

而 `ready` 函数首先将 g 的状态尝试设为 `Grunnable` ，并检查状态如果发生问题则抛出异常，然后如书中所说，通过 `runqput` 函数将`g` 放到自己这个 P （也就是书中所说的处理器） 的 `runnext` 字段中。

这里的 `runqput` 在给 P 本地存放 g 时遵循以下规则：

1. 如果入参 next 为 true ， 则将 g 放到 `p.runnext` 字段，作为下一个运行的 g，而原来放在 `p.runnext` 的 g 拿出来，按照第二条第三条规则处理。
2. 如果本地运行队列 `p.runq` （这是个循环队列）没有满，那么 g 加到 `p.runq` 队列的末尾
3. 如果 `p.runq` 满了，则调用 `runqputslow` 从本地队列中拿出一半的 goroutine，随机排列之后放入到调度的 goroutine 全局队列中

而事实上 `ready` 函数调用完 `runqput` 还没有结束，它还会调用 `wakep()` (`src/runtime/proc.go`)。 

```golang

// Tries to add one more P to execute G's.
// Called when a G is made runnable (newproc, ready).
func wakep() {
	if atomic.Load(&sched.npidle) == 0 {
		return
	}
	// be conservative about spinning threads
	if atomic.Load(&sched.nmspinning) != 0 || !atomic.Cas(&sched.nmspinning, 0, 1) {
		return
	}
	startm(nil, true)
}

```
`wakep` 会尝试唤醒一个空闲的 P，使它进入自旋状态（spinning），尝试去找一个就绪的 goroutine 并开始执行，具体的逻辑在 `startm` 里（因为要让 P 从空闲进入自旋，需要找到一个 M 即物理线程进行绑定）

而要让这个 `wakep` 函数真正执行，需要排除两个情况：

1. `sched.npidle == 0` ， 如果 `npidle` 为 0 ，说明目前没有空闲的 P，所有的 P 都在忙，那么直接退出。
2. `sched.nmspinning != 0` ， 代表当前已经存在一个以上正在自旋的 P，在不需要唤醒新的 P 的前提下，自己之前准备好的 g 也能被其他自旋 P 找到并执行，所以也退出。

如果不存在上述两种情况，那么 `wakep` 就会调用 `startm` (`src/runtime/proc.go`)

```golang
func startm(_p_ *p, spinning bool)
```

当传入的参数 `_p_=nil, spinning=true`时， `startm` 会首先调用 `pidleget` 在 `sched.pidle` 里找一个空闲的 P， 然后调用 `mget` 找一个空闲的 M，然后将 P 绑定给这个 M ，然后唤醒这个 M 对应的线程。

在这个过程中，如果 `mget` 没找到空闲 M，那么 `startm` 就会自己调用 `newm` 创建一个新的 M 来完成整个操作。

而如果通过 `wakep` 成功使一个 P 从空闲变为自旋，那么这个 P 会去找能够运行的 goroutine 来运行，这时候就有可能把我们当前 P 里的这个接收方的 goroutine 偷走。

综上来看，由于 `send` 会调用 `goready`，`goready` 在把接收方 goroutine 放入本地 P 运行队列后，会尝试唤醒一个新的空闲的 P 来运行准备就绪的 goroutine（可能会“偷取”接收者 goroutine），进而事实上这个接收者的 goroutine 是有可能与发送数据的 goroutine 并行执行的。

Go 调度器的部分还是挺有趣的，对于 channel 通信时 goroutine 的阻塞唤醒而言，并不像系统调用阻塞唤醒一样是 hand-off ， 即 M 找 P 的一个过程，而仍然是一个 G 唤醒 P 找 M 的一个过程。后续我会专门就 Go 调度进行分析，争取整理出全貌来。

## 3.2 缓冲区

这一部分就是当没有接收方 goroutine 时，`chansend` 会检查缓冲区是否已满，如果未满就会将发送的数据放入缓冲区，也就是这段代码：

```golang
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	// ...
	if c.qcount < c.dataqsiz {
		qp := chanbuf(c, c.sendx)
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}
	// ...
}
```

这里需要看下的就是 `chanbuf` 函数，其实是个非常简单的函数：

```golang
// chanbuf(c, i) is pointer to the i'th slot in the buffer.
func chanbuf(c *hchan, i uint) unsafe.Pointer {
	return add(c.buf, uintptr(i)*uintptr(c.elemsize))
}
```

`chanbuf` 基于下标 `c.sendx` 计算了指针偏移量 （`c.sendx * c.elemsize`），并返回了指向缓冲区中第 `c.sendx` 个元素的地址，并赋给了 `qp`。

获取到了 qp 之后， `chansend` 就通过 `typedmammove` 将 `ep` 复制到 `qp` ，同时增加 `c.sendx` 与 `c.qcount` ， 这里由于 `c.buf` 是个循环队列，当 `c.sendx` 处于队尾时，下一跳就会跳到队首。

## 3.3 阻塞发送

阻塞发送部分是在 `recvq` 中没有接收方 goroutine ， 缓冲区已满或者这个 channel 根本没有缓冲区时，发送方 goroutine 要进入阻塞等待另外的 goroutine 来接收数据。

这部分代码会分成三部分来分析（见注释划分），本身的核心还是在调度部分（`gopark`）：

```golang
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	// ...
	// 第一部分，非阻塞直接返回，将 goroutine 转化为 sudog 加入 c.sendq 队列
	if !block {
		unlock(&c.lock)
		return false
	}

	// Block on the channel. Some receiver will complete our operation for us.
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg)
	// Signal to anyone trying to shrink our stack that we're about
	// to park on a channel. The window between when this G's status
	// changes and when we set gp.activeStackChans is not safe for
	// stack shrinking.
	atomic.Store8(&gp.parkingOnChan, 1)

	// 第二部分， gopark
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)

	// 第三部分，唤醒后的处理
	// Ensure the value being sent is kept alive until the
	// receiver copies it out. The sudog has a pointer to the
	// stack object, but sudogs aren't considered as roots of the
	// stack tracer.
	KeepAlive(ep)

	// someone woke us up.
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	closed := !mysg.success
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
	releaseSudog(mysg)
	if closed {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	return true
```

### 3.3.1 加入 c.senq 队列

这部分代码首先会做一个的判断，也就是 `block` 是否为 false，若为 false 则说明这次 `chansend` 的调用是非阻塞的（比如是一定条件下`select ... default ...` 里的发送），那么就不走后面代码了，直接退出返回 false。

然后通过 `gp := getg()` 获取当前的 goroutine， 再通过 `mysg := acquireSudog()` 获取一个 `sudog` 的结构体。

这里 `acquireSudog` 多说两句：

```golang
//go:nosplit
func acquireSudog() *sudog {
	// Delicate dance: the semaphore implementation calls
	// acquireSudog, acquireSudog calls new(sudog),
	// new calls malloc, malloc can call the garbage collector,
	// and the garbage collector calls the semaphore implementation
	// in stopTheWorld.
	// Break the cycle by doing acquirem/releasem around new(sudog).
	// The acquirem/releasem increments m.locks during new(sudog),
	// which keeps the garbage collector from being invoked.
	mp := acquirem()
	pp := mp.p.ptr()
	if len(pp.sudogcache) == 0 {
		lock(&sched.sudoglock)
		// First, try to grab a batch from central cache.
		for len(pp.sudogcache) < cap(pp.sudogcache)/2 && sched.sudogcache != nil {
			s := sched.sudogcache
			sched.sudogcache = s.next
			s.next = nil
			pp.sudogcache = append(pp.sudogcache, s)
		}
		unlock(&sched.sudoglock)
		// If the central cache is empty, allocate a new one.
		if len(pp.sudogcache) == 0 {
			pp.sudogcache = append(pp.sudogcache, new(sudog))
		}
	}
	n := len(pp.sudogcache)
	s := pp.sudogcache[n-1]
	pp.sudogcache[n-1] = nil
	pp.sudogcache = pp.sudogcache[:n-1]
	if s.elem != nil {
		throw("acquireSudog: found s.elem != nil in cache")
	}
	releasem(mp)
	return s
}
```

这里 `acquireSudog` 函数的注释说明了缘由，一个主要原因是信号量实现调用了 `acquireSudog` ，`acquireSudog` 调用了 `new(sudog)`，`new` 调用 `malloc`， `malloc` 可能会调用垃圾回收，而 垃圾回收在 `stopTheWorld` 中调用了信号量实现。
因此使用了 `acquirem` / `releasem` 来包裹 `new(sudog)` ，避免垃圾回收器的调用。

此外， `acquireSudog` 实现了一个缓存机制，当要获取 `sudog` 时会先从缓存中取（`pp.sudogcache` , `sched.sudogcache` ），如果缓存里没有再 `new`，节省了内存分配的开销。对应的函数 `releaseSudog` 则是将用完的 `sudog` 结构体放回 cache 里。

我们再看下书中数据结构一节中没有介绍的 `sudog` 结构体的结构：

```golang
// sudog represents a g in a wait list, such as for sending/receiving
// on a channel.
//
// sudog is necessary because the g ↔ synchronization object relation
// is many-to-many. A g can be on many wait lists, so there may be
// many sudogs for one g; and many gs may be waiting on the same
// synchronization object, so there may be many sudogs for one object.
//
// sudogs are allocated from a special pool. Use acquireSudog and
// releaseSudog to allocate and free them.
type sudog struct {
	// The following fields are protected by the hchan.lock of the
	// channel this sudog is blocking on. shrinkstack depends on
	// this for sudogs involved in channel ops.

	g *g

	next *sudog
	prev *sudog
	elem unsafe.Pointer // data element (may point to stack)

	// The following fields are never accessed concurrently.
	// For channels, waitlink is only accessed by g.
	// For semaphores, all fields (including the ones above)
	// are only accessed when holding a semaRoot lock.

	acquiretime int64
	releasetime int64
	ticket      uint32

	// isSelect indicates g is participating in a select, so
	// g.selectDone must be CAS'd to win the wake-up race.
	isSelect bool

	// success indicates whether communication over channel c
	// succeeded. It is true if the goroutine was awoken because a
	// value was delivered over channel c, and false if awoken
	// because c was closed.
	success bool

	parent   *sudog // semaRoot binary tree
	waitlink *sudog // g.waiting list or semaRoot
	waittail *sudog // semaRoot
	c        *hchan // channel
}
```

因为信号量实现也使用到 `sudog` ，所以我们这边需要关心与这次 channel 发送相关的字段： `g` 、 `elem` 、`releasetime`、`isSelect`、`c`

通过 `acquireSudog` 获取 `sudog` 之后， `chansend` 函数会对 `gp` 与 `mysg` 的字段进行更新，作为阻塞的准备：

```golang
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
```

注意这里的 `mysg.elem` 被赋值了 `ep`，也就是这次要发送的数据。

然后就将 `mysg` 通过 `c.sendq.enqueue` 加入到发送等待队列里。

### 3.3.2 gopark

```golang
// Puts the current goroutine into a waiting state and calls unlockf on the
// system stack.
//
// If unlockf returns false, the goroutine is resumed.
//
// unlockf must not access this G's stack, as it may be moved between
// the call to gopark and the call to unlockf.
//
// Note that because unlockf is called after putting the G into a waiting
// state, the G may have already been readied by the time unlockf is called
// unless there is external synchronization preventing the G from being
// readied. If unlockf returns false, it must guarantee that the G cannot be
// externally readied.
//
// Reason explains why the goroutine has been parked. It is displayed in stack
// traces and heap dumps. Reasons should be unique and descriptive. Do not
// re-use reasons, add new ones.
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceEv byte, traceskip int) {
	if reason != waitReasonSleep {
		checkTimeouts() // timeouts may expire while two goroutines keep the scheduler busy
	}
	mp := acquirem()
	gp := mp.curg
	status := readgstatus(gp)
	if status != _Grunning && status != _Gscanrunning {
		throw("gopark: bad g status")
	}
	mp.waitlock = lock
	mp.waitunlockf = unlockf
	gp.waitreason = reason
	mp.waittraceev = traceEv
	mp.waittraceskip = traceskip
	releasem(mp)
	// can't do anything that might move the G between Ms here.
	mcall(park_m)
}
```

`gopark` 函数（`src/runtime/proc.go`）本身看不大出什么，入参 `reason` 是说明 park 的理由的（在 stack trace 和 heap dump 作为说明）， `traceEv` 与 `traceskip` 都是 trace 相关参数都可以不管，接下来我们重点看下 `unlockf`、`lock` ，在通过 `mcall` 调用 `park_m` 时，运行时会切换到 `g0` ， 然后这时候 g 的状态已经被切换为 `waiting`，当前 `g0` 对应栈是系统栈了，这时候再调用 `unlockf` 函数，传入这次进入等待状态的 `g` 和 `lock` 作为参数，以实现对 `lock` 的解锁。

回过头看 `chansend` 中 `gopark` 的调用，传入的是 `chanparkcommit` 函数和 channel 的锁 `unsafe.Pointer(&c.lock)`，我们再看 `chanparkcommit` (`src/runtime/chan.go`) 这个函数：

```golang
func chanparkcommit(gp *g, chanLock unsafe.Pointer) bool {
	// There are unlocked sudogs that point into gp's stack. Stack
	// copying must lock the channels of those sudogs.
	// Set activeStackChans here instead of before we try parking
	// because we could self-deadlock in stack growth on the
	// channel lock.
	gp.activeStackChans = true
	// Mark that it's safe for stack shrinking to occur now,
	// because any thread acquiring this G's stack for shrinking
	// is guaranteed to observe activeStackChans after this store.
	atomic.Store8(&gp.parkingOnChan, 0)
	// Make sure we unlock after setting activeStackChans and
	// unsetting parkingOnChan. The moment we unlock chanLock
	// we risk gp getting readied by a channel operation and
	// so gp could continue running before everything before
	// the unlock is visible (even to gp itself).
	unlock((*mutex)(chanLock))
	return true
}
```
`gp.activeStackChans` 与 `gp.parkingOnChan` 处理栈收缩与栈增长情况这里不展开， 这里的核心就是 `unlock(chanLock)` 对 `c.lock` 解锁。

在 `gopark` 中， `chanparkcommit` 作为入参已经传给了 `mp.waitunlockf` ，我们后面可以看到它的调用。

`gopark` 其他没啥好讲的（`checkTimeouts`看源码好像大部分操作系统实现里是空的，可以忽略），主要就是获取 M （`mp`）与 G（`g`），然后设置下 `lock`、`unlockf`、`reason`。 重点在 `mcall(park_m)` 这个调用，我们先看下 `mcall` （`src/runtime/stubs.go`）的注释：

```golang
// mcall switches from the g to the g0 stack and invokes fn(g),
// where g is the goroutine that made the call.
// mcall saves g's current PC/SP in g->sched so that it can be restored later.
// It is up to fn to arrange for that later execution, typically by recording
// g in a data structure, causing something to call ready(g) later.
// mcall returns to the original goroutine g later, when g has been rescheduled.
// fn must not return at all; typically it ends by calling schedule, to let the m
// run other goroutines.
//
// mcall can only be called from g stacks (not g0, not gsignal).
//
// This must NOT be go:noescape: if fn is a stack-allocated closure,
// fn puts g on a run queue, and g executes before fn returns, the
// closure will be invalidated while it is still executing.
func mcall(fn func(*g))

```
就如刚才讲的，`mcall` 会从调用 `mcall` 的 `g` 切换到 `g0` ， 并在 `g0` 调用 `fn(g)`。

`mcall` 将 `g` 的当前 PC/SP 保存在 `g->sched` 中，以便稍后恢复。 `fn` 负责安排稍后的执行，通常是通过将 `g` 记录在数据结构中（在这里就是 `c.sendq` 的 `sudog.g` 里了）， 以便稍后由某些调用 `ready(g)` 的操作来重新调度 `g`。 

`mcall` 在 `g` 被重新调度后返回到原来的 `goroutine g`。`fn` 绝对不能返回；通常它会通过调用 `schedule` 结束，以让 `m` 运行其他 `goroutine`。

我们再去看 `mcall` 这次对应 `fn` 的入参 `park_m` ，就更清晰了：

```golang
// park continuation on g0.
func park_m(gp *g) {
	_g_ := getg()

	if trace.enabled {
		traceGoPark(_g_.m.waittraceev, _g_.m.waittraceskip)
	}

	casgstatus(gp, _Grunning, _Gwaiting)
	dropg()

	if fn := _g_.m.waitunlockf; fn != nil {
		ok := fn(gp, _g_.m.waitlock)
		_g_.m.waitunlockf = nil
		_g_.m.waitlock = nil
		if !ok {
			if trace.enabled {
				traceGoUnpark(gp, 2)
			}
			casgstatus(gp, _Gwaiting, _Grunnable)
			execute(gp, true) // Schedule it back, never returns.
		}
	}
	schedule()
}
```

这里的 `_g_` 就是 `g0` ，它会将 `gp` 也就是我们本来运行着的 `g` 从 `Grunning` 状态切换为 `Gwaiting` 状态，然后调用 `dropg()`， 这个函数会将<strong> `gp` 对应的 `g` 与 M 分离</strong>：

```golang
// dropg removes the association between m and the current goroutine m->curg (gp for short).
// Typically a caller sets gp's status away from Grunning and then
// immediately calls dropg to finish the job. The caller is also responsible
// for arranging that gp will be restarted using ready at an
// appropriate time. After calling dropg and arranging for gp to be
// readied later, the caller can do other work but eventually should
// call schedule to restart the scheduling of goroutines on this m.
func dropg() {
	_g_ := getg()

	setMNoWB(&_g_.m.curg.m, nil)
	setGNoWB(&_g_.m.curg, nil)
}
```

这是个很重要的动作，这时候原先发送数据并在通道阻塞的 `g` 和 M 分离了，而<strong>这个 `g` 被存放在 `channel c` 的 `sendq` 对应 `sudog` 的 `g` 字段中，等待后续接收方 `g` 从 channel 的 `sendq` 中找到它，并接收它给的数据，并重新唤醒使它 runnable</strong>。

在 `dropg` 之后，就是从 `_g_.m.waitunlockf` ，并调用 `_g_.m.waitunlockf(gp, _g_.m.waitlock)`，其实就是之前 `gopark` 传的 `chanparkcommit` 与 `&c.lock` ，来解锁 `c.lock` 。这里可以看到，如果调用不 ok， 则将 `gp` 改回 `Grunnable` ，并且继续执行 `gp` 协程，相当于一个回滚操作了。

而一般情况下，最后就会调用 `schedule` ，由于 `gp` 已经与 M 解绑，那么就要 `g0` 就要为 M 找一个新的 goroutine 来运行了。

可以看到，goroutine 的 `park` 与 `ready` 是用户态的一种行为，发送方 `goroutine` “阻塞”了之后， M 就会找新的 goroutine 来运行。Channel 对应 goroutine 的调度可能是不少讲解 G-M-P 模型以及 Go 调度的教程没有覆盖到的地方（https://www.bilibili.com/video/BV19r4y1w7Nx?p=16）。


### 3.3.3 第三部分，唤醒后的处理

在被唤醒后，goroutine 会继续执行 `gopark` 之后的代码，`gopark` 之后会调用 `runtime.Keepalive(ep)` ，这是一个神奇的魔法方法。

具体可以见 [https://zhuanlan.zhihu.com/p/213744309](https://zhuanlan.zhihu.com/p/213744309) ， 它的作用是使作为入参的变量保持 '存活'，这就意味着它（或者它引用的变量）不会被垃圾收集，而且它所注册的任何终止器（finalizer）都不会被执行。 当代码中使用了 `runtime.KeepAlive()`，Go 编译器会设置一个名为 `OpKeepAlive` 的静态单赋值(SSA)，然后剩余的编译就会知道将这个变量的存活期保证到使用了 `runtime.KeepAlive()` 的时刻。

在 `chansend` 调用 `Keepalive` 的注释也提到了：

```golang
	// Ensure the value being sent is kept alive until the
	// receiver copies it out. The sudog has a pointer to the
	// stack object, but sudogs aren't considered as roots of the
	// stack tracer.
	// 确保被发送的值在接收者复制它之前保持存活。
	// sudog 有一个指向堆栈对象的指针，但 sudog 不被视为
	// 栈跟踪器的根。
```

然后是一些收尾工作，将一些 `gp` 的属性置为零并且通过 `releaseSudog` 释放 `sudog` 。

值得一提的是 `closed := !mysg.success`，这里的 `mysg.success` 其实在之前介绍的 【3.1 直接发送】章节 `send` 函数源码中有出现在发送成功后将  `sg.success = true` 设为成功，而在后面 `recv` 函数中也同样会把 `sg.success` 设为 true，而在 `closechan` 对应关闭 channel 时，也会将所有 `sendq` 与 `recvq` 里 `sudog` 的 `success` 设为 `false`，因此这是一个代表通道是否关闭的标志位。

正常而言接收方调用 `recv` 函数成功，`mysg.success` 应当为 true 才对，而当 `mysg.success` 为 false 时，说明通道关闭了，而这时候我们向一个关闭的通道发送数据，我们就要抛出 panic 报错。

### 3.3.4 总结

 `chansend` 的代码整体看下来，我觉得收获最多的就是对应 `goready` 与 `gopark` 的部分，代表了 channel 发送过程中 goroutine 的调度切换过程。后面结合 channel 接收部分的代码，就可以形成闭环，理解 channel 相关的 goroutine 流转的过程。

# 4. 接收数据

 接下来分析与发送相对的 channel 接收数据操作 `i <- ch` / `i, ok <- ch`。对应书中章节 6.4.5。

 书中讲了 `i<-ch` 会转换成 `runtime.chanrev1` ， `i, ok <- ch` 会转换成 `runtime.chanrecv2` ：

 ```go
// entry points for <- c from compiled code
//go:nosplit
func chanrecv1(c *hchan, elem unsafe.Pointer) {
	chanrecv(c, elem, true)
}

//go:nosplit
func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
	_, received = chanrecv(c, elem, true)
	return
}
 ```

 看源码确实就是调用 `chanrecv` ，所以重点还是在 `chanrecv`。

 `chanrecv` 的逻辑与 `chansend` 相对应，非常类似，很多可以对照着看，一开始的时候也有对于如果 `c == nil` 就是 channel 为空指针的判定：

 ```golang
// chanrecv receives on channel c and writes the received data to ep.
// ep may be nil, in which case received data is ignored.
// If block == false and no elements are available, returns (false, false).
// Otherwise, if c is closed, zeros *ep and returns (true, false).
// Otherwise, fills in *ep with an element and returns (true, true).
// A non-nil ep must point to the heap or the caller's stack.
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	// raceenabled: don't need to check ep, as it is always on the stack
	// or is new memory allocated by reflect.

	if debugChan {
		print("chanrecv: chan=", c, "\n")
	}

	if c == nil {
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}
	//  ...
 ```

和 `chansend` 一样，如果 `c == nil` ，那么会调用 `gopark` ，由于这个 goroutine 没有加到任何 wait 队列中，因此这个 goroutine 应该会在这里一直阻塞而不会被唤醒。

同样，`chanrecv` 也有对于不获取锁快速检查通道的方式：

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	// ...
	// Fast path: check for failed non-blocking operation without acquiring the lock.
	if !block && empty(c) {
		// After observing that the channel is not ready for receiving, we observe whether the
		// channel is closed.
		//
		// Reordering of these checks could lead to incorrect behavior when racing with a close.
		// For example, if the channel was open and not empty, was closed, and then drained,
		// reordered reads could incorrectly indicate "open and empty". To prevent reordering,
		// we use atomic loads for both checks, and rely on emptying and closing to happen in
		// separate critical sections under the same lock.  This assumption fails when closing
		// an unbuffered channel with a blocked send, but that is an error condition anyway.
		if atomic.Load(&c.closed) == 0 {
			// Because a channel cannot be reopened, the later observation of the channel
			// being not closed implies that it was also not closed at the moment of the
			// first observation. We behave as if we observed the channel at that moment
			// and report that the receive cannot proceed.
			return
		}
		// The channel is irreversibly closed. Re-check whether the channel has any pending data
		// to receive, which could have arrived between the empty and closed checks above.
		// Sequential consistency is also required here, when racing with such a send.
		if empty(c) {
			// The channel is irreversibly closed and empty.
			if raceenabled {
				raceacquire(c.raceaddr())
			}
			if ep != nil {
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
	}
	// ...
}

```

如果通道非阻塞，那么如果该通道关闭了或是缓冲区为空，那么直接返回。

而后面如果发现 `c.closed != 0` 代表通道关闭，那么直接返回 `selected = true` 以及 `received = false` ，代表没有接收成功。

后面我们按照书中对于 `chanrecv` 代码的划分，看下直接接收、缓冲区、阻塞接收的代码

## 4.1 直接接收

直接接收的代码和直接发送的代码是很像的，这部分代码会从 `c.sendq` 里找一个 `sudog` ，如果找到，则调用 `recv` 接收对应的数据。

```go
	if sg := c.sendq.dequeue(); sg != nil {
		// Found a waiting sender. If buffer is size 0, receive value
		// directly from sender. Otherwise, receive from head of queue
		// and add sender's value to the tail of the queue (both map to
		// the same buffer slot because the queue is full).
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}
```

但如书中所写，包括这里代码注释也提到了，`recv` 的行为会根据是否有缓冲区而存在变化：

```go
// recv processes a receive operation on a full channel c.
// There are 2 parts:
// 1) The value sent by the sender sg is put into the channel
//    and the sender is woken up to go on its merry way.
// 2) The value received by the receiver (the current G) is
//    written to ep.
// For synchronous channels, both values are the same.
// For asynchronous channels, the receiver gets its data from
// the channel buffer and the sender's data is put in the
// channel buffer.
// Channel c must be full and locked. recv unlocks c with unlockf.
// sg must already be dequeued from c.
// A non-nil ep must point to the heap or the caller's stack.
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if c.dataqsiz == 0 {
		if raceenabled {
			racesync(c, sg)
		}
		if ep != nil {
			// copy data from sender
			recvDirect(c.elemtype, sg, ep)
		}
	} else {
		// Queue is full. Take the item at the
		// head of the queue. Make the sender enqueue
		// its item at the tail of the queue. Since the
		// queue is full, those are both the same slot.
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			racereleaseacquire(qp)
			racereleaseacquireg(sg.g, qp)
		}
		// copy data from queue to receiver
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		// copy data from sender to queue
		typedmemmove(c.elemtype, qp, sg.elem)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
	}
	sg.elem = nil
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	sg.success = true
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	goready(gp, skip+1)
}
```

如代码 `recv` 注释提到的， `recv` 只有在该通道没有缓冲区或者缓冲区为满的情况下才会被调用，因为只有这种情况下，在发送等待队列里才有 `sudog`。

在没有缓冲区的情况下（`c.dataqsiz == 0`），会比较简单，直接调用 `recvDirect` ：

```go
func recvDirect(t *_type, sg *sudog, dst unsafe.Pointer) {
	// dst is on our stack or the heap, src is on another stack.
	// The channel is locked, so src will not move during this
	// operation.
	src := sg.elem
	typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)
	memmove(dst, src, t.size)
}
```

将 `sg.elem` 的数据复制到 `ep` 指向的内存空间。

而**有缓冲区的情况**，首先通过 `chanbuf` 获取指向缓冲区第 `recvx` 个元素的指针，将这个元素数据复制到 `ep` 指向的空间。也就是取了目前 `c.buf` 循环队列中队首的元素给了 `ep`

然后再通过 `typedmemmove` 将 `sg.elem` 复制到 `ep` ，然后移动 `recvx++` 下标（同时取模 `c.dataqsiz`），这个时候其实 `ep` 指向的是循环队列的队尾，而 `recvx` 下标现在指向的就是循环队列的头（因为下一个会被拿出来的 `sudog` 就是 `recvx` 指向的元素）。

而 `c.sendx = c.recvx` 看似有点让人困惑，在这里需要结合后面从缓冲区接收数据的代码，当缓冲区满并从缓冲区接收数据时，`c.recvx` 指向的数据被复制出去，同时 `c.recvx ++ ` 移动 `recvx` 下标， 而 `c.sendx` 指向不变，这里 `c.sendx` 就是指向的是缓冲区里唯一一个可以放入发送数据的下标，亦即循环队列的队尾。

两个下标之所以要这么维护操作我觉得主要是为了通过循环队列是为了保证缓冲区数据接收的 FIFO 先进先出原则。

做完数据的接收复制后，就是一些对 `sg` 、 `gp` 更新数据的一些收尾动作，包括 `sg.success = true` 表明数据接收成功，然后就是和直接发送的代码一样，调用 `goready` （亦即调用 `runqput` 与 `wakep`）唤醒发送方 `sudog` 存放着的 g，这部分逻辑前文有分析，不再赘述了。

总之没缓冲区时 `sendq` 的 `sudog` 直接复制数据给 `ep`；有缓冲区时，`sendq` 有`sudog` 就是缓冲区满的情况，整个代码会将 `sudog` 的发送数据放到缓冲区队列末尾，将缓冲区队列头部的数据拿出来给到接收方。


## 4.2 缓冲区

```go
	if c.qcount > 0 {
		// Receive directly from queue
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			racereleaseacquire(qp)
		}
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}
```

和 `chansend` 缓冲区处理逻辑一样，这里也是根据 `c.recvx` 下标获取要接收的数据，通过 `typedmemmove` 将数据复制到 `ep` 指针指向的地址，调用 `typedmemclr` 清空缓冲区 `qp` 指向的数据，之后累加 `c.recvx` 下标并取模。

## 4.3 阻塞接收

```go
	if !block {
		unlock(&c.lock)
		return false, false
	}

	// no sender available: block on this channel.
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	c.recvq.enqueue(mysg)
	// Signal to anyone trying to shrink our stack that we're about
	// to park on a channel. The window between when this G's status
	// changes and when we set gp.activeStackChans is not safe for
	// stack shrinking.
	atomic.Store8(&gp.parkingOnChan, 1)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

	// someone woke us up
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	success := mysg.success
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, success
```

这部分逻辑基本和发送是相似的，`chanrecv` 将 g 包装成 `sudog` ，添加到 `c.recvq` 队列中，并调用 `gopark` 将该 g 与 M 解绑，等待被唤醒。

唤醒之后对应的 `ep`（`mysg.elem`）也收到了数据，就做些收尾工作返回出去。


# 5. 关闭 channel

```go
func closechan(c *hchan) {
	if c == nil {
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock)
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}

	if raceenabled {
		callerpc := getcallerpc()
		racewritepc(c.raceaddr(), callerpc, funcPC(closechan))
		racerelease(c.raceaddr())
	}

	c.closed = 1

	var glist gList

	// release all readers
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}

	// release all writers (they will panic)
	for {
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}
	unlock(&c.lock)

	// Ready all Gs now that we've dropped the channel lock.
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}

```

这部分 `closechan` 代码也很直观，主要就是对设置 channel 关闭状态 `c.closed = 1` ， 将 `sendq` 与 `recvq` 里的 `sudog` 都拿出来清理掉，同时把这些 `sudog` 对应的协程都通过 `goready` 唤醒，这些协程会根据 `chansend` 与 `chanrecv` 后续的代码基于 `c.closed != 0` 且 `sg.success != true` 这一情况进行处理：`chansend` 由于向关闭 channel 发送数据直接 panic，`chanrecv` 根据 `sg.success` 来返回最终接收的结果。


# 6. 总结

感觉自己把代码过了一遍之后确实很多细节清晰了不少。我觉得比较有趣的是关于 channel 发送接收时 goroutine 调度的部分，即  `goready` 与 `gopark` 的逻辑，以及 `c.buf` 循环队列的下标维护与处理。后面在整理 Go 调度器部分的时候如果发现这部分分析有问题，回头会进行勘误。

在创建 channel 的时候其实还涉及到了 gc 的部分，后面也统一整理，书中还提到了 dmitry 大佬实现过的无锁 channel 的 proposal，这次就没有研究了，以后有机会好好看看。

这篇文章更多是自己对于 Golang Channel 源码的学习与梳理，可读性比较差，里面可能存在较多错误，如果有老板觉得有问题请及时反馈，多谢！

