## 8.7.Gosched

用户可以通过调用的`runtime.Gosched`将当前G任务暂停，重新放回全局队列，让出当前M去执行其他任务。我们无须对G做唤醒操作，因为它总归会被某个M重新拿到，并从”断点“恢复。

```go
// Gosched yields the processor, allowing other goroutines to run. It does not
// suspend the current goroutine, so execution resumes automatically.
func Gosched() {
	checkTimeouts()
	mcall(gosched_m)
}

// Gosched continuation on g0.
func gosched_m(gp *g) {
	if trace.enabled {
		traceGoSched()
	}
	goschedImpl(gp)
}

func goschedImpl(gp *g) {
	status := readgstatus(gp)
	if status&^_Gscan != _Grunning {
		dumpgstatus(gp)
		throw("bad g status")
	}
  // 重置属性
	casgstatus(gp, _Grunning, _Grunnable)
	dropg()
	lock(&sched.lock)
  
  // 将当前G放回全局队列
	globrunqput(gp)
	unlock(&sched.lock)

  // 重新调度
	schedule()
}

func dropg() {
	_g_ := getg()

	setMNoWB(&_g_.m.curg.m, nil)
	setGNoWB(&_g_.m.curg, nil)
}
```

实现”断点恢复“的关键由`mcall`实现，它将当前执行状态，包括SP，PC寄存器等值保存到G.sched区域。

看看mcall的实现：

```bash
// func mcall(fn func(*g))
// Switch to m->g0's stack, call fn(g).
// Fn must never return. It should gogo(&g->sched)
// to keep running g.
TEXT runtime·mcall(SB), NOSPLIT, $0-8
	MOVQ	fn+0(FP), DI

	get_tls(CX)
	MOVQ	g(CX), AX	// save state in g->sched
	MOVQ	0(SP), BX	// caller's PC
	MOVQ	BX, (g_sched+gobuf_pc)(AX)
	LEAQ	fn+0(FP), BX	// caller's SP
	MOVQ	BX, (g_sched+gobuf_sp)(AX)
	MOVQ	AX, (g_sched+gobuf_g)(AX)
	MOVQ	BP, (g_sched+gobuf_bp)(AX)

	// switch to m->g0 & its stack, call fn
	MOVQ	g(CX), BX
	MOVQ	g_m(BX), BX
	MOVQ	m_g0(BX), SI
	CMPQ	SI, AX	// if g == m->g0 call badmcall
	JNE	3(PC)
	MOVQ	$runtime·badmcall(SB), AX
	JMP	AX
	MOVQ	SI, g(CX)	// g = m->g0
	MOVQ	(g_sched+gobuf_sp)(SI), SP	// sp = m->g0->sched.sp
	PUSHQ	AX
	MOVQ	DI, DX
	MOVQ	0(DI), DI
	CALL	DI
	POPQ	AX
	MOVQ	$runtime·badmcall2(SB), AX
	JMP	AX
	RET
```

当`execute/gogo`再次执行该任务时，自然可从中恢复状态。反正执行栈是G自带的，不用担心执行数据丢失。



## 8.8.gopark

与`Gosched`最大的区别在于，`gopark`并没将G放回待运行队列。也就是说，必须主动恢复，否则该任务会遗失。

```go
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

可以看到`gopark`同样是由`mcall`保存执行状态，还有个`unlockf`作为暂停判断条件。

```go
// park continuation on g0.
func park_m(gp *g) {
	_g_ := getg()

	if trace.enabled {
		traceGoPark(_g_.m.waittraceev, _g_.m.waittraceskip)
	}

  // 重置属性
	casgstatus(gp, _Grunning, _Gwaiting)
	dropg()

  // 执行解锁函数。如果返回false，则恢复执行
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
  // 调度执行其他任务
	schedule()
}
```

与之配套，`goready`用于恢复执行，G被放回优先级最高的`P.runnext`。

```go
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
  // 修正状态，重新放回本地 runnext
	casgstatus(gp, _Gwaiting, _Grunnable)
	runqput(_g_.m.p.ptr(), gp, next)
	if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 {
		wakep()
	}
	releasem(mp)
}
```



## 8.9.notesleep

相比`gosched`，`gopark`，反应更敏捷的`notesleep`既不让出M，也就不会让G重回任务队列。它直接让线程休眠直到被唤醒，更适合`stopm`，`gcMark`这类近似自旋的场景。

在Linux，DragonFly，FreeBSD平台，`notesleep`是基于`Futex`的高性能实现。

**Futex**：通常称作”快速用户区互斥“，是一种在用户空间实现的锁（互斥）机制。多执行单位（进程或线程）通过共享同一块内存（整数）来实现等待和唤醒操作。因为Futex只在操作结果不一致时才进入内核仲裁，所以有非常高的执行效率。



**关于Futex，这个讲得比较清楚：**Futex按英文翻译过来就是快速用户空间互斥体。其设计思想其实 不难理解，在传统的Unix系统中，System V IPC(inter process communication)，如 semaphores, msgqueues, sockets还有文件锁机制(flock())等进程间同步机制都是对一个内核对象操作来完成的，这个内核对象对要同步的进程都是可见的，其提供了共享 的状态信息和原子操作。当进程间要同步的时候必须要通过系统调用(如semop())在内核中完成。可是经研究发现，很多同步是无竞争的，即某个进程进入 互斥区，到再从某个互斥区出来这段时间，常常是没有进程也要进这个互斥区或者请求同一同步变量的。但是在这种情况下，这个进程也要陷入内核去看看有没有人 和它竞争，退出的时侯还要陷入内核去看看有没有进程等待在同一同步变量上。这些不必要的系统调用(或者说内核陷入)造成了大量的性能开销。为了解决这个问 题，Futex就应运而生，Futex是一种用户态和内核态混合的同步机制。首先，同步的进程间通过mmap共享一段内存，futex变量就位于这段共享 的内存中且操作是原子的，当进程尝试进入互斥区或者退出互斥区的时候，先去查看共享内存中的futex变量，如果没有竞争发生，则只修改futex,而不 用再执行系统调用了。当通过访问futex变量告诉进程有竞争发生，则还是得执行系统调用去完成相应的处理(wait 或者 wake up)。简单的说，futex就是通过在用户态的检查，（motivation）如果了解到没有竞争就不用陷入内核了，大大提高了low-contention时候的效率。 Linux从2.5.7开始支持Futex。



runtime.go

```go
type note struct {
	// Futex-based impl treats it as uint32 key,
	// while sema-based impl as M* waitm.
	// Used to be a union, but unions break precise GC.
	key uintptr
}
```

围绕`note.key`值来处理休眠和唤醒操作。

lock_futex.go

```go
func notesleep(n *note) {
	gp := getg()
	if gp != gp.m.g0 {
		throw("notesleep not on g0")
	}
	ns := int64(-1)
	if *cgo_yield != nil {
		// Sleep for an arbitrary-but-moderate interval to poll libc interceptors.
		ns = 10e6
	}
	for atomic.Load(key32(&n.key)) == 0 {
		gp.m.blocked = true
		futexsleep(key32(&n.key), 0, ns)  // 检查n.key == 0，休眠
		if *cgo_yield != nil {  
			asmcgocall(*cgo_yield, nil)
		}
		gp.m.blocked = false  // 唤醒后 n.key == 1
	}
}

func notewakeup(n *note) {
  // 如果old != 0，表示已经执行过唤醒操作
	old := atomic.Xchg(key32(&n.key), 1)
	if old != 0 {
		print("notewakeup - double wakeup (", old, ")\n")
		throw("notewakeup - double wakeup")
	}
  // 唤醒后 n.key = 1
	futexwakeup(key32(&n.key), 1)
}

// One-time notifications.
// 重置休眠条件
func noteclear(n *note) {
	n.key = 0
}
```

os_linux.go

```go
// Atomically,
//	if(*addr == val) sleep
// Might be woken up spuriously; that's allowed.
// Don't sleep longer than ns; ns < 0 means forever.
//go:nosplit
func futexsleep(addr *uint32, val uint32, ns int64) {
	// Some Linux kernels have a bug where futex of
	// FUTEX_WAIT returns an internal error code
	// as an errno. Libpthread ignores the return value
	// here, and so can we: as it says a few lines up,
	// spurious wakeups are allowed.
  // 不超时
	if ns < 0 {
		futex(unsafe.Pointer(addr), _FUTEX_WAIT_PRIVATE, val, nil, nil, 0)
		return
	}

	var ts timespec
	ts.setNsec(ns)
  // 如果futex_value == val，则进入休眠等待状态，直到FUTEX_WAKE或超时。
	futex(unsafe.Pointer(addr), _FUTEX_WAIT_PRIVATE, val, unsafe.Pointer(&ts), nil, 0)
}

// If any procs are sleeping on addr, wake up at most cnt.
//go:nosplit
func futexwakeup(addr *uint32, cnt uint32) {
  // 唤醒cnt个等待单位，这会设置futex_value = 1
	ret := futex(unsafe.Pointer(addr), _FUTEX_WAKE_PRIVATE, cnt, nil, nil, 0)
	if ret >= 0 {
		return
	}

	// I don't know that futex wakeup can return
	// EAGAIN or EINTR, but if it does, it would be
	// safe to loop and call futex again.
	systemstack(func() {
		print("futexwakeup addr=", addr, " returned ", ret, "\n")
	})

	*(*int32)(unsafe.Pointer(uintptr(0x1006))) = 0x1006
}
```



## 8.10.Goexit

用户可以调用`runtime.Goexit`立即终止G任务，不管当前处于调用堆栈的哪个层次。在终止前，它确保所有`G.defer`被执行。

panic.go

```go
func Goexit() {
	// Run all deferred functions for the current goroutine.
	// This code is similar to gopanic, see that implementation
	// for detailed comments.
	gp := getg()
	for {
		d := gp._defer
		if d == nil {
			break
		}
		if d.started {
			if d._panic != nil {
				d._panic.aborted = true
				d._panic = nil
			}
			d.fn = nil
			gp._defer = d.link
			freedefer(d)
			continue
		}
		d.started = true
		reflectcall(nil, unsafe.Pointer(d.fn), deferArgs(d), uint32(d.siz), uint32(d.siz))
		if gp._defer != d {
			throw("bad defer entry in Goexit")
		}
		d._panic = nil
		d.fn = nil
		gp._defer = d.link
		freedefer(d)
		// Note: we ignore recovers here because Goexit isn't a panic
	}
	goexit1()
}
```

**比较有趣的是：**在`main goroutine`里执行`Goexit`，它会等待其他`goroutine`结束后才会崩溃。



## 8.11.stopTheWorld和startTheWorld

用户逻辑必须暂停在一个安全的点上，否则会引发很多意外问题。因此，`stopTheWorld`同样是通过”通知“机制，让G主动停止。比如，设置”gcwaiting = 1“让调度函数`schedule`主动休眠M；向所有正在运行的G任务发出抢占调度，使其暂停。

proc.go

```go
func stopTheWorld(reason string) {
	semacquire(&worldsema)
	getg().m.preemptoff = reason
	systemstack(stopTheWorldWithSema)
}

func stopTheWorldWithSema() {
	_g_ := getg()

	// If we hold a lock, then we won't be able to stop another M
	// that is blocked trying to acquire the lock.
	if _g_.m.locks > 0 {
		throw("stopTheWorld: holding locks")
	}

	lock(&sched.lock)
	sched.stopwait = gomaxprocs
  // 设置停止标志，让schedule之类的调用主动休眠M
	atomic.Store(&sched.gcwaiting, 1)
  
  // 向所有正在运行的G发出抢占调度
	preemptall()
	// stop current P
  // 
	_g_.m.p.ptr().status = _Pgcstop // Pgcstop is only diagnostic.
	sched.stopwait--
	// try to retake all P's in Psyscall status
  // 尝试暂停所有syscall
	for _, p := range allp {
		s := p.status
		if s == _Psyscall && atomic.Cas(&p.status, s, _Pgcstop) {
			if trace.enabled {
				traceGoSysBlock(p)
				traceProcStop(p)
			}
			p.syscalltick++
			sched.stopwait--
		}
	}
	// stop idle P's
  // 处理空闲P
	for {
		p := pidleget()
		if p == nil {
			break
		}
		p.status = _Pgcstop
		sched.stopwait--
	}
	wait := sched.stopwait > 0
	unlock(&sched.lock)

	// wait for remaining P's to stop voluntarily
  // 等待
	if wait {
		for {
			// wait for 100us, then try to re-preempt in case of any races
      // 暂停100us后，重新发出抢占调度
      // handoffp，gcstopm, entersyscall_gcwait 等操作都会 sched.stopwait--
      // 如果stopwait == 0,则尝试唤醒 stopnote
      // 若唤醒成功，跳出循环，失败，则重新发出抢占调度，再次等待。
			if notetsleep(&sched.stopnote, 100*1000) {
				noteclear(&sched.stopnote)
				break
			}
			preemptall()
		}
	}

	// sanity checks
	bad := ""
	if sched.stopwait != 0 {
		bad = "stopTheWorld: not stopped (stopwait != 0)"
	} else {
    // 检查所有P状态
		for _, p := range allp {
			if p.status != _Pgcstop {
				bad = "stopTheWorld: not stopped (status != _Pgcstop)"
			}
		}
	}
	if atomic.Load(&freezing) != 0 {
		// Some other thread is panicking. This can cause the
		// sanity checks above to fail if the panic happens in
		// the signal handler on a stopped thread. Either way,
		// we should halt this thread.
		lock(&deadlock)
		lock(&deadlock)
	}
	if bad != "" {
		throw(bad)
	}
}

// 向所有P发起抢占调度
func preemptall() bool {
	res := false
	for _, _p_ := range allp {
		if _p_.status != _Prunning {
			continue
		}
		if preemptone(_p_) {
			res = true
		}
	}
	return res
}
```

总体上看，`stopTheWorld`还是很平和的一种手段，会循环等待目标任务进入一个安全点后主动暂停。而`startTheWorld`就更简单，毕竟是从冻结状态开始，无非是唤醒相关P/M继续执行任务。

```go
// startTheWorld undoes the effects of stopTheWorld.
func startTheWorld() {
	systemstack(func() { startTheWorldWithSema(false) })
	// worldsema must be held over startTheWorldWithSema to ensure
	// gomaxprocs cannot change while worldsema is held.
	semrelease(&worldsema)
	getg().m.preemptoff = ""
}

func startTheWorldWithSema(emitTraceEvent bool) int64 {
	mp := acquirem() // disable preemption because it can be holding p in a local var
	if netpollinited() {
		list := netpoll(false) // non-blocking
		injectglist(&list)
	}
	lock(&sched.lock)

	procs := gomaxprocs
	if newprocs != 0 {
		procs = newprocs
		newprocs = 0
	}
  
  // 检查是否需要procresize
	p1 := procresize(procs)
  
  // 解除停止状态
	sched.gcwaiting = 0
  
  // 唤醒 sysmon
	if sched.sysmonwait != 0 {
		sched.sysmonwait = 0
		notewakeup(&sched.sysmonnote)
	}
	unlock(&sched.lock)

  // 循环有任务的P链表，让它们继续工作
	for p1 != nil {
		p := p1
		p1 = p1.link.ptr()
		if p.m != 0 {
			mp := p.m.ptr()
			p.m = 0
			if mp.nextp != 0 {
				throw("startTheWorld: inconsistent mp->nextp")
			}
			mp.nextp.set(p)
			notewakeup(&mp.park)
		} else {
			// Start M to run P.  Do not start another M below.
			newm(nil, p)
		}
	}

	// Capture start-the-world time before doing clean-up tasks.
	startTime := nanotime()
	if emitTraceEvent {
		traceGCSTWDone()
	}

	// Wakeup an additional proc in case we have excessive runnable goroutines
	// in local queues or in the global queue. If we don't, the proc will park itself.
	// If we have lots of excessive work, resetspinning will unpark additional procs as necessary.
  // 让闲置的家伙都起来工作
	if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 {
		wakep()
	}

  // 重置抢占标志
	releasem(mp)

	return startTime
}
```



## 8.12.总结

### 8.12.1.线程模型

线程模型一般分三种：

+ **N:1**：即全部的用户线程都映射到一个OS线程上面，**优点：**上下文切换成本最低，**缺点：**但无法利用多核资源；
+ **1:1**：一个用户线程对应到一个OS线程上，**优点：**能利用到多核资源，**缺点：**但是上下文切换成本较高，这也是Java Hotspot VM的默认实现；
+ **M:N**：权衡上面两者方案，既能利用多核资源也能尽可能减少上下文切换成本，但是调度算法的实现成本偏高。



### 8.12.2.Golang线程模型的选型

**为什么Go Scheduler需要实现M:N的方案？**

+ **线程创建开销大。**对于OS线程而言，其很多特性均是操作系统给予的，但对于Go程序而言，其中1很多特性可能是非必要的。这样一来，如果是1:1的方案，那么每次**go func(){...}**都需要创建一个OS线程，而在创建线程过程中，OS线程里某些Go用不上的特性会转化为不必要的性能开销，不经济；
+ **减少Go垃圾回收的复杂度。**依据1:1方案，Go产生所用用户级线程均交由OS直接调度。Go的辣鸡回收器要求在运行时需要停止所有线程，才能使得内存达到稳定一致的状态，而OS不可能清楚这些，垃圾回收器也不能控制OS去阻塞线程。



### 8.12.3.GMP模型

![img](https://static.studygolang.com/190608/a0d499878e72f8188b79895f51ddc8c9.jpg)



+ **M**(Machine)：就是OS线程本身，数量可配置；
+ **P**(Processor)：调度器的核心处理器，通常表示执行上下文，用于匹配M和G。P的数量不能超过**GOMAXPROCS**配置数量，这个参数的默认值为CPU核心数；通常一个P可以与多个M对应，但同一时刻，这个P只能和其中一个M发生绑定关系；M被创建之后需要自行在P的freeList中找到P进行绑定，没有绑定P的M，会进入阻塞态。（GOMAXPROCS既决定了P的最大数量，也决定了自旋M的最大数量）
+ **G**(Goroutine)：Go的用户级线程，常说的协程，真正携带代码执行逻辑的部分，由**go func(){...}**直接生成。
  + **G0**：其本身也是G，也需要跟具体的M结合才能被执行，只不过它比较特殊，其本身就是一个schedule函数。在进程执行过程中，有两类代码需要运行。一类自然是用户逻辑，直接使用G栈内存；另一类是运行时管理指令，它并不便于直接在用户栈上执行，因为这需要处理与用户逻辑现场有关的一大堆事务。举例来说，G任务可在中途暂停，放回队列后由其他M获取执行。如果不更改执行栈，可能会造成多个线程共享内存，从而引发混乱。另外，在执行垃圾回收操作时，如何收缩依旧被线程持有的G栈空间？因此，当需要执行管理指令时，会将线程栈临时切换到g0，与用户逻辑彻底隔离。



### 8.12.4.Work-Steal模型



![img](https://static.studygolang.com/190608/be523e473c1f9c656ba6c8c543f413f8.jpg)



明确一下的几个概念：

+ **本地队列**：本地是相对于P而言的本地，每个P都维护一个本地队列；与P绑定的M中如若生成新的G，一般情况下会放到P的本地队列；当本地队列满了的时候，才会截取本地队列中“一半”的G放入到全局队列中；
+ **全局队列：**承载本地队列“溢出“的G。为了保证调度公平性，schedule过程中有 1/61 的几率优先检查全局队列，否则本地队列一直满载的情况下，全局队列中的G将永远无法被调度到；
+ **窃取：**为了使得空闲（idle）的M有活干，不空等，提高计算资源的利用率。窃取也是有章法的，规则是随机从其他P的本地队列里窃取”一半“的G。



### 8.12.5.go scheduler调度总过程

整个调度过程如下：

+ 计数器每隔61，会有一次在全局队列中寻找G执行，其他情况下会在P的本地队列中寻找G；
+ 上面的过程，如果在全局队列中找不到G，会再返回P的本地队列寻找G；
+ 如果还是找不到，会从其他P的本地队列中“窃取“G（先尝试从其他P的本地队列中获取，如果获取不到，连P的runnext也拿过来，其中runnext是指P紧接下来要执行的G）；
+ 如果还是找不到G，则从全局队列中转移一部分的G到P的本地队列，这里”一部分“满足公式：`n = min(len(GQ)/GOMAXPROCS+1, len(GQ)/ 2)`，其中之所以加一是要把当前要执行的G也要算进去；
+ 如果还是找不到，陷入`netpoll`，尝试在网络中`poll G`。



只要找到了G，P就会把G丢给M进行处理。所以寻找G的前提是存在可以运行的M，没有可执行的M，G无法真正被执行，包括调度本身。



**调度的本质，就是P将G合理的分配给某个M的过程。**



### 8.12.6.线程自旋

**线程自旋是相对于线程阻塞而言的。**表象就是循环执行一个指定逻辑（就是上面提到的调度逻辑，目的是不停地寻找G）。这样做的问题显而易见，如果G迟迟不来，CPU会白白浪费在这无意义的计算上，但好处也很明显，降低了M的上下文切换，提高了性能。

具体来说，假设Scheduler中全局和本地队列均为空，M此时没有任何任务可以处理，那么你会选择让M进入阻塞状态还是选择让CPU空转等待G的驾临？

Go的设计者倾向于高性能的并发表现，选择了后者。当然前面也提到过，为了避免过多浪费CPU资源，自旋的线程数不会超过**GOMAXPROCS**，这是因为一个P在同一时刻只能绑定一个M，P的数量不会超过**GOMAXPROCS**，自然被绑定的M的数量也不会超过。对于未被绑定的”游离态“的M，会进入休眠阻塞态。



**那么，如果M因为G发起了系统调用进入了阻塞态会怎么样？**

![img](https://static.studygolang.com/190608/77826ccf4195fb1216becf383302af10.png)

如图，如果G8发起了阻塞系统调用（例如阻塞IO操作），使得对应的M2进入了阻塞态。此时如果没有任何的处理，Go Scheduler就会在这段阻塞的时间内，白白缺失了一个OS线程单元。



Go设计者的解决方案是：一旦G8发起了`Syscall`使得M2进入阻塞态，此时的P2会立即与M2解绑，保留M2与G8的关系，继而与新的OS线程M5绑定，继续下一轮的调度（上面的schedule过程）。那么虽然M2进入了阻塞态，但宏观来看，并没有缺失任何处理单元，P2依然正常工作。



**那么，G8的阻塞操作返回后怎么办？**

G8 失去了 P2，意味着失去了执行机会，M2 被唤醒以后第一件事就是要窃取一个上下文（Processor），还给 G8 执行机会。然而现实是 M2 不一定能够找到 P 绑定，不过找不到也没关系，M2 会将 G8 丢到全局队列中，等待调度。

这样一来G8会被其他的M调度到，重新获得执行机会，继续执行阻塞返回之后的逻辑。



### 8.12.7.调度过程

![image-20191114231406863](/Users/dengzhuowen/Library/Application Support/typora-user-images/image-20191114231406863.png)



### 8.12.8.杂项

+ **lockedm处理**：位于`schedule`的处理逻辑，即`lockedm`会休眠，直到某人把`lockedg`交给他，而不幸拿到`lockedg`的M，则要将`lockedg`连同P一起传递给`lockedm`，还负责将其唤醒。至于它自己，则因失去P而被迫休眠，直到`wakep`带着新的P唤醒它。
+ **连续栈：**每次进行函数调用的时候，通过比较SP寄存器和`stackguard0`来决定是否需要发起系统调用`runtime.morestack`来申请更大的空间来防止栈溢出。
+ 函数栈的管理跟对象内存空间的管理类似（tcmalloc）；
+ **sysmon细节：**详见8.7；
+ 几个调度中断的区别：
  + **Gosched**：将当前G任务暂停，重新放回全局队列，让出当前的M去执行其他任务。我们无须对G做唤醒操作，因为它总归会被某个M重新拿到，并从”断点“恢复（由`mcall`实现）；
  + **gopark**：暂停当前G，但是并没有放回全局队列，也就是说，必须主动恢复（`goready`），否则该任务会遗失；
  + **notesleep**：既不让出M，也不让G放回全局队列，它直接让线程休眠直到被唤醒，更适合`stopm`，`gcMark`这类近似自旋的场景，基于`futex`的高速实现；
+ **Goexit**：用户可以调用`runtime.Goexit`立即终止G任务，不管当前处于调用堆栈的哪个层次。在终止前，它确保所有`G.defer`被执行。在`main goroutine`里执行`Goexit`，它会等待其他`goroutine`结束后才会崩溃。
+ **stopTheWorld**：采用通知机制，主要做这三方面的工作：
  + 设置”gcwaiting = 1“让调度函数`schedule`主动休眠M；
  + 向所有正在运行的G任务发出抢占调度（**通过在目标G上面设置一个抢占标志`stackguard0`，当该任务调用某个函数时，被编译器安插的指令就会检查这个标志，从而决定是否暂停当前任务**），使其暂停；
  + 调用`notesleep`陷入自旋，检查正在运行的P是否已经暂停了。
+ **startTheWorld**：比较简单，从冻结状态开始，遍历循环，唤醒相关P/M继续执行任务。

