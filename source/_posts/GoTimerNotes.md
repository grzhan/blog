title: 【Go 语言设计与实现】 笔记 — 定时器源码分析
date: 2024-05-26 16:59:46
tags:
	- Golang
	- Golang Internal
	- 定时器
categories: Golang
thumbnailImage: gopher_clock.png
---

{% asset_img "gopher_clock.png" %}

左书祺老师的[《Go语言设计与实现》](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-timer)写得很好，不过在阅读过程中发现不少部分还是要结合阅读源码才能够理解其中细节。

因此写下这篇笔记，记录下《Go 语言设计与实现》 6.3 章通过查看资料和阅读源码补充的部分，以免自己回头忘记。

也欢迎熟悉这部分源码或是感兴趣的老板参与讨论。

基于阅读的 Golang 源码版本是 1.16 

<!-- toc -->

# 1. 数据结构

## 1.1 runtime.timer

在 [6.3.2 数据结构](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-timer/#632-%e6%95%b0%e6%8d%ae%e7%bb%93%e6%9e%84) 一节中，介绍了数据结构 runtime.timer 结构体。


```golang
type timer struct {
	pp puintptr

	when     int64
	period   int64
	f        func(interface{}, uintptr)
	arg      interface{}
	seq      uintptr
	nextwhen int64
	status   uint32
}
```

这里不介绍书中已经介绍的部分，下面会基于源码补充一些细节，也是为后面源码解读做铺垫。

### 1.1.1 pp

里面有个很重要的但是没介绍的字段，就是 `pp` ，在 `src/runtime/time.go` 可以看到在 timer 源码有关于 `pp` 的注释：

> // If this timer is on a heap, which P's heap it is on.
> // puintptr rather than \*p to match uintptr in the versions
> // of this struct defined in other packages.
> 
> // 如果这个定时器在堆上，它位于哪个P的堆上。
> // 使用 puintptr 而不是 *p 以匹配在其他包中定义的该结构体的 uintptr 版本。

也就是说这个 `pp` 代表的是该 `timer` 在四叉堆时所在的对应 P。而 P 则是 Golang 调度器 `G-M-P` 模型中的核心部分，也就是书中前文的 `runtime.p` ，在书中的[调度器一节](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/)中会详细介绍（大概我也会写篇笔记，因为调度器也是很有意思的内容）。后面这个 `timer.pp` 在后续源码会出现多次。

### 1.1.2 period

而另外一个字段 `period` ，书中介绍是 “两次被唤醒的间隔”，更具体地，可以看源码注释：

> // Timer wakes up at **when**, and then at **when+period**, ... (period > 0 only)
> // 定时器在 when 时唤醒，然后在 when + period 时唤醒，...（仅当 period > 0 时）

这说明两点：

1. `period` 代表的是一个周期，代表 timer 会在 **when**、**when + period** 、 **when + 2 * period** …… 唤醒，以此类推。
2. `period` 只会在其值大于 0 才会生效，为 0 时可忽略。



## 1.2 time.Timer 与 NewTimer

书中简单提了一嘴 `runtime.Timer` 与实际库中使用的 `time.Timer` 的关系。 在 `src/time/sleep.go` 我们可以通过源码看到更明确的关联， `time.Timer` 结构体如下：

```golang
// The Timer type represents a single event.
// When the Timer expires, the current time will be sent on C,
// unless the Timer was created by AfterFunc.
// A Timer must be created with NewTimer or AfterFunc.
type Timer struct {
	C <-chan Time
	r runtimeTimer
}
```

这里的 `runtimeTimer` 就对应我们内部的 `runtime.timer`，我们可以看下 `time.NewTimer` （`src/time/sleep.go`）的源码，看下创建 `time.Timer` 时，它如何与 `runtime.Timer` 关联：

```golang
// NewTimer creates a new Timer that will send
// the current time on its channel after at least duration d.
func NewTimer(d Duration) *Timer {
	c := make(chan Time, 1)
	t := &Timer{
		C: c,
		r: runtimeTimer{
			when: when(d),
			f:    sendTime,
			arg:  c,
		},
	}
	startTimer(&t.r)
	return t
}
```

看这一段我们发现创建 `time.Timer` 时， `NewTimer` 也同步创建了一个 `runtime.timer` ，并设置了如下字段：

+ `when` - 通过 `when(d)` 将 `now() + d` 转换成一个纳秒级的时间值（`int64`）给到该字段（后面会具体讲）
+ `f` - 函数 `sendTime` （后面具体讲）
+ `arg` - 新创建的缓冲通道 `c` (`chan Time`)

创建完 `t`（`time.Timer`） 后，函数就调用 `startTimer`，并返回结构体 `t` 。 下面我们分别看下 `when`、`sendTime`、`startTimer` ，`runtime.Timer` 作为底层如何实现 `NewTimer` 整个逻辑就很清晰了。

### 1.2.1 time.when(d Duration)

还是在 `src/time/sleep.go`，我们可以找到 `when` 函数实现：

```golang
// when is a helper function for setting the 'when' field of a runtimeTimer.
// It returns what the time will be, in nanoseconds, Duration d in the future.
// If d is negative, it is ignored. If the returned value would be less than
// zero because of an overflow, MaxInt64 is returned.
func when(d Duration) int64 {
	if d <= 0 {
		return runtimeNano()
	}
	t := runtimeNano() + int64(d)
	if t < 0 {
		t = 1<<63 - 1 // math.MaxInt64
	}
	return t
}

```

如注释所说， `when` 一个用于设置`runtimeTimer`的 'when' 字段的辅助函数。 它返回**将来 `Duration d` 时间之后的时间（以纳秒为单位）**。

 如果`d`是负数，则忽略它。如果由于溢出导致返回值小于零，则返回`MaxInt64`。

代码里的 `runtimeNano` 函数对应 `runtime.nanotime`，具体实现取决于运行的操作系统平台。

### 1.2.2 time.sendTime

在 `NewTimer` 中， `sendTime` 对应创建的 `runtimeTimer`的字段 `f` ，也就是在时间 `d` 之后会被 runtime 调用的函数。

`sendTime` （`src/time/sleep.go`）的源码如下：

```golang
func sendTime(c interface{}, seq uintptr) {
	// Non-blocking send of time on c.
	// Used in NewTimer, it cannot block anyway (buffer).
	// Used in NewTicker, dropping sends on the floor is
	// the desired behavior when the reader gets behind,
	// because the sends are periodic.
	select {
	case c.(chan Time) <- Now():
	default:
	}
}
```

该函数会向通道 `c` 传递 `Now()`的值，即当前时间。

此外，这个函数确保整个发送是非阻塞的（带 `default` 的 `select`）。

### 1.2.3 time.startTimer

这个函数（`src/time/sleep.go`）其实是对应 `runtime.startTimer`（`src/runtime/time.go`），二者通过 `go:linkname` 关联。

```golang
// startTimer adds t to the timer heap.
//
//go:linkname startTimer time.startTimer
func startTimer(t *timer) {
	if raceenabled {
		racerelease(unsafe.Pointer(t))
	}
	addtimer(t)
}
```

核心就是 `addtimer(t)` ，也就是把我们新创建的 `runtime.timer` 加入到当前 P 的 `timer` 四叉堆里去

`addtimer` 的逻辑在书中有讲，后文也会再进行些补充

### 1.2.4 NewTimer 逻辑

通过上述分析，`NewTimer`、`time.Timer` 与 `runtime.Timer` 互相之间的逻辑关系就很清晰了：

1. `NewTimer` 会创建一个 `time.Timer` t，并创建一个 `runtime.Timer` r 给到 t， r 会通过 `startTimer` 加入到 P 的 `timer` 四叉堆中，并在经过时间 d 之后被唤醒。函数`NewTimer`最后返回 t。
2. 在经过时间 d 之后，r 会被 runtime 唤醒，对应的 f （`sendTime`）会被调用，该函数会将当前时间（也就是经过 d 之后的时间）发送给通道 c
3. 外部 `NewTimer` 的调用方，可以通过等待接收通道 `time.Timer.C`发送的值，来实现经过时间 d 之后被唤醒

除了 `NewTimer` ， 像 `time.After` 本质上就是 `NewTimer` 的一个封装，直接把 `time.Timer` 的 Channel 返回出来了而已。

```golang
// After waits for the duration to elapse and then sends the current time
// on the returned channel.
// It is equivalent to NewTimer(d).C
// The underlying Timer is not recovered by the garbage collector
// until the timer fires. If efficiency is a concern, use NewTimer
// instead and call Timer.Stop if the timer is no longer needed.
func After(d Duration) <-chan Time {
	return NewTimer(d).C
}
```

`NewTimer` 与 `After` 作为常见 Go 的定时器应用场景，它们的逻辑体现了 `runtime.Timer` 起到的作用，了解这些我们就可以更好地看底层整个机制是如何运作的了。


# 2. 状态机

在 [6.3.3 状态机](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-timer/#633-%e7%8a%b6%e6%80%81%e6%9c%ba) 一节中，介绍了定时器的各种状态以及变更这些状态的函数。

下面会主要对这些函数的源码分析进行些补充。

## 2.1 addtimer 添加定时器

书中的 `addtimer` 源码并不是全部的，这里列出全部的代码（`src/runtime/time.go`）：

```golang
// addtimer adds a timer to the current P.
// This should only be called with a newly created timer.
// That avoids the risk of changing the when field of a timer in some P's heap,
// which could cause the heap to become unsorted.
func addtimer(t *timer) {
	// when must never be negative; otherwise runtimer will overflow
	// during its delta calculation and never expire other runtime timers.
	if t.when < 0 {
		t.when = maxWhen
	}
	if t.status != timerNoStatus {
		throw("addtimer called with initialized timer")
	}
	t.status = timerWaiting

	when := t.when

	pp := getg().m.p.ptr()  // 获取当前 goroutine 绑定的 M，进而获取该 M 对应的 P 的指针
	lock(&pp.timersLock)    // pp.timersLock 加锁，对应书中前文的 runtime.p 提到的 timersLock 字段
	cleantimers(pp)
	doaddtimer(pp, t)
	unlock(&pp.timersLock)

	wakeNetPoller(when)
}
```

值得注意的是书中没提到的两部分代码， 第一部分是 `pp`，在书中这段代码出现了 `pp`（`cleantimers` 与 `doaddtimer`的入参）但没有交代，实际上 `pp` 当前运行的协程对应的 P 的指针，在前面的代码中有所创建：

```golang
pp := getg().m.p.ptr()
```

这里的 `getg()` 会返回当前 goroutine 对应 g 结构体的指针，这里的 g 也是 `G-M-P` 模型的核心组成，代表 Goroutine 协程。得到 g 后再通过 g 获得目前与该协程绑定的 m，再通过 m 获得当前对应的 p 的指针。

而第二部分， `cleantimers` 与 `doaddtimer` 的调用是加了 `timersLock` 锁的。

关于 `timersLock` 锁，在书中介绍 `runtime.p` 的数据结构时有提到，而在 `runtime.p` 的源码中（`src/runtime/runtime2.go`），对于 `timers` 的注释有提到在访问四叉堆 `timers` 之前需要加 `timersLock` 锁：

```golang
type p struct {
	...
	// Lock for timers. We normally access the timers while running
	// on this P, but the scheduler can also do it from a different P.
	timersLock mutex

	// Actions to take at some time. This is used to implement the
	// standard library's time package.
	// Must hold timersLock to access.
	timers []*timer
	...
}

```

之所以要加锁，是因为 `p.timers` 不单单自己这个 P 运行时候会访问，调度器有时会让其他 P 来访问这个 timers，进而带来并发读写问题。

所以涉及到访问 timers 的代码都需要加锁，这也是为什么给 `cleantimers` 与 `doaddtimers` 加锁的原因。

下面我们深入下 `addtimers` 涉及到的几个函数来理解整个 `addtimers` 的逻辑，`cleantimers` 后面会有专门章节去分析，这里主要分析剩下两个：

+ `doaddtimer` - 真正把 `timer` t 加到 pp 的 timers 四叉堆里的函数
+ `wakeNetPoller` - 唤醒 netpoller 中休眠的线程，触发调度器检查是否有定时器到期


### 2.1.1 doaddtimer

```golang
// doaddtimer adds t to the current P's heap.
// The caller must have locked the timers for pp.
func doaddtimer(pp *p, t *timer) {
	// Timers rely on the network poller, so make sure the poller
	// has started.
	if netpollInited == 0 {
		netpollGenericInit()
	}

	if t.pp != 0 {
		throw("doaddtimer: P already set in timer")
	}
	t.pp.set(pp)
	i := len(pp.timers)
	pp.timers = append(pp.timers, t)
	siftupTimer(pp.timers, i)
	if t == pp.timers[0] {
		atomic.Store64(&pp.timer0When, uint64(t.when))
	}
	atomic.Xadd(&pp.numTimers, 1)
}
```

这里分析下上面这段代码：

1. 通过全局变量 `netpollInited` （`src/runtime/netpoll.go`）判断 network poller 是否已经初始化，如果没有，则调用 `netpollGernicInit` 进行初始化，`netpollGernicInit` 的 network poller 初始化逻辑涉及到对应操作系统平台的异步 IO 方案的初始化（Windows 里对应 [IOCP](https://learn.microsoft.com/en-us/windows/win32/fileio/i-o-completion-ports) 的 [CreateIoCompletionPort](https://learn.microsoft.com/en-us/windows/win32/fileio/createiocompletionport)），这里不展开。在书中 [6.6 网络轮询器](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-netpoller/) 一节提到这里也是 netpoll 初始化的触发点之一（另外就是 `os.newFile` 与 `net.netFD.init`）。
2. 如果 `t.pp != 0`，代表该 `timer` t 已经有所属的 P 了，因此报错退出
3. `t.pp.set(pp)` ， 将 `t.pp` 指向 `pp` 对应的 p
4. `pp.timers = append(pp.timers, t)` 将 t 添加到 `pp.timers` 的末尾；`siftupTimer` 则从 `pp.timers` 的末尾（i）开始，将 t 尝试按照四叉堆 sift up 操作进行上推，来维护整个最小堆。
5. 如果 `t == pp.timers[0]`， 代表 t 在四叉堆的最顶端，亦即 `t.when` 是目前整个堆的最小值，因此通过原子操作 `pp.timer0When` 设为 `t.when` ，以更新最小值。
6. 通过原子操作将 `pp.numTimers` 加一 


### 2.1.2 wakeNetPoller

```golang
// wakeNetPoller wakes up the thread sleeping in the network poller if it isn't
// going to wake up before the when argument; or it wakes an idle P to service
// timers and the network poller if there isn't one already.
func wakeNetPoller(when int64) {
	if atomic.Load64(&sched.lastpoll) == 0 {
		// In findrunnable we ensure that when polling the pollUntil
		// field is either zero or the time to which the current
		// poll is expected to run. This can have a spurious wakeup
		// but should never miss a wakeup.
		pollerPollUntil := int64(atomic.Load64(&sched.pollUntil))
		if pollerPollUntil == 0 || pollerPollUntil > when {
			netpollBreak()
		}
	} else {
		// There are no threads in the network poller, try to get
		// one there so it can handle new timers.
		wakep()
	}
}
```

wakeNetPoller 的代码涉及到调度器（sched）与网络轮询器 network poller 的机制，以后单独写分析详细展开。
这里简单过一下，如注释里介绍的一样，如果网络轮询器中睡眠的线程不会在 when 参数之前醒来 ，wakeNetPoller 会唤醒它；或者函数发现如果没有 P 在网络轮询器，则唤醒一个空闲的 P 去处理定时器和网络轮询器。

因此上面的函数代码也分为两部分， 首先是判断 `sched.lastpoll` 是否等于 0 。这里的 `sched`（`type schedt struct`） 是代表调度器的环境变量，其字段 `lastpoll` 表示最后一次网络轮询器轮询的时间，若为 0 则代表目前正在轮询。

若 `sched.lastpoll == 0`，代表目前网络轮询其正在轮询，这时会取 `sched.pollUntil` 的值，这个字段表示这次轮询会持续的时间。而在这里，我们判断如果 `pollerUntil` 为 0 或者 `pollerUntil` 的时间要比 `when` 的时间晚，则调用 `netpollBreak()`中断阻塞的网络轮询，同时唤醒对应线程，检查 `timers` 是否有定时器到期。

值得一提的是 `netpollBreak` 会触发异步 IO 的系统调用，在 Windows IOCP 对应 [PostQueuedCompletionStatus](https://learn.microsoft.com/en-us/windows/win32/fileio/postqueuedcompletionstatus) （`src/runtime/netpoll_windows.go`），它会向 IOCP 发一个 I/O completion 包，使轮询等待处理 IO 的线程从阻塞中唤醒。

而对应 [epoll](https://en.wikipedia.org/wiki/Epoll) 的网络轮询器(`src/runtime/netpoll_epoll.go`)对于 `netpollBreak` 的实现是类似的，它通过向 `netpollBreakWr` 传入一个字节的数据，通过通道传到 `netpollBreakRd`，`epoll_wait` 监听到 read pipe 上的 event，立即返回。当然 go 对于 netpoll 还有很多基于不同 IO 模型的实现，这里不展开了。

而当 `sched.lastpoll != 0` 代表没有线程在处理网络轮询器，那么就通过 `wakep` 调度一个新的 M （对应 `wakep` 中调用 `startm`，`startm` 是调度器中重要的函数，书中调度器一章会重点讲）去唤醒一个空闲的 P，进而去检查定时器是否就绪。


### 2.1.3 总结

稍微总结一下，`addtimer` 会将新的 `timer` t 加入到当前 P 的 `timers` 四叉堆中，并调整维护堆的次序以及 P 相关的定时器字段。同时会调用 `wakeNetPoller` 唤醒网络轮询器中休眠的线程，以检查是否有定时器到期。


## 2.2 deltimer 删除定时器

deltimer 书中没有展示源码，主要介绍了该函数会将入参 `timer` t 根据其不同的状态进行状态变换。

这里展开一下，分析下源码（`src/runtime/time.go`）：

```golang
// deltimer deletes the timer t. It may be on some other P, so we can't
// actually remove it from the timers heap. We can only mark it as deleted.
// It will be removed in due course by the P whose heap it is on.
// Reports whether the timer was removed before it was run.
func deltimer(t *timer) bool {
	for {
		switch s := atomic.Load(&t.status); s {
		case timerWaiting, timerModifiedLater:
			// Prevent preemption while the timer is in timerModifying.
			// This could lead to a self-deadlock. See #38070.
			mp := acquirem()
			if atomic.Cas(&t.status, s, timerModifying) {
				// Must fetch t.pp before changing status,
				// as cleantimers in another goroutine
				// can clear t.pp of a timerDeleted timer.
				tpp := t.pp.ptr()
				if !atomic.Cas(&t.status, timerModifying, timerDeleted) {
					badTimer()
				}
				releasem(mp)
				atomic.Xadd(&tpp.deletedTimers, 1)
				// Timer was not yet run.
				return true
			} else {
				releasem(mp)
			}
		case timerModifiedEarlier:
			// Prevent preemption while the timer is in timerModifying.
			// This could lead to a self-deadlock. See #38070.
			mp := acquirem()
			if atomic.Cas(&t.status, s, timerModifying) {
				// Must fetch t.pp before setting status
				// to timerDeleted.
				tpp := t.pp.ptr()
				atomic.Xadd(&tpp.adjustTimers, -1)
				if !atomic.Cas(&t.status, timerModifying, timerDeleted) {
					badTimer()
				}
				releasem(mp)
				atomic.Xadd(&tpp.deletedTimers, 1)
				// Timer was not yet run.
				return true
			} else {
				releasem(mp)
			}
		case timerDeleted, timerRemoving, timerRemoved:
			// Timer was already run.
			return false
		case timerRunning, timerMoving:
			// The timer is being run or moved, by a different P.
			// Wait for it to complete.
			osyield()
		case timerNoStatus:
			// Removing timer that was never added or
			// has already been run. Also see issue 21874.
			return false
		case timerModifying:
			// Simultaneous calls to deltimer and modtimer.
			// Wait for the other call to complete.
			osyield()
		default:
			badTimer()
		}
	}
}
```

注释里指出，之所以 `deltimer` 只是设置该 `timer` 的状态，而不是直接将它从对应所属 `runtime.p` 的 `timers` 中删除。是因为**这个函数是可能被不是这个 `timer` 所在的 P 调用的**，因此，当前 P 只会调用 `deltimer` 修改这个`timer` 的状态，真正的删除会让这个 `timer` 所属的 P 自己统一去做。

这段代码的基本逻辑就是通过原子操作获取 `t.status` 的状态， 然后通过 `switch ... case ...` 判断，根据不同状态然后通过 `atomic.Cas` CAS 原子操作进行状态切换。

这里有值得展开讲的几块内容。

### 2.2.1 acquirem 与 releasem

可以看到在每次变更 `t.status` 状态之前，都会调用 `acquirem()` ， 并在变更之后通过 `releasem(mp)` 释放。 `acquirem()` 会通过当前的 g 获取对应的 m （代表目前 goroutine 运行对应的线程），并将其返回，这个过程中会将内部的 `m.locks` +1 ：

```golang
//go:nosplit
func acquirem() *m {
	_g_ := getg()
	_g_.m.locks++
	return _g_.m
}
```

而 `releasem()` 则对应把 `m.locks` - 1 。之所以会添加这个过程是如注释提到的，是为了在 `timer` 状态更换的过程中避免调度抢占，进而避免这可能导致的死锁（见 [#38070](https://github.com/golang/go/issues/38070))，在函数 `canPreemptM` （`src/runtime/preempt.go`）中就能看到对于 `m.locks` 的判断：

```golang
// canPreemptM reports whether mp is in a state that is safe to preempt.
//
// It is nosplit because it has nosplit callers.
//
//go:nosplit
func canPreemptM(mp *m) bool {
	return mp.locks == 0 && mp.mallocing == 0 && mp.preemptoff == "" && mp.p.ptr().status == _Prunning
}
```

### 2.2.2 timerModifying 与 osyield

值得注意的是 `timeWaiting` 、`timerModifedEarlier` 、`timerModifiedLater` 这些状态都会首先转换成 `timerModifying` 再转换成 `timerDeleted`， 之所以会加入 `timerModifying` 这个中间过程，是为了考虑多个并发调用同时修改这个 `timer` 状态的情况（有点像是加了个锁）。

因此，当调用 `deltimer` 发现 `t.status` 为 `timerModifying` 状态时，说明这个 `timer` 在被其他调用方修改状态，这时候 `deltimer` 会调用 `osyield` 使线程让出这个处理器，等待这个 t 的状态被其他调用转换完，当再被调度执行时，函数通过外部 `for` 循环再回到 `switch` 开头，基于当前 t 的状态进行状态转换。

类似地，如果碰到 `t.status` 状态为 `timerRunning` 与 `timerMoving` ， 说明目前这个 timer 正在运行或者正在被移动，因此也会调用 `osyield` 并在下个循环检查其状态是否已变化。

而如果 `deltimer` 碰到 `t.status` 状态为 `timerDeleted`、`timerRemoving`、`timerRemoved`、`timerNoStatus` ，说明这个 timer 已经被删除（`timerNoStatus`也可能是这个 timer 根本没加到堆里），因此不做任何处理返回 false，也代表这次调用并没有成功删除 t。

## 2.3 cleantimers 清除定时器

`cleantimers` 是 `addtimer` 也会调用的函数，作用是清除对应 P 的 `timers` 堆头部的定时器。

书中给出了源码以及对于状态处理的解释，也就是对于头部的 `timer` 即 `pp.timers[0]`，根据其 `status` 进行处理：

+ 若为 `timersDeleted` ，设置状态为 `timerRemoving`，调用 `dodeltimer0` 删除，然后再设置状态为 `timerRemoved`
+ 若为 `timerModifiedEarlier` 、 `timerModifiedLater`，设置状态为 `timerMoving` ，将 `t.when` 设置为 `t.nextwhen`，然后调用 `dodeltimer0` 从堆里删除，再通过 `doaddtimer` 将其加进去，触发排序维护整个堆；最后再把状态设为 `timerWaiting`。

在 2.4 modtimer 中，我们会看到更新 `timer` 的 `when` 的时候会设置 `nextwhen` 而不是直接更新 `when` 的，具体这么做的原因在后文会写，这边可以看到这里 `t.when = t.nextwhen` 这部分逻辑是和 `modtimer` 设置 `nextwhen` 的逻辑是对应的。

下面我们对于书中没展开的 `dodeltimer0` 的源码进行介绍

### 2.3.1 dodeltimer0

```golang
// dodeltimer0 removes timer 0 from the current P's heap.
// We are locked on the P when this is called.
// It reports whether it saw no problems due to races.
// The caller must have locked the timers for pp.
func dodeltimer0(pp *p) {
	if t := pp.timers[0]; t.pp.ptr() != pp {
		throw("dodeltimer0: wrong P")
	} else {
		t.pp = 0
	}
	last := len(pp.timers) - 1
	if last > 0 {
		pp.timers[0] = pp.timers[last]
	}
	pp.timers[last] = nil
	pp.timers = pp.timers[:last]
	if last > 0 {
		siftdownTimer(pp.timers, 0)
	}
	updateTimer0When(pp)
	atomic.Xadd(&pp.numTimers, -1)
}
```
 `dodeltimer0` 就是把 `timers[0]` 从整个堆中删除，这里可以看到整个函数本质上就是一个维护四叉堆的删除操作：将堆中末尾的元素放到队首（覆盖），同时然后从队首开始进行堆的下推操作（sift-down），以完成整个堆的排序。

完成堆的删除元素操作后，`dodeltimer0` 会更新 P 对应的需要更新的 `timers` 字段： `timer0When`、`numTimers`



## 2.4 modtimer 修改定时器

`modtimer` 函数可以修改 `timer` 的 `when`、`period`、`f`、`arg`、`seq` 字段，也是几个`timer`状态变更的函数里最复杂的，代码也比较长：

```golang
// modtimer modifies an existing timer.
// This is called by the netpoll code or time.Ticker.Reset or time.Timer.Reset.
// Reports whether the timer was modified before it was run.
func modtimer(t *timer, when, period int64, f func(interface{}, uintptr), arg interface{}, seq uintptr) bool {
	if when < 0 {
		when = maxWhen
	}

	status := uint32(timerNoStatus)
	wasRemoved := false
	var pending bool
	var mp *m
// 第一部分： t.status 状态更新为 timerModifying
loop:
	for {
		switch status = atomic.Load(&t.status); status {
		case timerWaiting, timerModifiedEarlier, timerModifiedLater:
			// Prevent preemption while the timer is in timerModifying.
			// This could lead to a self-deadlock. See #38070.
			mp = acquirem()
			if atomic.Cas(&t.status, status, timerModifying) {
				pending = true // timer not yet run
				break loop
			}
			releasem(mp)
		case timerNoStatus, timerRemoved:
			// Prevent preemption while the timer is in timerModifying.
			// This could lead to a self-deadlock. See #38070.
			mp = acquirem()

			// Timer was already run and t is no longer in a heap.
			// Act like addtimer.
			if atomic.Cas(&t.status, status, timerModifying) {
				wasRemoved = true
				pending = false // timer already run or stopped
				break loop
			}
			releasem(mp)
		case timerDeleted:
			// Prevent preemption while the timer is in timerModifying.
			// This could lead to a self-deadlock. See #38070.
			mp = acquirem()
			if atomic.Cas(&t.status, status, timerModifying) {
				atomic.Xadd(&t.pp.ptr().deletedTimers, -1)
				pending = false // timer already stopped
				break loop
			}
			releasem(mp)
		case timerRunning, timerRemoving, timerMoving:
			// The timer is being run or moved, by a different P.
			// Wait for it to complete.
			osyield()
		case timerModifying:
			// Multiple simultaneous calls to modtimer.
			// Wait for the other call to complete.
			osyield()
		default:
			badTimer()
		}
	}

	t.period = period
	t.f = f
	t.arg = arg
	t.seq = seq
// 第二部分：当前 t 已经被删除（removed）的情况
	if wasRemoved {
		t.when = when
		pp := getg().m.p.ptr()
		lock(&pp.timersLock)
		doaddtimer(pp, t)
		unlock(&pp.timersLock)
		if !atomic.Cas(&t.status, timerModifying, timerWaiting) {
			badTimer()
		}
		releasem(mp)
		wakeNetPoller(when)
// 第三部分：没有被删除的情况
	} else {
		// The timer is in some other P's heap, so we can't change
		// the when field. If we did, the other P's heap would
		// be out of order. So we put the new when value in the
		// nextwhen field, and let the other P set the when field
		// when it is prepared to resort the heap.
		t.nextwhen = when

		newStatus := uint32(timerModifiedLater)
		if when < t.when {
			newStatus = timerModifiedEarlier
		}

		tpp := t.pp.ptr()

		// Update the adjustTimers field.  Subtract one if we
		// are removing a timerModifiedEarlier, add one if we
		// are adding a timerModifiedEarlier.
		adjust := int32(0)
		if status == timerModifiedEarlier {
			adjust--
		}
		if newStatus == timerModifiedEarlier {
			adjust++
			updateTimerModifiedEarliest(tpp, when)
		}
		if adjust != 0 {
			atomic.Xadd(&tpp.adjustTimers, adjust)
		}

		// Set the new status of the timer.
		if !atomic.Cas(&t.status, timerModifying, newStatus) {
			badTimer()
		}
		releasem(mp)

		// If the new status is earlier, wake up the poller.
		if newStatus == timerModifiedEarlier {
			wakeNetPoller(when)
		}
	}

	return pending
}

```

注释里提到了 `modtimer` 的使用场景，要么由 netpoll 代码调用，或者由 `time.Ticker.Reset` 、 `time.Timer.Reset` 调用

这里我们分成三部分来分析整个 `modtimer`，如代码中我的注释所划分的。

### 2.4.1 t.status 状态更新为 timerModifying

这一部分的代码和 `deltimer` 的代码很像：一个无限 for 循环内一个判断 `t.status` 的 switch 代码段。但值得注意的是它没有通过 return 直接退出函数（因为循环体后面还有代码要执行），而是通过 `break loop` 来退出整个循环体。

对于`t.status`更新前的不同状态，这段代码是这么处理的：

+ `timerWaiting` / `timerModifiedEarlier` / `timerModifiedLater` - 将 `t.status` 设置为 `timerModifying` ； 同时设置 `pending` 为 `true`，`pending` 作为函数的返回值代表了该 `timer` **是否是在运行前被修改（whether the timer was modified before it was run）**。
+ `timerNoStatus` / `timerRemoved` - 将 `t.status` 设置为 `timerModifying`； 同时设置 `pending` 为 `false`，因为 `t.status` 这个状态代表了 t 已经执行完或者已经被删除了； 同时设置 `wasRemoved` 为 `true` 代表这个 t 是被删除的
+ `timerDeleted` -  将 `t.status` 设置为 `timerModifying`；同时通过 `t.pp` 找到所属 P，再 `p.deletedTimers += 1`，相当于还原删除前 P 里的相关字段 ；同时设置 `pending` 为 `false` ，因为这个 t 处于 `timerDeleted` 状态，已经停止，等待被删除了。
+ `timerRunning` / `timerRemoving` / `timerMoving` / `timerModifying` - 和 `deltimer` 这部分代码类似，通过调用 `osyield` 函数交出处理器，等待其他调用完成这个 `timer` 状态的变更。

完成 `t.status` 状态的处理后，代码会将入参的 `period`、`f`、`arg`、`seq` 更新到 t 的对应字段，至于 `when` 与 `status` 会在后面的代码更新。

### 2.4.2 当前 t 已经被删除（removed）的情况

这部分书里也有提到，当 `wasRemoved` 为 `true` 时，代表这个定时器已经从堆中移除，因此重新设置了 when 等字段后，需要重新加入到堆里。

这段代码做了这些事：

1. 设置 `t.when` 为当前入参 `when`
2. 获取到当前 goroutine 对应的 P （`pp`）， 通过 `doaddtimer` 将 t 加入到 `pp.timers` 四叉堆中，并调用 `wakeNetPoller`
3. 将 `t.status` 设置为 `timerWaiting`
4. 调用 `wakeNetPoller` 唤醒网络轮询器中的线程

### 2.4.3 没有被删除的情况

这段代码体现了 `nextwhen` 字段的作用。代码注释也说明了，当我们修改 `when` 时可能修改的是其他 P 持有的 `t` ，贸然修改 `t.when` 会使对应 P 的`timers`四叉堆失序，因此，我们这边将更新的值放在 `nextwhen` 中，由持有这个 `t` 的对应的 P 选择合适的时机进行堆的调整排序。

然后就是根据入参 `when` 的值决定要更新的状态，这个书里也讲了：

+ 若入参 `when` 小于 `t.when` ，则更新状态为 `timerModifiedEarlier` ， 
+ 否则，若入参 `when` 大于等于 `t.when` ，则更新状态为 `timerModifiedLater`

之后，根据要更新的状态调整 `tpp.adjustTimers` 的值，`adjustTimers` 就是用来记录 `timers` 堆中 `timerModifiedEarlier` 状态的 `timer` 的数量

在确定更新状态为 `timerModifiedEarlier` 后， 函数会调用 `updateTimerModifiedEarliest` ，这个函数会比较 `when` 与 `tpp.timerModifiedEarliest` 的大小，若 `when` 更小则更新 `tpp.timerModifiedEarliest` 的值；同时，函数会调用 `wakeNetPoller` ，触发调度器去检查 `timers` 堆是否有需要处理的定时器。


## 2.5 adjusttimers 调整定时器

如书中所说，`adjusttimers` 的逻辑和 `cleantimers` 是很类似的，`cleantimers` 主要处理 `timers` 堆头部 `timer` ，而 `adjusttimers` 则是循环遍历 `timers` 堆中所有元素，对于状态进行处理。

`adjusttimers` 会在 `checktimers` 中调用，以集中处理堆中 deleted、earlier、later 等状态，在后面【触发定时器】章节会提到。

下面是 `adujsttimers` 整体代码：

```golang
// adjusttimers looks through the timers in the current P's heap for
// any timers that have been modified to run earlier, and puts them in
// the correct place in the heap. While looking for those timers,
// it also moves timers that have been modified to run later,
// and removes deleted timers. The caller must have locked the timers for pp.
func adjusttimers(pp *p, now int64) {
	if atomic.Load(&pp.adjustTimers) == 0 {
		if verifyTimers {
			verifyTimerHeap(pp)
		}
		// There are no timers to adjust, so it is safe to clear
		// timerModifiedEarliest. Do so in case it is stale.
		// Everything will work if we don't do this,
		// but clearing here may save future calls to adjusttimers.
		atomic.Store64(&pp.timerModifiedEarliest, 0)
		return
	}

	// If we haven't yet reached the time of the first timerModifiedEarlier
	// timer, don't do anything. This speeds up programs that adjust
	// a lot of timers back and forth if the timers rarely expire.
	// We'll postpone looking through all the adjusted timers until
	// one would actually expire.
	if first := atomic.Load64(&pp.timerModifiedEarliest); first != 0 {
		if int64(first) > now {
			if verifyTimers {
				verifyTimerHeap(pp)
			}
			return
		}

		// We are going to clear all timerModifiedEarlier timers.
		atomic.Store64(&pp.timerModifiedEarliest, 0)
	}

	var moved []*timer
loop:
	for i := 0; i < len(pp.timers); i++ {
		t := pp.timers[i]
		if t.pp.ptr() != pp {
			throw("adjusttimers: bad p")
		}
		switch s := atomic.Load(&t.status); s {
		case timerDeleted:
			if atomic.Cas(&t.status, s, timerRemoving) {
				dodeltimer(pp, i)
				if !atomic.Cas(&t.status, timerRemoving, timerRemoved) {
					badTimer()
				}
				atomic.Xadd(&pp.deletedTimers, -1)
				// Look at this heap position again.
				i--
			}
		case timerModifiedEarlier, timerModifiedLater:
			if atomic.Cas(&t.status, s, timerMoving) {
				// Now we can change the when field.
				t.when = t.nextwhen
				// Take t off the heap, and hold onto it.
				// We don't add it back yet because the
				// heap manipulation could cause our
				// loop to skip some other timer.
				dodeltimer(pp, i)
				moved = append(moved, t)
				if s == timerModifiedEarlier {
					if n := atomic.Xadd(&pp.adjustTimers, -1); int32(n) <= 0 {
						break loop
					}
				}
				// Look at this heap position again.
				i--
			}
		case timerNoStatus, timerRunning, timerRemoving, timerRemoved, timerMoving:
			badTimer()
		case timerWaiting:
			// OK, nothing to do.
		case timerModifying:
			// Check again after modification is complete.
			osyield()
			i--
		default:
			badTimer()
		}
	}

	if len(moved) > 0 {
		addAdjustedTimers(pp, moved)
	}

	if verifyTimers {
		verifyTimerHeap(pp)
	}
}
```
在函数开始，`adjusttimers` 会做两件事：

第一件事情是检查 `pp.adjustTimers` 的值，若值为 0 ，则说明 `timers` 堆里应该没有要调整的 `timer` ，这时候如果开启了 `verifyTimers` (调试用)，则会调用 `verifyTimerHeap` 检查四叉堆的状态是否还正常。之后再将 `pp.timerModifiedEarliest` 赋 0 。最后直接退出这个函数。

第二件事是检查 `pp.timerModifiedEarliest` 的值，若值晚于现在的时间，则调用 `verifyTimerHeap`  检查四叉堆（开启`verifyTimers`时）并退出函数。这优化了那些来回调整大量定时器但定时器很少过期的程序的性能，也就是说函数会推迟检查所有调整过的定时器，直到其中一个实际过期。

做完这两件事就是开始循环遍历 `timers` 里的所有 `timer` 了，循环将根据 `timer` 的状态进行分条件处理：

1. `timerDeleted` - 被标记为删除的定时器，函数会将这个定时器状态设为 `timerRemoving` ，调用 `dodeltimer` 后再将这个定时器状态设为 `timerRemoved`（`dodeltimer`函数后文会分析），由于调用了 `dodeltimer` 将这个定时器从 `timers` 移除了，当前位置 i 是其他上移了的定时器，因此这里会将 `i--`，重新循环看这个位置的定时器的状态。

2. `timerModifiedEarlier`, `timerModifiedLater` - 将这个定时器从 `timers` 移除，同时将这个定时器加入到 `moved` 这个 slice 中，之所以不直接把这个定时器加到堆里，是因为在循环内直接添加这个定时器到堆里并触发维护会导致堆里的元素位置都出现不可控的变化，从而导致循环元素的错乱，所以先把这个定时器放到 `moved`里，在循环结束后统一添加到堆里。然后这部分代码会检查 `pp.adjustTimers` ，如果为 0 代表已经没有要调整的定时器就退出循环，最后和前面情况一样，`i--`，重新循环看这个位置的定时器。

循环完之后，如果 `moved > 0` 则会调用 `addAdjustedTimers` 函数将 `moved` 中的定时器加入到堆中。

下面我们看下 `dodeltimer` 和 `addAdjustedTimers` 代码，这两部分代码比较简单也比较好理解。

### 2.5.1 dodeltimer

```golang
// dodeltimer removes timer i from the current P's heap.
// We are locked on the P when this is called.
// It reports whether it saw no problems due to races.
// The caller must have locked the timers for pp.
func dodeltimer(pp *p, i int) {
	if t := pp.timers[i]; t.pp.ptr() != pp {
		throw("dodeltimer: wrong P")
	} else {
		t.pp = 0
	}
	last := len(pp.timers) - 1
	if i != last {
		pp.timers[i] = pp.timers[last]
	}
	pp.timers[last] = nil
	pp.timers = pp.timers[:last]
	if i != last {
		// Moving to i may have moved the last timer to a new parent,
		// so sift up to preserve the heap guarantee.
		siftupTimer(pp.timers, i)
		siftdownTimer(pp.timers, i)
	}
	if i == 0 {
		updateTimer0When(pp)
	}
	atomic.Xadd(&pp.numTimers, -1)
}
```
与 `dodeltimer0` 类似，只不过 `dodeltimer0` 移除头部的元素，而 `dodeltimer` 可以指定要删除元素下标 `i` 。

这里 `dodeltimer` 会将堆末尾元素 `last` 覆盖 `pp.timer[i]`，并去掉末尾元素 last `pp.timers[last] = nil`、`pp.timers = pp.timers[:last]` 

当 `i != last` 时，函数会先调用 `siftUpTimer` 将元素 i 尝试上移（因为这次覆盖可能到其他子树上了），再调用 `siftDownTimer` 下移，以维护整个堆的有序。

最后更新 `pp.timer0when` 和 `pp.numTimers`


### 2.5.2 addAdjustedTimers

```golang
// addAdjustedTimers adds any timers we adjusted in adjusttimers
// back to the timer heap.
func addAdjustedTimers(pp *p, moved []*timer) {
	for _, t := range moved {
		doaddtimer(pp, t)
		if !atomic.Cas(&t.status, timerMoving, timerWaiting) {
			badTimer()
		}
	}
}
```

批量添加定时器到 `pp.timers` ， 逻辑很简单，循环调用 `doaddtimer` ， 然后将定时器状态从 `timerMoving` 改为 `timerWaiting`。

## 2.6 runtimer 运行定时器

这部分代码书中解释比较清楚，对于 `timerDeleted`、`timerModifiedEarlier`、`timerModifiedLater` 状态的处理和 `cleantimer` 基本一致。重点是对于 `timerWaiting` 的处理，它在堆顶定时器如果没有到期时会让函数直接退出，否则会调用 `runOneTimer` 来执行这个定时器。

这部分代码涉及到的字段、函数等大部分在前文已经介绍过了，故省略。

不过要补充的一点是，这个函数的注释里介绍了 `runtimer` 函数的返回值：

```golang
// runtimer examines the first timer in timers. If it is ready based on now,
// it runs the timer and removes or updates it.
// Returns 0 if it ran a timer, -1 if there are no more timers, or the time
// when the first timer should run.
// The caller must have locked the timers for pp.
// If a timer is run, this will temporarily unlock the timers.
//
```
当函数运行了一个定时器，返回 0 ；如果已经没有 `timer` 了，返回 -1； 如果第一个定时器没执行（时间没到），则返回应该执行的时间。




# 3. 触发定时器 （checkTimers）


这部分内容事实上牵扯到 netpoll 网络轮询器以及调度器整体的机制，书中也没有详细展开这部分 netpoll 与调度器触发定时器的机制，我还没整理完。后面整理完之后这一章应该会更新。

而书中提到的 `checkTimers` 可以展开再详细讲讲，我们分为两个部分：

```golang
// checkTimers runs any timers for the P that are ready.
// If now is not 0 it is the current time.
// It returns the current time or 0 if it is not known,
// and the time when the next timer should run or 0 if there is no next timer,
// and reports whether it ran any timers.
// If the time when the next timer should run is not 0,
// it is always larger than the returned time.
// We pass now in and out to avoid extra calls of nanotime.
//go:yeswritebarrierrec
func checkTimers(pp *p, now int64) (rnow, pollUntil int64, ran bool) {
	// 第一部分，判断是否运行 timer
	// If it's not yet time for the first timer, or the first adjusted
	// timer, then there is nothing to do.
	next := int64(atomic.Load64(&pp.timer0When))
	nextAdj := int64(atomic.Load64(&pp.timerModifiedEarliest))
	if next == 0 || (nextAdj != 0 && nextAdj < next) {
		next = nextAdj
	}

	if next == 0 {
		// No timers to run or adjust.
		return now, 0, false
	}

	if now == 0 {
		now = nanotime()
	}
	if now < next {
		// Next timer is not ready to run, but keep going
		// if we would clear deleted timers.
		// This corresponds to the condition below where
		// we decide whether to call clearDeletedTimers.
		if pp != getg().m.p.ptr() || int(atomic.Load(&pp.deletedTimers)) <= int(atomic.Load(&pp.numTimers)/4) {
			return now, next, false
		}
	}

	// 第二部分，循环执行 timer
	lock(&pp.timersLock)

	if len(pp.timers) > 0 {
		adjusttimers(pp, now)
		for len(pp.timers) > 0 {
			// Note that runtimer may temporarily unlock
			// pp.timersLock.
			if tw := runtimer(pp, now); tw != 0 {
				if tw > 0 {
					pollUntil = tw
				}
				break
			}
			ran = true
		}
	}

	// If this is the local P, and there are a lot of deleted timers,
	// clear them out. We only do this for the local P to reduce
	// lock contention on timersLock.
	if pp == getg().m.p.ptr() && int(atomic.Load(&pp.deletedTimers)) > len(pp.timers)/4 {
		clearDeletedTimers(pp)
	}

	unlock(&pp.timersLock)

	return now, pollUntil, ran
}

```

第一部分是函数通过 `timer0When`、`timerModifiedEarliest` 与 `now` 判断 `timers` 里需要执行的第一个定时器是否到时间执行了。如果还没到，则直接退出函数。

第二部分则是开始循环执行定时器：

+ 加锁 `timersLock`
+ 如果四叉堆里有定时器，则继续执行
  + `adjusttimers` 调整堆内  `timerDeleted`、`timerModifiedEarlier`、`timerModifiedLater` 状态的定时器
  + 开启循环，执行 `runtimer`
    + 如果 `runtimer` 返回了 0 ，代表 `runtimer` 执行了一个定时器，则循环继续看下一个定时器是否要执行
	+ 如果 `runtimer` 没返回 0，代表 `runtimer` 没执行（时间没到），`pollUntil = tw` 确定下一个定时器执行的时间，退出循环
+ 如果已经需要删除的定时器大于 `timers` 总数的四分之一，则调用 `clearDeletedTimers` 清理这些要删除的定时器
+ 解锁

# 4. 总结

文章末尾提到了定时器执行的时间误差在 10ms 左右，个人觉得这应该是因为调度器调度 goroutine 执行时间片最大在 10ms，因此最多在 10ms 之后 P 中所在的定时器就会被检查是否被执行。

我发现通过写博客来梳理源码等内容还是能够增强记忆的，这样梳理下来对于定时器部分的数据结构，以及状态变换的逻辑理解得更清晰了。

但目前欠缺理清楚的还是定时器触发机制与调度器、网络轮询器之间的关系。后面理清楚了会在本文进行更新。

这篇文章更多是自己对于 Golang 定时器源码的学习与梳理，可读性比较差，里面可能存在较多错误，如果有老板觉得有问题请及时反馈，多谢！




