# 8.并发调度

golang的并发调度基本示意图如下：（GMP模型）

![image-20191105120158914](https://github.com/sysublackbear/golang_runtime_analysis/blob/master/img/image-20191105120158914.png)

其中：

+ P（Processor）：其作用类似CPU核，用来控制可同时并发执行的任务数量。每个工作线程都必须绑定一个有效P才被允许执行任务，否则只能休眠，直到有空闲P被唤醒。P还为线程提供执行资源，比如对象分配内存，本地任务队列等。线程独享所绑定的P资源，可在无锁状态下执行高效操作。
+ G(Goroutine)：基本上，进程内的一切都在以goroutine的方式运行，包括运行时相关服务，以及`main.main`入口函数。需要指出，G并非执行体，它仅仅保存并发任务状态，为任务执行提供所需栈内存空间。G任务创建后被放置在P的本地队列或全局队列，等待工作线程调度执行。
+ M(Machine)：是实际执行体，它和P绑定，以调度循环方式不停执行G并发任务。M通过修改寄存器，将执行栈指向G自带的栈内存，并在此空间内分配堆栈帧，执行任务函数。当需要中途切换时，**只要将相关寄存器值保存回G空间即可维持状态**，任何M都可据此恢复执行。线程仅负责执行，不再持有状态，这是并发任务跨线程调度，实现多路复用的根本所在。
+ 尽管P/M构成执行组合体，但两者数量并非一一对应。通常情况下，P的数量相对恒定，默认与CPU核数量相同，但也可能更多或更少，而M则是由调度器按需创建的。举例来说，当M因陷入系统调用而长时间阻塞时，P就会被监控线程抢回，去新建（或唤醒）一个M执行其他任务，这样M的数量就会增长。
+ 因为G初始栈仅有2KB，且创建操作只是在用户空间简单地分配对象，远比进入内核态分配线程要简单得多。调度器让多个M进入调度循环，不停获取并执行任务，所以我们才能创建成千上万个并发任务。



## 8.1.调度器初始化

位于`proc.go`的`schedinit`中。如下：

```go
// The bootstrap sequence is:
//
//	call osinit
//	call schedinit
//	make & queue new G
//	call runtime·mstart
//
// The new G calls runtime·main.
func schedinit() {
	// raceinit must be the first call to race detector.
	// In particular, it must be done before mallocinit below calls racemapshadow.
	_g_ := getg()
	if raceenabled {
		_g_.racectx, raceprocctx0 = raceinit()
	}

  // 设置最大M的数量
	sched.maxmcount = 10000

	tracebackinit()
	moduledataverify()
  // 初始化栈空间复用管理链表
	stackinit()
	mallocinit()
	mcommoninit(_g_.m)
	cpuinit()       // must run before alginit
	alginit()       // maps must not be used before this call
	modulesinit()   // provides activeModules
	typelinksinit() // uses maps, activeModules
	itabsinit()     // uses activeModules

	msigsave(_g_.m)
	initSigmask = _g_.m.sigmask

	goargs()
	goenvs()
	parsedebugvars()
	gcinit()

	sched.lastpoll = uint64(nanotime())
  // 默认值从1调整为CPU core的数量
	procs := ncpu
	if n, ok := atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
		procs = n
	}
  
  // 调整P数量
  // 注意：此刻所有P都是新建的，所以不可能返回有本地任务的P
	if procresize(procs) != nil {
		throw("unknown runnable goroutine during bootstrap")
	}

	// For cgocheck > 1, we turn on the write barrier at all times
	// and check all pointer writes. We can't do this until after
	// procresize because the write barrier needs a P.
	if debug.cgocheck > 1 {
		writeBarrier.cgo = true
		writeBarrier.enabled = true
		for _, p := range allp {
			p.wbBuf.reset()
		}
	}

	if buildVersion == "" {
		// Condition should never trigger. This code just serves
		// to ensure runtime·buildVersion is kept in the resulting binary.
		buildVersion = "unknown"
	}
	if len(modinfo) == 1 {
		// Condition should never trigger. This code just serves
		// to ensure runtime·modinfo is kept in the resulting binary.
		modinfo = ""
	}
}
```

P结构的数量在`runtime2.go`，如下：

```go
var (
	allp       []*p  // len(allp) == gomaxprocs; may change at safe points, otherwise immutable
)

type schedt struct {
	// accessed atomically. keep at top to ensure alignment on 32-bit systems.
	goidgen  uint64
	lastpoll uint64

	lock mutex

	// When increasing nmidle, nmidlelocked, nmsys, or nmfreed, be
	// sure to call checkdead().

	midle        muintptr // idle m's waiting for work
	nmidle       int32    // number of idle m's waiting for work
	nmidlelocked int32    // number of locked m's waiting for work
	mnext        int64    // number of m's that have been created and next M ID
	maxmcount    int32    // maximum number of m's allowed (or die)
	nmsys        int32    // number of system m's not counted for deadlock
	nmfreed      int64    // cumulative number of freed m's

	ngsys uint32 // number of system goroutines; updated atomically

	pidle      puintptr // 空闲p链表
	npidle     uint32  // 空闲p数量
	nmspinning uint32 // See "Worker thread parking/unparking" comment in proc.go.

	// Global runnable queue.
	runq     gQueue
	runqsize int32

	// disable controls selective disabling of the scheduler.
	//
	// Use schedEnableUser to control this.
	//
	// disable is protected by sched.lock.
	disable struct {
		// user disables scheduling of user goroutines.
		user     bool
		runnable gQueue // pending runnable Gs
		n        int32  // length of runnable
	}

	// Global cache of dead G's.
	gFree struct {
		lock    mutex
		stack   gList // Gs with stacks
		noStack gList // Gs without stacks
		n       int32
	}

	// Central cache of sudog structs.
	sudoglock  mutex
	sudogcache *sudog

	// Central pool of available defer structs of different sizes.
	deferlock mutex
	deferpool [5]*_defer

	// freem is the list of m's waiting to be freed when their
	// m.exited is set. Linked through m.freelink.
	freem *m

	gcwaiting  uint32 // gc is waiting to run
	stopwait   int32
	stopnote   note
	sysmonwait uint32
	sysmonnote note

	// safepointFn should be called on each P at the next GC
	// safepoint if p.runSafePointFn is set.
	safePointFn   func(*p)
	safePointWait int32
	safePointNote note

	profilehz int32 // cpu profiling rate

	procresizetime int64 // nanotime() of last change to gomaxprocs
	totaltime      int64 // ∫gomaxprocs dt up to procresizetime
}

```

再看看`procresize`的实现：`proc.go`

```go
// Change number of processors. The world is stopped, sched is locked.
// gcworkbufs are not being modified by either the GC or
// the write barrier code.
// Returns list of Ps with local work, they need to be scheduled by the caller.
func procresize(nprocs int32) *p {
	old := gomaxprocs
	if old < 0 || nprocs <= 0 {
		throw("procresize: invalid arg")
	}
	if trace.enabled {
		traceGomaxprocs(nprocs)
	}

	// update statistics
	now := nanotime()
	if sched.procresizetime != 0 {
		sched.totaltime += int64(old) * (now - sched.procresizetime)
	}
	sched.procresizetime = now

	// Grow allp if necessary.
  // allp数量不够，进行扩容
	if nprocs > int32(len(allp)) {
		// Synchronize with retake, which could be running
		// concurrently since it doesn't run on a P.
		lock(&allpLock)
		if nprocs <= int32(cap(allp)) {
			allp = allp[:nprocs]
		} else {
			nallp := make([]*p, nprocs)
			// Copy everything up to allp's cap so we
			// never lose old allocated Ps.
			copy(nallp, allp[:cap(allp)])
			allp = nallp
		}
		unlock(&allpLock)
	}

	// initialize new P's
  // 新增P
	for i := old; i < nprocs; i++ {
		pp := allp[i]
		if pp == nil {  // 申请新P对象
			pp = new(p)
		}
    // init方法的实现见下方
		pp.init(i)
    // 保存到allp切片中
		atomicstorep(unsafe.Pointer(&allp[i]), unsafe.Pointer(pp))
	}

	_g_ := getg()
  
  // 如果当前正在用的P属于被释放的那拨，那就换成allp[0]
	if _g_.m.p != 0 && _g_.m.p.ptr().id < nprocs {
		// continue to use the current P
    // 继续使用当前P
		_g_.m.p.ptr().status = _Prunning
		_g_.m.p.ptr().mcache.prepareForSweep()
	} else {
		// release the current P and acquire allp[0].
		//
		// We must do this before destroying our current P
		// because p.destroy itself has write barriers, so we
		// need to do that from a valid P.
    // 释放当前P，因为它已经失效
		if _g_.m.p != 0 {
			if trace.enabled {
				// Pretend that we were descheduled
				// and then scheduled again to keep
				// the trace sane.
				traceGoSched()
				traceProcStop(_g_.m.p.ptr())
			}
			_g_.m.p.ptr().m = 0
		}
		_g_.m.p = 0
		_g_.m.mcache = nil
    // 换成allp[0]
		p := allp[0]
		p.m = 0
		p.status = _Pidle
		acquirep(p)
		if trace.enabled {
			traceGoStart()
		}
	}

	// release resources from unused P's
  // 释放多余的P
	for i := nprocs; i < old; i++ {
		p := allp[i]
    // destroy方法见下方
		p.destroy()
		// can't free P itself because it can be referenced by an M in syscall
	}

	// Trim allp.
	if int32(len(allp)) != nprocs {
		lock(&allpLock)
		allp = allp[:nprocs]
		unlock(&allpLock)
	}

  // 将没有本地任务的P放到空闲链表
	var runnablePs *p
	for i := nprocs - 1; i >= 0; i-- {
		p := allp[i]
    // 确保不是当前正在用的P
		if _g_.m.p.ptr() == p {
			continue
		}
		p.status = _Pidle
		if runqempty(p) {
      // 放入空闲链表（具体实现见下方）
			pidleput(p)
		} else {
      // 有本地任务，构建链表
			p.m.set(mget())
			p.link.set(runnablePs)
			runnablePs = p
		}
	}
	stealOrder.reset(uint32(nprocs))
	var int32p *int32 = &gomaxprocs // make compiler check that gomaxprocs is an int32
	atomic.Store((*uint32)(unsafe.Pointer(int32p)), uint32(nprocs))
  // 返回有本地任务的P（链表）
	return runnablePs
}

// init initializes pp, which may be a freshly allocated p or a
// previously destroyed p, and transitions it to status _Pgcstop.
func (pp *p) init(id int32) {
	pp.id = id
	pp.status = _Pgcstop
	pp.sudogcache = pp.sudogbuf[:0]
	for i := range pp.deferpool {
		pp.deferpool[i] = pp.deferpoolbuf[i][:0]
	}
	pp.wbBuf.reset()
  // 为p保存cache对象
	if pp.mcache == nil {
		if id == 0 {
			if getg().m.mcache == nil {
				throw("missing mcache?")
			}
      pp.mcache = getg().m.mcache // bootstrap(引导程序)
		} else {
      // 创建cache
			pp.mcache = allocmcache()
		}
	}
	if raceenabled && pp.raceprocctx == 0 {
		if id == 0 {
			pp.raceprocctx = raceprocctx0
			raceprocctx0 = 0 // bootstrap
		} else {
			pp.raceprocctx = raceproccreate()
		}
	}
}


// destroy releases all of the resources associated with pp and
// transitions it to status _Pdead.
//
// sched.lock must be held and the world must be stopped.
func (pp *p) destroy() {
	// Move all runnable goroutines to the global queue
  // 将本地任务转移到全局队列
	for pp.runqhead != pp.runqtail {
		// Pop from tail of local queue
		pp.runqtail--
		gp := pp.runq[pp.runqtail%uint32(len(pp.runq))].ptr()
		// Push onto head of global queue
		globrunqputhead(gp)
	}
	if pp.runnext != 0 {
		globrunqputhead(pp.runnext.ptr())
		pp.runnext = 0
	}
	// If there's a background worker, make it runnable and put
	// it on the global queue so it can clean itself up.
	if gp := pp.gcBgMarkWorker.ptr(); gp != nil {
		casgstatus(gp, _Gwaiting, _Grunnable)
		if trace.enabled {
			traceGoUnpark(gp, 0)
		}
		globrunqput(gp)
		// This assignment doesn't race because the
		// world is stopped.
		pp.gcBgMarkWorker.set(nil)
	}
	// Flush p's write barrier buffer.
	if gcphase != _GCoff {
		wbBufFlush1(pp)
		pp.gcw.dispose()
	}
	for i := range pp.sudogbuf {
		pp.sudogbuf[i] = nil
	}
	pp.sudogcache = pp.sudogbuf[:0]
	for i := range pp.deferpool {
		for j := range pp.deferpoolbuf[i] {
			pp.deferpoolbuf[i][j] = nil
		}
		pp.deferpool[i] = pp.deferpoolbuf[i][:0]
	}
  // 释放当前P绑定的cache
	freemcache(pp.mcache)
	pp.mcache = nil
  // 将当前P的G复用链表转移到全局
	gfpurge(pp)
	traceProcFree(pp)
	if raceenabled {
		raceprocdestroy(pp.raceprocctx)
		pp.raceprocctx = 0
	}
	pp.gcAssistTime = 0
	pp.status = _Pdead
}


// Put p to on _Pidle list.
// Sched must be locked.
// May run during STW, so write barriers are not allowed.
//go:nowritebarrierrec
// 将P放入空闲链表
func pidleput(_p_ *p) {
	if !runqempty(_p_) {
		throw("pidleput: P has non-empty run queue")
	}
	_p_.link = sched.pidle
	sched.pidle.set(_p_)
	atomic.Xadd(&sched.npidle, 1) // TODO: fast atomic
}
```



上面的`processsize`函数，只有`schedinit`和`startTheWorldWithSema`才会调用。在调度器初始化阶段，所有P对象都是新建的。除分配给当前主线程的外，其他都被放入空闲链表。而`startTheWorld`会激活全部有本地任务的P对象。

在完成调度器初始化后，引导过程才创建并允许main goroutine。如下：`asm_amd64.s`

```pascal
TEXT runtime·rt0_go(SB),NOSPLIT,$0
	// copy arguments forward on an even stack
	MOVQ	DI, AX		// argc
	MOVQ	SI, BX		// argv
	SUBQ	$(4*8+7), SP		// 2args 2auto
	ANDQ	$~15, SP
	MOVQ	AX, 16(SP)
	MOVQ	BX, 24(SP)

	// create istack out of the given (operating system) stack.
	// _cgo_init may update stackguard.
	MOVQ	$runtime·g0(SB), DI
	LEAQ	(-64*1024+104)(SP), BX
	MOVQ	BX, g_stackguard0(DI)
	MOVQ	BX, g_stackguard1(DI)
	MOVQ	BX, (g_stack+stack_lo)(DI)
	MOVQ	SP, (g_stack+stack_hi)(DI)

	// find out information about the processor we're on
	MOVL	$0, AX
	CPUID
	MOVL	AX, SI
	CMPL	AX, $0
	JE	nocpuinfo

	// Figure out how to serialize RDTSC.
	// On Intel processors LFENCE is enough. AMD requires MFENCE.
	// Don't know about the rest, so let's do MFENCE.
	CMPL	BX, $0x756E6547  // "Genu"
	JNE	notintel
	CMPL	DX, $0x49656E69  // "ineI"
	JNE	notintel
	CMPL	CX, $0x6C65746E  // "ntel"
	JNE	notintel
	MOVB	$1, runtime·isIntel(SB)
	MOVB	$1, runtime·lfenceBeforeRdtsc(SB)
notintel:

	// Load EAX=1 cpuid flags
	MOVL	$1, AX
	CPUID
	MOVL	AX, runtime·processorVersionInfo(SB)

nocpuinfo:
	// if there is an _cgo_init, call it.
	MOVQ	_cgo_init(SB), AX
	TESTQ	AX, AX
	JZ	needtls
	// arg 1: g0, already in DI
	MOVQ	$setg_gcc<>(SB), SI // arg 2: setg_gcc
#ifdef GOOS_android
	MOVQ	$runtime·tls_g(SB), DX 	// arg 3: &tls_g
	// arg 4: TLS base, stored in slot 0 (Android's TLS_SLOT_SELF).
	// Compensate for tls_g (+16).
	MOVQ	-16(TLS), CX
#else
	MOVQ	$0, DX	// arg 3, 4: not used when using platform's TLS
	MOVQ	$0, CX
#endif
#ifdef GOOS_windows
	// Adjust for the Win64 calling convention.
	MOVQ	CX, R9 // arg 4
	MOVQ	DX, R8 // arg 3
	MOVQ	SI, DX // arg 2
	MOVQ	DI, CX // arg 1
#endif
	CALL	AX

	// update stackguard after _cgo_init
	MOVQ	$runtime·g0(SB), CX
	MOVQ	(g_stack+stack_lo)(CX), AX
	ADDQ	$const__StackGuard, AX
	MOVQ	AX, g_stackguard0(CX)
	MOVQ	AX, g_stackguard1(CX)

#ifndef GOOS_windows
	JMP ok
#endif
needtls:
#ifdef GOOS_plan9
	// skip TLS setup on Plan 9
	JMP ok
#endif
#ifdef GOOS_solaris
	// skip TLS setup on Solaris
	JMP ok
#endif
#ifdef GOOS_illumos
	// skip TLS setup on illumos
	JMP ok
#endif
#ifdef GOOS_darwin
	// skip TLS setup on Darwin
	JMP ok
#endif

	LEAQ	runtime·m0+m_tls(SB), DI
	CALL	runtime·settls(SB)

	// store through it, to make sure it works
	get_tls(BX)
	MOVQ	$0x123, g(BX)
	MOVQ	runtime·m0+m_tls(SB), AX
	CMPQ	AX, $0x123
	JEQ 2(PC)
	CALL	runtime·abort(SB)
ok:
	// set the per-goroutine and per-mach "registers"
	get_tls(BX)
	LEAQ	runtime·g0(SB), CX
	MOVQ	CX, g(BX)
	LEAQ	runtime·m0(SB), AX

	// save m->g0 = g0
	MOVQ	CX, m_g0(AX)
	// save m0 to g0->m
	MOVQ	AX, g_m(CX)

	CLD				// convention is D is always left cleared
	CALL	runtime·check(SB)

	// 重点：这一段
	MOVL	16(SP), AX		// copy argc
	MOVL	AX, 0(SP)
	MOVQ	24(SP), AX		// copy argv
	MOVQ	AX, 8(SP)
	CALL	runtime·args(SB)
	CALL	runtime·osinit(SB)
	CALL	runtime·schedinit(SB)

	// create a new goroutine to start program
	// 创建main.goroutine，并将其放入当前P本地队列
	MOVQ	$runtime·mainPC(SB), AX		// entry
	PUSHQ	AX
	PUSHQ	$0			// arg size
	CALL	runtime·newproc(SB)
	POPQ	AX
	POPQ	AX

	// start this M
	// 让当前M0进入调度，执行main goroutine
	CALL	runtime·mstart(SB)

	CALL	runtime·abort(SB)	// mstart should never return
	RET

	// Prevent dead-code elimination of debugCallV1, which is
	// intended to be called by debuggers.
	MOVQ	$runtime·debugCallV1(SB), AX
	RET
```

虽然可在运行期用`runtime.GOMAXPROCS`函数修改P的数量，但须付出极大代价。

```go
func GOMAXPROCS(n int) int {
	if GOARCH == "wasm" && n > 1 {
		n = 1 // WebAssembly has no threads yet, so only one CPU is possible.
	}

	lock(&sched.lock)
  // 返回当前值
	ret := int(gomaxprocs)
	unlock(&sched.lock)
	if n <= 0 || n == ret {
		return ret
	}

  // STW
	stopTheWorld("GOMAXPROCS")

	// newprocs will be processed by startTheWorld
	newprocs = int32(n)

  // 里面调用startTheWorldWithSema，startTheWorldWithSema里调用了procresize，激活了有任务的P
	startTheWorld()
	return ret
}
```



## 8.2.任务

我们通过反汇编，看下`go func(...)`这个语法做了啥。

test.go如下：

```go
package main

func add(x, y int) int {
    z := x + y
    return z
}

func main() {
    x := 0x100
    y := 0x200
    go add(x, y)
}
```

进行反汇编：

```bash
[sysublackbear@centos-linux zhuowen.deng]$ go build -gcflags "-N -l" -o test test.go
[sysublackbear@centos-linux zhuowen.deng]$ go tool objdump -s "main\.main" test
TEXT main.main(SB) /home/sysublackbear/go/src/zhuowen.deng/test.go
  test.go:8		0x44c190		64488b0c25f8ffffff	MOVQ FS:0xfffffff8, CX
  test.go:8		0x44c199		483b6110		CMPQ 0x10(CX), SP
  test.go:8		0x44c19d		7656			JBE 0x44c1f5
  test.go:8		0x44c19f		4883ec40		SUBQ $0x40, SP
  test.go:8		0x44c1a3		48896c2438		MOVQ BP, 0x38(SP)
  test.go:8		0x44c1a8		488d6c2438		LEAQ 0x38(SP), BP
  test.go:9		0x44c1ad		48c744243000010000	MOVQ $0x100, 0x30(SP)  
  test.go:10		0x44c1b6		48c744242800020000	MOVQ $0x200, 0x28(SP)  
  test.go:11		0x44c1bf		488b442430		MOVQ 0x30(SP), AX  // 实参x入栈
  test.go:11		0x44c1c4		4889442410		MOVQ AX, 0x10(SP)
  test.go:11		0x44c1c9		488b442428		MOVQ 0x28(SP), AX  // 实参y入栈
  test.go:11		0x44c1ce		4889442418		MOVQ AX, 0x18(SP)
  test.go:11		0x44c1d3		c7042418000000		MOVL $0x18, 0(SP)  // 参数长度入栈
  test.go:11		0x44c1da		488d054f1d0200		LEAQ 0x21d4f(IP), AX  // 将函数add地址存入AX寄存器
  test.go:11		0x44c1e1		4889442408		MOVQ AX, 0x8(SP)  // 地址入栈
  test.go:11		0x44c1e6		e805e1fdff		CALL runtime.newproc(SB)  // 这里go语法，实质调用了newproc方法。
  test.go:12		0x44c1eb		488b6c2438		MOVQ 0x38(SP), BP
  test.go:12		0x44c1f0		4883c440		ADDQ $0x40, SP
  test.go:12		0x44c1f4		c3			RET
  test.go:8		0x44c1f5		e87683ffff		CALL runtime.morestack_noctxt(SB)
  test.go:8		0x44c1fa		eb94			JMP main.main(SB)
```

由调用方负责提供参数空间，并从右往左入栈。

看看runtime.newproc方法：

```go
func newproc(siz int32, fn *funcval) {
  // 获取第一参数地址
	argp := add(unsafe.Pointer(&fn), sys.PtrSize)
	gp := getg()
  // 获取调用方PC/IP寄存器值
	pc := getcallerpc()
  // 用g0栈创建G/goroutine对象
	systemstack(func() {
		newproc1(fn, (*uint8)(argp), siz, gp, pc)
	})
}
```

可以看到，`newproc`只有两个参数，但是`main`函数却向栈压入了4个值。按照顺序，后三个值应该会被合并成`funcval`。还有，`add`返回值会被忽略。



`funcval`的结构如下：

```go
type funcval struct {
	fn uintptr
	// variable-size, fn-specific data here
}
```

属于变长结构，此处其补全状态应该是：

```go
type struct {
  fn uintptr
  x  int
  y  int
}
```

如此一来，关于”go语言会复制参数值“的规则就很好理解了。站在`newproc`的角度，我们可以画出执行栈的状态示意图：

![image-20191105164831001](https://github.com/sysublackbear/golang_runtime_analysis/blob/master/img/image-20191105164831001.png)

最开始进入到了函数`add`的地址，`add(unsafe,Pointer(&fn), ptrSize)`跳过了size这个参数，获得了第一个参数x的地址，`getcallerpc`用size - 8 读取 CALL指令压入的 main PC/IP寄存器值，这就是 `newproc`为`newproc1`准备的相关参数值。如下：

**注意:**`getcallerpc`这个函数的实现没有找到，先备注。

然后我们再看`newproc1`的实现：

```go
// Create a new g running fn with narg bytes of arguments starting
// at argp. callerpc is the address of the go statement that created
// this. The new g is put on the queue of g's waiting to run.
func newproc1(fn *funcval, argp *uint8, narg int32, callergp *g, callerpc uintptr) {
	_g_ := getg()

	if fn == nil {
		_g_.m.throwing = -1 // do not dump full stacks
		throw("go of nil func value")
	}
	acquirem() // disable preemption because it can be holding p in a local var
  // ”参数+返回值“所需空间进行对齐
	siz := narg
	siz = (siz + 7) &^ 7

	// We could allocate a larger initial stack if necessary.
	// Not worth it: this is almost always an error.
	// 4*sizeof(uintreg): extra space added below
	// sizeof(uintreg): caller's LR (arm) or return address (x86, in gostartcall).
	if siz >= _StackMin-4*sys.RegSize-sys.RegSize {
		throw("newproc: function arguments too large for new goroutine")
	}

  // 从当前P复用链表获取空闲的G对象。
	_p_ := _g_.m.p.ptr()
	newg := gfget(_p_)
  // 获取失败，进行新建
	if newg == nil {
		newg = malg(_StackMin)
		casgstatus(newg, _Gidle, _Gdead)
		allgadd(newg) // publishes with a g->status of Gdead so GC scanner doesn't look at uninitialized stack.
	}
  // 测试G stack
	if newg.stack.hi == 0 {
		throw("newproc1: newg missing stack")
	}

  // 测试G status
	if readgstatus(newg) != _Gdead {
		throw("newproc1: new g is not Gdead")
	}

  // 计算所需空间大小，并对齐
	totalSize := 4*sys.RegSize + uintptr(siz) + sys.MinFrameSize // extra space in case of reads slightly beyond frame
	totalSize += -totalSize & (sys.SpAlign - 1)                  // align to spAlign
	
  // 确定SP和参数入栈位置
  sp := newg.stack.hi - totalSize
	spArg := sp
	if usesLR {
		// caller's LR
		*(*uintptr)(unsafe.Pointer(sp)) = 0
		prepGoExitFrame(sp)
		spArg += sys.MinFrameSize
	}
	if narg > 0 {
    // 将执行参数拷贝入栈
		memmove(unsafe.Pointer(spArg), unsafe.Pointer(argp), uintptr(narg))
		// This is a stack-to-stack copy. If write barriers
		// are enabled and the source stack is grey (the
		// destination is always black), then perform a
		// barrier copy. We do this *after* the memmove
		// because the destination stack may have garbage on
		// it.
		if writeBarrier.needed && !_g_.m.curg.gcscandone {
			f := findfunc(fn.fn)
			stkmap := (*stackmap)(funcdata(f, _FUNCDATA_ArgsPointerMaps))
			if stkmap.nbit > 0 {
				// We're in the prologue, so it's always stack map index 0.
				bv := stackmapdata(stkmap, 0)
				bulkBarrierBitmap(spArg, spArg, uintptr(bv.n)*sys.PtrSize, 0, bv.bytedata)
			}
		}
	}

  // 初始化用于保存执行现场的区域
	memclrNoHeapPointers(unsafe.Pointer(&newg.sched), unsafe.Sizeof(newg.sched))
  
  // 此处保存的是 goexit 地址
	newg.sched.sp = sp
	newg.stktopsp = sp
	newg.sched.pc = funcPC(goexit) + sys.PCQuantum // +PCQuantum so that previous instruction is in same function
	newg.sched.g = guintptr(unsafe.Pointer(newg))
  // 此处的调用时关键所在
	gostartcallfn(&newg.sched, fn)
  
  // 初始化基本状态
	newg.gopc = callerpc
	newg.ancestors = saveAncestors(callergp)
	newg.startpc = fn.fn
	if _g_.m.curg != nil {
		newg.labels = _g_.m.curg.labels
	}
	if isSystemGoroutine(newg, false) {
		atomic.Xadd(&sched.ngsys, +1)
	}
	newg.gcscanvalid = false
	casgstatus(newg, _Gdead, _Grunnable)

  // 设置唯一id
	if _p_.goidcache == _p_.goidcacheend {
		// Sched.goidgen is the last allocated id,
		// this batch must be [sched.goidgen+1, sched.goidgen+GoidCacheBatch].
		// At startup sched.goidgen=0, so main goroutine receives goid=1.
    // sched.goidgen 是一个全局计数器
    // 每次取回一段有效区间，然后在该区间分配，避免频繁地去全局操作
    // 到全局取回一段区间（全局计数变量goidgen增加一段值）（这种实现很值得学习）
		_p_.goidcache = atomic.Xadd64(&sched.goidgen, _GoidCacheBatch)
		_p_.goidcache -= _GoidCacheBatch - 1
		_p_.goidcacheend = _p_.goidcache + _GoidCacheBatch
	}
	newg.goid = int64(_p_.goidcache)
	_p_.goidcache++
	if raceenabled {
		newg.racectx = racegostart(callerpc)
	}
	if trace.enabled {
		traceGoCreate(newg, newg.startpc)
	}
  
  // 将G放入待运行队列
	runqput(_p_, newg, true)

  // 如果有其他空闲P，则尝试唤醒某个M出来执行任务。
  // 如果有M处于自旋等待P或G状态
  // 如果当前创建的是main goroutine（runtime.main)，那么还没有其他任务需要执行，放弃（因为主goroutine都没初始化好）
	if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 && mainStarted {
		wakep()
	}
	releasem(_g_.m)
}
```



针对上面的程序一一分析：

1. G对象默认会复用，这个看上去有点像 cache/object 的做法。除 P 本地的复用链表外，还有全局链表在多个P之间共享。

```go
// Get from gfree list.
// If local list is empty, grab a batch from global list.
func gfget(_p_ *p) *g {
retry:
  // 如果提取失败，尝试从全局链表转移一批对象到P本地
	if _p_.gFree.empty() && (!sched.gFree.stack.empty() || !sched.gFree.noStack.empty()) {
		lock(&sched.gFree.lock)
		// Move a batch of free Gs to the P.
    // 最多转移32个
		for _p_.gFree.n < 32 {
			// Prefer Gs with stacks.
			gp := sched.gFree.stack.pop()
			if gp == nil {
				gp = sched.gFree.noStack.pop()
				if gp == nil {
					break
				}
			}
			sched.gFree.n--
			_p_.gFree.push(gp)
			_p_.gFree.n++
		}
		unlock(&sched.gFree.lock)
    // 再次进入判断
		goto retry
	}
  // 从P本地队列提取复用对象
	gp := _p_.gFree.pop()
	if gp == nil {
		return nil
	}
  // 调整P复用链表
	_p_.gFree.n--
  // 检查G stack
	if gp.stack.lo == 0 {
		// Stack was deallocated in gfput. Allocate a new one.
    // 分配新栈
		systemstack(func() {
			gp.stack = stackalloc(_FixedStack)
		})
		gp.stackguard0 = gp.stack.lo + _StackGuard
	} else {
		if raceenabled {
			racemalloc(unsafe.Pointer(gp.stack.lo), gp.stack.hi-gp.stack.lo)
		}
		if msanenabled {
			msanmalloc(unsafe.Pointer(gp.stack.lo), gp.stack.hi-gp.stack.lo)
		}
	}
	return gp
```

而当goroutine执行完毕，调度器相关函数会将G对象放回P复用链表。实现在`proc1.go`的`gfput`函数中：

```go
// Put on gfree list.
// If local list is too long, transfer a batch to the global list.
func gfput(_p_ *p, gp *g) {
	if readgstatus(gp) != _Gdead {
		throw("gfput: bad status (not Gdead)")
	}

	stksize := gp.stack.hi - gp.stack.lo

	if stksize != _FixedStack {
		// non-standard stack size - free it.
		stackfree(gp.stack)
		gp.stack.lo = 0
		gp.stack.hi = 0
		gp.stackguard0 = 0
	}

  // 放回 P本地复用链表
	_p_.gFree.push(gp)
	_p_.gFree.n++
  
  // 如果本地复用对象过多，则转移一批到全局链表。
	if _p_.gFree.n >= 64 {
		lock(&sched.gFree.lock)
		for _p_.gFree.n >= 32 {
			_p_.gFree.n--
			gp = _p_.gFree.pop()
			if gp.stack.lo == 0 {
				sched.gFree.noStack.push(gp)
			} else {
				sched.gFree.stack.push(gp)
			}
			sched.gFree.n++
		}
		unlock(&sched.gFree.lock)
	}
}
```



2. G对象的新建都是通过`malg`创建。

```go
// Allocate a new g, with a stack big enough for stacksize bytes.
func malg(stacksize int32) *g {
	newg := new(g)
	if stacksize >= 0 {
		stacksize = round2(_StackSystem + stacksize)
		systemstack(func() {
			newg.stack = stackalloc(uint32(stacksize))
		})
		newg.stackguard0 = newg.stack.lo + _StackGuard
		newg.stackguard1 = ^uintptr(0)
	}
	return newg
}
```

默认采用8kb栈空间。然后被allg引用。这是垃圾回收遍历扫描的需要，以便获取指针引用，收缩栈空间。详见`allgadd`：

```go
func allgadd(gp *g) {
	if readgstatus(gp) == _Gidle {
		throw("allgadd: bad status Gidle")
	}

	lock(&allglock)
	allgs = append(allgs, gp)
	allglen = uintptr(len(allgs))
	unlock(&allglock)
}
```

在获取到G对象之后，`newproc1`会进行一系列初始化操作，毕竟不管新建还是复用，这些参数都必须正确地设置。同时，相关执行参数会被拷贝到G的栈空间，因为它和当前任务不再有任何关系，各自使用独立的栈空间。毕竟，`go func(...)`语句仅创建并发任务，当前流程会继续自己的逻辑。



3. 创建完毕的G任务被优先放入P本地队列等待执行，这属于无锁操作，详见`runqput`

```go
// runqput tries to put g on the local runnable queue.
// If next is false, runqput adds g to the tail of the runnable queue.
// If next is true, runqput puts g in the _p_.runnext slot.
// If the run queue is full, runnext puts g on the global queue.
// Executed only by the owner P.
func runqput(_p_ *p, gp *g, next bool) {
	if randomizeScheduler && next && fastrand()%2 == 0 {
		next = false
	}

  // 如果可能，将G直接保存在 P.runnext，作为下一个优先执行任务
	if next {
	retryNext:
		oldnext := _p_.runnext
		if !_p_.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
			goto retryNext
		}
		if oldnext == 0 {
			return
		}
		// Kick the old runnext out to the regular run queue.
    // 原本的next G会被放回本地队列
		gp = oldnext.ptr()
	}

retry:
  // runqhead是一个数组实现的循环队列
  // head, tail累加，通过取模即可获得索引位置，很典型的算法
	h := atomic.LoadAcq(&_p_.runqhead) // load-acquire, synchronize with consumers
	t := _p_.runqtail
  
  // 如果本地队列未满，直接放到尾部
	if t-h < uint32(len(_p_.runq)) {
		_p_.runq[t%uint32(len(_p_.runq))].set(gp)
		atomic.StoreRel(&_p_.runqtail, t+1) // store-release, makes the item available for consumption
		return
	}
  
  // 本地队列满了，放入全局队列
  // 因为需要加锁，所以叫slow
	if runqputslow(_p_, gp, h, t) {
		return
	}
	// the queue is not full, now the put above must succeed
	goto retry
}
```

任务队列分成三级，按优先级从高到低分别是：`p.runnext`，`p.runq`，`schedt.runq`

再看`runqputslow`的实现：

```go
// Put g and a batch of work from local runnable queue on global queue.
// Executed only by the owner P.
func runqputslow(_p_ *p, gp *g, h, t uint32) bool {
  // 这意思是要从p本地队列转移一半任务到全局队列
  // +1 是算上当前这个gp
	var batch [len(_p_.runq)/2 + 1]*g

	// First, grab a batch from local queue.
  // 计算一半的实际量
	n := t - h
	n = n / 2
	if n != uint32(len(_p_.runq)/2) {
		throw("runqputslow: queue is not full")
	}
  
  // 从队列头部提取
	for i := uint32(0); i < n; i++ {
		batch[i] = _p_.runq[(h+i)%uint32(len(_p_.runq))].ptr()
	}
  
  // 调整P队列头部位置
	if !atomic.CasRel(&_p_.runqhead, h, h+n) { // cas-release, commits consume
		return false
	}
  // 加上当前gp这家伙
	batch[n] = gp

  // 对顺序进行洗牌
	if randomizeScheduler {
		for i := uint32(1); i <= n; i++ {
			j := fastrandn(i + 1)
			batch[i], batch[j] = batch[j], batch[i]
		}
	}

	// Link the goroutines.
  // 串成链表
	for i := uint32(0); i < n; i++ {
		batch[i].schedlink.set(batch[i+1])
	}
	var q gQueue
	q.head.set(batch[0])
	q.tail.set(batch[n])

	// Now put the batch on global queue.
	lock(&sched.lock)
  // 添加到全局队列尾部
	globrunqputbatch(&q, int32(n+1))
	unlock(&sched.lock)
	return true
}

// Put a batch of runnable goroutines on the global runnable queue.
// This clears *batch.
// Sched must be locked.
func globrunqputbatch(batch *gQueue, n int32) {
	sched.runq.pushBackAll(*batch)
	sched.runqsize += n
	*batch = gQueue{}
}

// pushBackAll adds all Gs in l2 to the tail of q. After this q2 must
// not be used.
func (q *gQueue) pushBackAll(q2 gQueue) {
	if q2.tail == 0 {
		return
	}
	q2.tail.ptr().schedlink = 0
	if q.tail != 0 {
		q.tail.ptr().schedlink = q2.head
	} else {
		q.head = q2.head
	}
	q.tail = q2.tail
}
```

若本地队列已满，一次性转移半数到全局队列。这个好理解，因为其他P可能正饿着呢。这也正好解释了`newproc1`最后尝试用`wakep`唤醒其他M/P去执行任务的意图，毕竟充分发挥多核优势才是正途。



G的状态切换过程：

```bash
-- gfree -----+

--> IDLE --> DEAD --> RUNNABLE --> RUNNING --> DEAD --- ... --> gfree -->
    新建     初始化前   初始化后      调度执行     执行完毕
```



## 8.3.线程

当`newproc1`成功创建G任务后，会尝试用`wakep`唤醒M执行任务。

```go
// Tries to add one more P to execute G's.
// Called when a G is made runnable (newproc, ready).
func wakep() {
	// be conservative about spinning threads
  // 被绑定的线程需要绑定P，累加自旋计数，避免newproc1 唤醒过多线程
	if !atomic.Cas(&sched.nmspinning, 0, 1) {
		return
	}
	startm(nil, true)
}


// Schedules some M to run the p (creates an M if necessary).
// If p==nil, tries to get an idle P, if no idle P's does nothing.
// May run with m.p==nil, so write barriers are not allowed.
// If spinning is set, the caller has incremented nmspinning and startm will
// either decrement nmspinning or set m.spinning in the newly started M.
//go:nowritebarrierrec
func startm(_p_ *p, spinning bool) {
	lock(&sched.lock)
  // 如果没有指定P，
	if _p_ == nil {
		_p_ = pidleget()
    // 获取失败，终止
		if _p_ == nil {
			unlock(&sched.lock)
      // 递减自旋计数
			if spinning {
				// The caller incremented nmspinning, but there are no idle Ps,
				// so it's okay to just undo the increment and give up.
				if int32(atomic.Xadd(&sched.nmspinning, -1)) < 0 {
					throw("startm: negative nmspinning")
				}
			}
			return
		}
	}
  // 获取休眠的闲置M
	mp := mget()
	unlock(&sched.lock)
  // 如果没有闲置M，新建
	if mp == nil {
    // 默认启动函数
    // 主要是判断M.nextp是否有暂存的P，以此调整自旋计数
		var fn func()
		if spinning {
			// The caller incremented nmspinning, so set m.spinning in the new M.
			fn = mspinning
		}
		newm(fn, _p_)
		return
	}
	if mp.spinning {
		throw("startm: m is spinning")
	}
	if mp.nextp != 0 {
		throw("startm: m has p")
	}
	if spinning && !runqempty(_p_) {
		throw("startm: p has runnable gs")
	}
	// The caller incremented nmspinning, so set m.spinning in the new M.
  // 设置自旋状态和暂存P
	mp.spinning = spinning
	mp.nextp.set(_p_)
  // 唤醒M
	notewakeup(&mp.park)
}


```



然后再看看，M是如何创建的，如何包装系统线程。

```go
// Create a new m. It will start off with a call to fn, or else the scheduler.
// fn needs to be static and not a heap allocated closure.
// May run with m.p==nil, so write barriers are not allowed.
//go:nowritebarrierrec
func newm(fn func(), _p_ *p) {
  // 创建M对象
	mp := allocm(_p_, fn)
  // 暂存P
	mp.nextp.set(_p_)
	mp.sigmask = initSigmask
	if gp := getg(); gp != nil && gp.m != nil && (gp.m.lockedExt != 0 || gp.m.incgo) && GOOS != "plan9" {
		// We're on a locked M or a thread that may have been
		// started by C. The kernel state of this thread may
		// be strange (the user may have locked it for that
		// purpose). We don't want to clone that into another
		// thread. Instead, ask a known-good thread to create
		// the thread for us.
		//
		// This is disabled on Plan 9. See golang.org/issue/22227.
		//
		// TODO: This may be unnecessary on Windows, which
		// doesn't model thread creation off fork.
		lock(&newmHandoff.lock)
		if newmHandoff.haveTemplateThread == 0 {
			throw("on a locked thread with no template thread")
		}
		mp.schedlink = newmHandoff.newm
		newmHandoff.newm.set(mp)
		if newmHandoff.waiting {
			newmHandoff.waiting = false
			notewakeup(&newmHandoff.wake)
		}
		unlock(&newmHandoff.lock)
		return
	}
	newm1(mp)
}

func newm1(mp *m) {
	if iscgo {
		var ts cgothreadstart
		if _cgo_thread_start == nil {
			throw("_cgo_thread_start missing")
		}
		ts.g.set(mp.g0)
		ts.tls = (*uint64)(unsafe.Pointer(&mp.tls[0]))
		ts.fn = unsafe.Pointer(funcPC(mstart))
		if msanenabled {
			msanwrite(unsafe.Pointer(&ts), unsafe.Sizeof(ts))
		}
		execLock.rlock() // Prevent process clone.
		asmcgocall(_cgo_thread_start, unsafe.Pointer(&ts))
		execLock.runlock()
		return
	}
	execLock.rlock() // Prevent process clone.
  // 创建系统线程
	newosproc(mp)
	execLock.runlock()
}

// Allocate a new m unassociated with any thread.
// Can use p for allocation context if needed.
// fn is recorded as the new m's m.mstartfn.
//
// This function is allowed to have write barriers even if the caller
// isn't because it borrows _p_.
//
//go:yeswritebarrierrec
func allocm(_p_ *p, fn func()) *m {
	_g_ := getg()
	acquirem() // disable GC because it can be called from sysmon
	if _g_.m.p == 0 {
		acquirep(_p_) // temporarily borrow p for mallocs in this function
	}

	// Release the free M list. We need to do this somewhere and
	// this may free up a stack we can use.
	if sched.freem != nil {
		lock(&sched.lock)
		var newList *m
		for freem := sched.freem; freem != nil; {
			if freem.freeWait != 0 {
				next := freem.freelink
				freem.freelink = newList
				newList = freem
				freem = next
				continue
			}
			stackfree(freem.g0.stack)
			freem = freem.freelink
		}
		sched.freem = newList
		unlock(&sched.lock)
	}

	mp := new(m)
	mp.mstartfn = fn  // 启动函数
	mcommoninit(mp)   // 初始化

	// In case of cgo or Solaris or illumos or Darwin, pthread_create will make us a stack.
	// Windows and Plan 9 will layout sched stack on OS stack.
  // 创建g0
	if iscgo || GOOS == "solaris" || GOOS == "illumos" || GOOS == "windows" || GOOS == "plan9" || GOOS == "darwin" {
		mp.g0 = malg(-1)
	} else {
		mp.g0 = malg(8192 * sys.StackGuardMultiplier)
	}
	mp.g0.m = mp

	if _p_ == _g_.m.p.ptr() {
		releasep()
	}
	releasem(_g_.m)

	return mp
}
```

M最特别的就是自带一个名为g0，默认8kb栈内存的G对象属性。它的栈内存地址被传给`newosproc`函数，作为系统线程的默认堆栈空间（并非所有系统都支持）。

**g0的作用：**

在进程执行过程中，有两类代码需要运行。一类自然是用户逻辑，直接使用G栈内存；另一类是运行时管理指令，它并不便于直接在用户栈上执行，因为这需要处理与用户逻辑现场有关的一大堆事务。

举例来说，G任务可在中途暂停，放回队列后由其他M获取执行。如果不更改执行栈，可能会造成多个线程共享内存，从而引发混乱。另外，在执行垃圾回收操作时，如何收缩依旧被线程持有的G栈空间？因此，当需要执行管理指令时，会将线程栈临时切换到g0，与用户逻辑彻底隔离。

比如我们之前看到的`systemstack`这种执行方式。

```go
// func systemstack(fn func())
TEXT runtime·systemstack(SB), NOSPLIT, $0-8
	MOVQ	fn+0(FP), DI	// DI = fn
	get_tls(CX)
	MOVQ	g(CX), AX	// AX = g
	MOVQ	g_m(AX), BX	// BX = m

	CMPQ	AX, m_gsignal(BX)  
	JEQ	noswitch

	MOVQ	m_g0(BX), DX	// DX = g0
	CMPQ	AX, DX   // 如果当前g已经是g0，那么无须切换
	JEQ	noswitch

	CMPQ	AX, m_curg(BX)
	JNE	bad

	// switch stacks
	// save our state in g->sched. Pretend to
	// be systemstack_switch if the G stack is scanned.
	// 将G状态保存到sched
	MOVQ	$runtime·systemstack_switch(SB), SI
	MOVQ	SI, (g_sched+gobuf_pc)(AX)
	MOVQ	SP, (g_sched+gobuf_sp)(AX)
	MOVQ	AX, (g_sched+gobuf_g)(AX)
	MOVQ	BP, (g_sched+gobuf_bp)(AX)

	// switch to g0
	// 切换到g0.stack
	MOVQ	DX, g(CX)  // DX = g0
	MOVQ	(g_sched+gobuf_sp)(DX), BX  // 从g0.sched获取SP
	// make it look like mstart called systemstack on g0, to stop traceback
	SUBQ	$8, BX  // 调整SP
	MOVQ	$runtime·mstart(SB), DX
	MOVQ	DX, 0(BX)
	MOVQ	BX, SP  // 通过调整SP寄存器值切换栈内存

	// call target function
	MOVQ	DI, DX  // DI = fn
	MOVQ	0(DI), DI
	CALL	DI

	// switch back to g
	// 切换回G，恢复执行现场
	get_tls(CX)
	MOVQ	g(CX), AX
	MOVQ	g_m(AX), BX
	MOVQ	m_curg(BX), AX
	MOVQ	AX, g(CX)
	MOVQ	(g_sched+gobuf_sp)(AX), SP
	MOVQ	$0, (g_sched+gobuf_sp)(AX)
	RET

noswitch:
	// already on m stack; tail call the function
	// Using a tail call here cleans up tracebacks since we won't stop
	// at an intermediate systemstack.
	MOVQ	DI, DX
	MOVQ	0(DI), DI
	JMP	DI

bad:
	// Bad: g is not gsignal, not g0, not curg. What is it?
	MOVQ	$runtime·badsystemstack(SB), AX
	CALL	AX
	INT	$3
```



### 8.3.1.M的初始化操作

M初始化操作会检查已有的M数量，如超出最大限制（默认为10000）会导致进程崩溃。所有M被添加到allm链表，且不被释放。

```go
func mcommoninit(mp *m) {
	_g_ := getg()

	// g0 stack won't make sense for user (and is not necessary unwindable).
	if _g_ != _g_.m.g0 {
		callers(1, mp.createstack[:])
	}

	lock(&sched.lock)
	if sched.mnext+1 < sched.mnext {
		throw("runtime: thread ID overflow")
	}
	mp.id = sched.mnext
	sched.mnext++
	checkmcount()

	mp.fastrand[0] = 1597334677 * uint32(mp.id)
	mp.fastrand[1] = uint32(cputicks())
	if mp.fastrand[0]|mp.fastrand[1] == 0 {
		mp.fastrand[1] = 1
	}

	mpreinit(mp)
	if mp.gsignal != nil {
		mp.gsignal.stackguard1 = mp.gsignal.stack.lo + _StackGuard
	}

	// Add to allm so garbage collector doesn't free g->m
	// when it is just in a register or thread-local storage.
	mp.alllink = allm

	// NumCgoCall() iterates over allm w/o schedlock,
	// so we need to publish it safely.
	atomicstorep(unsafe.Pointer(&allm), unsafe.Pointer(mp))
	unlock(&sched.lock)

	// Allocate memory to hold a cgo traceback if the cgo call crashes.
	if iscgo || GOOS == "solaris" || GOOS == "illumos" || GOOS == "windows" {
		mp.cgoCallers = new(cgoCallers)
	}
}


func checkmcount() {
	// sched lock is held
	if mcount() > sched.maxmcount {
		print("runtime: program exceeds ", sched.maxmcount, "-thread limit\n")
		throw("thread exhaustion")
	}
}
```



### 8.3.2.获取闲置的M

回到函数`wakep->startn`，默认优先选用闲置M。

```go
// Try to get an m from midle list.
// Sched must be locked.
// May run during STW, so write barriers are not allowed.
//go:nowritebarrierrec
// 从空闲链表获取M
func mget() *m {
	mp := sched.midle.ptr()
	if mp != nil {
		sched.midle = mp.schedlink
		sched.nmidle--
	}
	return mp
}
```

被唤醒而进入工作状态的M，会陷入调度循环，从各种可能场所获取并执行G任务。只有当彻底找不到可执行任务，或因任务用时过长，系统调用阻塞等原因被剥夺P时，M才会进入休眠状态。

看下怎么让M进入休眠状态：

```go
// Stops execution of the current m until new work is available.
// Returns with acquired P.
func stopm() {
	_g_ := getg()

	if _g_.m.locks != 0 {
		throw("stopm holding locks")
	}
	if _g_.m.p != 0 {
		throw("stopm holding p")
	}
	if _g_.m.spinning {
		throw("stopm spinning")
	}

	lock(&sched.lock)
  // 放回闲置队列
	mput(_g_.m)
	unlock(&sched.lock)
  
  // 休眠，等待被唤醒
	notesleep(&_g_.m.park)
	noteclear(&_g_.m.park)
  
  // 绑定P
	acquirep(_g_.m.nextp.ptr())
	_g_.m.nextp = 0
}

// Put mp on midle list.
// Sched must be locked.
// May run during STW, so write barriers are not allowed.
//go:nowritebarrierre
// 将M放入闲置链表
func mput(mp *m) {
	mp.schedlink = sched.midle
	sched.midle.set(mp)
	sched.nmidle++
	checkdead()
}
```

我们允许进程里有成千上万的并发任务G，但最好不要有太多的M。且不说通过系统调用创建线程本身就有很大的性能损耗，大量闲置且不被回收的线程，M对象，g0栈空间都是资源浪费。好在这种情形极少出现，不过还是建议在生产部署前进行严格的测试。



## 8.4.执行

M执行G并发任务有两个起点：

+ 线程启动函数`mstart`；
+ `stopm`休眠唤醒后再度恢复调度循环。

先看`mstart`函数：

```go
// mstart is the entry-point for new Ms.
//
// This must not split the stack because we may not even have stack
// bounds set up yet.
//
// May run during STW (because it doesn't have a P yet), so write
// barriers are not allowed.
//
//go:nosplit
//go:nowritebarrierrec
func mstart() {
	_g_ := getg()

  // 确定栈边界
	osStack := _g_.stack.lo == 0
	if osStack {
		// Initialize stack bounds from system stack.
		// Cgo may have left stack size in stack.hi.
		// minit may update the stack bounds.
    // 对于无法使用g0 stack的系统，直接在系统堆栈上划出所需空间。
		size := _g_.stack.hi
		if size == 0 {
			size = 8192 * sys.StackGuardMultiplier
		}
    // 通过取size变量指针来确定高位地址
		_g_.stack.hi = uintptr(noescape(unsafe.Pointer(&size)))
		_g_.stack.lo = _g_.stack.hi - size + 1024
	}
	// Initialize stack guard so that we can start calling regular
	// Go code.
	_g_.stackguard0 = _g_.stack.lo + _StackGuard
	// This is the g0, so we can also call go:systemstack
	// functions, which check stackguard1.
	_g_.stackguard1 = _g_.stackguard0
	mstart1()

	// Exit this thread.
	if GOOS == "windows" || GOOS == "solaris" || GOOS == "illumos" || GOOS == "plan9" || GOOS == "darwin" || GOOS == "aix" {
		// Windows, Solaris, illumos, Darwin, AIX and Plan 9 always system-allocate
		// the stack, but put it in _g_.stack before mstart,
		// so the logic above hasn't set osStack yet.
		osStack = true
	}
	mexit(osStack)
}


func mstart1() {
	_g_ := getg()

	if _g_ != _g_.m.g0 {
		throw("bad runtime·mstart")
	}

	// Record the caller for use as the top of stack in mcall and
	// for terminating the thread.
	// We're never coming back to mstart1 after we call schedule,
	// so other calls can reuse the current frame.
  // 初始化g0执行现场
	save(getcallerpc(), getcallersp())
	asminit()
	minit()

	// Install signal handlers; after minit so that minit can
	// prepare the thread to be able to handle the signals.
	if _g_.m == &m0 {
		mstartm0()
	}

  // 执行启动函数
	if fn := _g_.m.mstartfn; fn != nil {
		fn()
	}

  // 绑定p
	if _g_.m != &m0 {
		acquirep(_g_.m.nextp.ptr())
		_g_.m.nextp = 0
	}
  // 进入任务调度循环（不再返回）
	schedule()
}
```

准备进入工作状态的M必须绑定一个有效P，nextp临时持有待绑定的P对象。因为在未正式执行前，并不适合直接设置相关属性。P为M提供cache，以便为工作线程提供对象内存分配。

看看绑定函数`acquirep(_g_.m.nextp.ptr())`的实现：

```go
// Associate p and the current m.
//
// This function is allowed to have write barriers even if the caller
// isn't because it immediately acquires _p_.
//
//go:yeswritebarrierrec
func acquirep(_p_ *p) {
	// Do the part that isn't allowed to have write barriers.
	wirep(_p_)

	// Have p; write barriers now allowed.

	// Perform deferred mcache flush before this P can allocate
	// from a potentially stale mcache.
	_p_.mcache.prepareForSweep()

	if trace.enabled {
		traceProcStart()
	}
}


// wirep is the first step of acquirep, which actually associates the
// current M to _p_. This is broken out so we can disallow write
// barriers for this part, since we don't yet have a P.
//
//go:nowritebarrierrec
//go:nosplit
func wirep(_p_ *p) {
  // 绑定 mcache
	_g_ := getg()

	if _g_.m.p != 0 || _g_.m.mcache != nil {
		throw("wirep: already in go")
	}
	if _p_.m != 0 || _p_.status != _Pidle {
		id := int64(0)
		if _p_.m != 0 {
			id = _p_.m.ptr().id
		}
		print("wirep: p->m=", _p_.m, "(", id, ") p->status=", _p_.status, "\n")
		throw("wirep: invalid p state")
	}
	_g_.m.mcache = _p_.mcache
	_g_.m.p.set(_p_)
	_p_.m.set(_g_.m)
	_p_.status = _Prunning
}


// prepareForSweep flushes c if the system has entered a new sweep phase
// since c was populated. This must happen between the sweep phase
// starting and the first allocation from c.
func (c *mcache) prepareForSweep() {
	// Alternatively, instead of making sure we do this on every P
	// between starting the world and allocating on that P, we
	// could leave allocate-black on, allow allocation to continue
	// as usual, use a ragged barrier at the beginning of sweep to
	// ensure all cached spans are swept, and then disable
	// allocate-black. However, with this approach it's difficult
	// to avoid spilling mark bits into the *next* GC cycle.
	sg := mheap_.sweepgen
	if c.flushGen == sg {
		return
	} else if c.flushGen != sg-2 {
		println("bad flushGen", c.flushGen, "in prepareForSweep; sweepgen", sg)
		throw("bad flushGen")
	}
	c.releaseAll()
	stackcache_clear(c)
	atomic.Store(&c.flushGen, mheap_.sweepgen) // Synchronizes with gcStart
}
```

上面的一切就绪后，M进入核心调度循环，这是一个由`schedule`，`execute`，`goroutine fn`， `goexit`函数构成的逻辑循环。就算M在休眠唤醒后，也只是从”断点“恢复。



### 8.4.1.schedule

```go
// One round of scheduler: find a runnable goroutine and execute it.
// Never returns.
func schedule() {
	_g_ := getg()

	if _g_.m.locks != 0 {
		throw("schedule: holding locks")
	}

  // 如果当前M是 lockedm,那么休眠
  // 没有立即execute(lockedg),是因为该lockedg此时可能被其他M获取
  // 兴许是中途用gosched暂时让出P，进入待运行队列
	if _g_.m.lockedg != 0 {
		stoplockedm()
		execute(_g_.m.lockedg.ptr(), false) // Never returns.
	}

	// We should not schedule away from a g that is executing a cgo call,
	// since the cgo call is using the m's g0 stack.
	if _g_.m.incgo {
		throw("schedule: in cgo")
	}

top:
  // 准备进入GC STW，休眠
	if sched.gcwaiting != 0 {
		gcstopm()
		goto top
	}
	if _g_.m.p.ptr().runSafePointFn != 0 {
		runSafePointFn()
	}

	var gp *g
  
  // 当从 P.next 提取G时，inheritTime = true
  // 不累加 P.schedtick 计数，使得它延长本地队列处理时间
	var inheritTime bool

	// Normal goroutines will check for need to wakeP in ready,
	// but GCworkers and tracereaders will not, so the check must
	// be done here instead.
	tryWakeP := false
	if trace.enabled || trace.shutdown {
		gp = traceReader()
		if gp != nil {
			casgstatus(gp, _Gwaiting, _Grunnable)
			traceGoUnpark(gp, 0)
			tryWakeP = true
		}
	}
  
  // 进入GC MarkWorker 工作模式
	if gp == nil && gcBlackenEnabled != 0 {
		gp = gcController.findRunnableGCWorker(_g_.m.p.ptr())
		tryWakeP = tryWakeP || gp != nil
	}
  
  // 每处理n个任务后就去全局队列获取G任务，以确保公平(当前值是61)
	if gp == nil {
		// Check the global runnable queue once in a while to ensure fairness.
		// Otherwise two goroutines can completely occupy the local runqueue
		// by constantly respawning each other.
		if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
			lock(&sched.lock)
			gp = globrunqget(_g_.m.p.ptr(), 1)
			unlock(&sched.lock)
		}
	}
  
  // 从P本地队列获取G任务
	if gp == nil {
		gp, inheritTime = runqget(_g_.m.p.ptr())
		if gp != nil && _g_.m.spinning {
			throw("schedule: spinning with local work")
		}
	}
  
  // 从其他可能的地方获取G任务
  // 如果获取失败，会让M进入休眠状态，被唤醒后重试
	if gp == nil {
		gp, inheritTime = findrunnable() // blocks until work is available
	}

	// This thread is going to run a goroutine and is not spinning anymore,
	// so if it was marked as spinning we need to reset it now and potentially
	// start a new spinning M.
	if _g_.m.spinning {
		resetspinning()
	}

	if sched.disable.user && !schedEnabled(gp) {
		// Scheduling of this goroutine is disabled. Put it on
		// the list of pending runnable goroutines for when we
		// re-enable user scheduling and look again.
		lock(&sched.lock)
		if schedEnabled(gp) {
			// Something re-enabled scheduling while we
			// were acquiring the lock.
			unlock(&sched.lock)
		} else {
			sched.disable.runnable.pushBack(gp)
			sched.disable.n++
			unlock(&sched.lock)
			goto top
		}
	}

	// If about to schedule a not-normal goroutine (a GCworker or tracereader),
	// wake a P if there is one.
	if tryWakeP {
		if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 {
			wakep()
		}
	}
  
  // 如果获取到的G是lockedg,那么将其连同P交给lockedm去执行
  // 休眠，等待唤醒后重新获取可用G
	if gp.lockedm != 0 {
		// Hands off own p to the locked m,
		// then blocks waiting for a new p.
		startlockedm(gp)
		goto top  // 进入等待STW，获取可用的G
	}

  // 执行 goroutine 任务函数
	execute(gp, inheritTime)
}


// Schedules the locked m to run the locked gp.
// May run during STW, so write barriers are not allowed.
//go:nowritebarrierrec
func startlockedm(gp *g) {
	_g_ := getg()

	mp := gp.lockedm.ptr()
	if mp == _g_.m {
		throw("startlockedm: locked to me")
	}
	if mp.nextp != 0 {
		throw("startlockedm: m has p")
	}
	// directly handoff current P to the locked m
	incidlelocked(-1)
  
  // 移交P，并唤醒 lockedm
	_p_ := releasep()
	mp.nextp.set(_p_)
	notewakeup(&mp.park)
  
  // 当前M休眠
	stopm()
}
```

调度函数获取可用的G后，交由`execute`去执行。同时，它还检查环境开关来决定是否参与垃圾回收。



### 8.4.2.execute

```go
// Schedules gp to run on the current M.
// If inheritTime is true, gp inherits the remaining time in the
// current time slice. Otherwise, it starts a new time slice.
// Never returns.
//
// Write barriers are allowed because this is called immediately after
// acquiring a P in several places.
//
//go:yeswritebarrierrec
func execute(gp *g, inheritTime bool) {
	_g_ := getg()

	casgstatus(gp, _Grunnable, _Grunning)
	gp.waitsince = 0
	gp.preempt = false
	gp.stackguard0 = gp.stack.lo + _StackGuard
	if !inheritTime {
		_g_.m.p.ptr().schedtick++
	}
	_g_.m.curg = gp
	gp.m = _g_.m

	// Check whether the profiler needs to be turned on or off.
	hz := sched.profilehz
	if _g_.m.profilehz != hz {
		setThreadCPUProfiler(hz)
	}

	if trace.enabled {
		// GoSysExit has to happen when we have a P, but before GoStart.
		// So we emit it here.
		if gp.syscallsp != 0 && gp.sysblocktraced {
			traceGoSysExit(gp.sysexitticks)
		}
		traceGoStart()
	}

  // 关键逻辑
	gogo(&gp.sched)
}
```

这里真正关键的就是汇编实现的`gogo`函数。它从g0栈切换到G栈，然后用一个`JMP`指令进入G任务函数的代码。

```bash
// func gogo(buf *gobuf)
// restore state from Gobuf; longjmp
TEXT runtime·gogo(SB), NOSPLIT, $16-8
	MOVQ	buf+0(FP), BX		// gobuf
	MOVQ	gobuf_g(BX), DX  // G
	MOVQ	0(DX), CX		// make sure g != nil
	get_tls(CX)
	MOVQ	DX, g(CX)   // g = G
	MOVQ	gobuf_sp(BX), SP	// 通过恢复SP寄存器值切换到G栈
	MOVQ	gobuf_ret(BX), AX
	MOVQ	gobuf_ctxt(BX), DX
	MOVQ	gobuf_bp(BX), BP
	MOVQ	$0, gobuf_sp(BX)	// clear to help garbage collector
	MOVQ	$0, gobuf_ret(BX)
	MOVQ	$0, gobuf_ctxt(BX)
	MOVQ	$0, gobuf_bp(BX)
	MOVQ	gobuf_pc(BX), BX  // 获取G任务函数地址
	JMP	BX                  // 执行
```

这里有个细节，JMP并不是CALL，也就是说不会将PC/IP入栈，那么执行完任务函数后，RET指令恢复的PC/IP值是什么？我们在`schedule`，`execute`里也没看到`goexit`调用，究竟如何再次进入调度循环呢？

其实在`newproc1`创建G任务时，我们忽略了一个细节。

```go
	// 此处保存的是 goexit 地址
	newg.sched.sp = sp
	newg.stktopsp = sp
	newg.sched.pc = funcPC(goexit) + sys.PCQuantum // +PCQuantum so that previous instruction is in same function
	newg.sched.g = guintptr(unsafe.Pointer(newg))
  // 此处的调用时关键所在
	gostartcallfn(&newg.sched, fn)
  
  // 初始化基本状态
	newg.gopc = callerpc
	newg.ancestors = saveAncestors(callergp)
	newg.startpc = fn.fn
```

在初始化`G.sched`时，pc保存的是`goexit`而非`fn`。关键秘密就是随后调用的`gostartcallfn`函数。

```go
// adjust Gobuf as if it executed a call to fn
// and then did an immediate gosave.
func gostartcallfn(gobuf *gobuf, fv *funcval) {
	var fn unsafe.Pointer
	if fv != nil {
		fn = unsafe.Pointer(fv.fn)
	} else {
		fn = unsafe.Pointer(funcPC(nilfunc))
	}
	gostartcall(gobuf, fn, unsafe.Pointer(fv))
}

// adjust Gobuf as if it executed a call to fn with context ctxt
// and then did an immediate gosave.
func gostartcall(buf *gobuf, fn, ctxt unsafe.Pointer) {
  // 调整sp
	sp := buf.sp
	if sys.RegSize > sys.PtrSize {
		sp -= sys.PtrSize
		*(*uintptr)(unsafe.Pointer(sp)) = 0
	}
	sp -= sys.PtrSize
  // 将buf.pc也就是goexit入栈
	*(*uintptr)(unsafe.Pointer(sp)) = buf.pc
  
  // 然后再次设置 sp 和 pc,此时pc才是G任务函数
	buf.sp = sp
	buf.pc = uintptr(fn)
	buf.ctxt = ctxt
}
```

很显然，在初始化完成后，G栈顶被压入了`goexit`地址。汇编函数`gogo JMP`跳转执行G任务，那么函数尾部的RET指令必然是将`goexit`地址恢复到 PC/IP，从而实现任务结束清理操作和再次进入调度循环。

看下`goexit`的实现：

```bash
// The top-most function running on a goroutine
// returns to goexit+PCQuantum.
TEXT runtime·goexit(SB),NOSPLIT,$0-0
	BYTE	$0x90	// NOP
	CALL	runtime·goexit1(SB)	// does not return
	// traceback from goexit1 must hit code range of goexit
	BYTE	$0x90	// NOP
```

`proc.go`：

```go
// Finishes execution of the current goroutine.
func goexit1() {
	if raceenabled {
		racegoend()
	}
	if trace.enabled {
		traceGoEnd()
	}
  // 切换到g0执行 goexit0
	mcall(goexit0)
}

// goexit continuation on g0.
func goexit0(gp *g) {
	_g_ := getg()

  // 清理G状态
	casgstatus(gp, _Grunning, _Gdead)
	if isSystemGoroutine(gp, false) {
		atomic.Xadd(&sched.ngsys, -1)
	}
	gp.m = nil
	locked := gp.lockedm != 0
	gp.lockedm = 0
	_g_.m.lockedg = 0
	gp.paniconfault = false
	gp._defer = nil // should be true already but just in case.
	gp._panic = nil // non-nil for Goexit during panic. points at stack-allocated data.
	gp.writebuf = nil
	gp.waitreason = 0
	gp.param = nil
	gp.labels = nil
	gp.timer = nil

	if gcBlackenEnabled != 0 && gp.gcAssistBytes > 0 {
		// Flush assist credit to the global pool. This gives
		// better information to pacing if the application is
		// rapidly creating an exiting goroutines.
		scanCredit := int64(gcController.assistWorkPerByte * float64(gp.gcAssistBytes))
		atomic.Xaddint64(&gcController.bgScanCredit, scanCredit)
		gp.gcAssistBytes = 0
	}

	// Note that gp's stack scan is now "valid" because it has no
	// stack.
	gp.gcscanvalid = true
	dropg()

	if GOARCH == "wasm" { // no threads yet on wasm
		gfput(_g_.m.p.ptr(), gp)
		schedule() // never returns
	}

	if _g_.m.lockedInt != 0 {
		print("invalid m->lockedInt = ", _g_.m.lockedInt, "\n")
		throw("internal lockOSThread error")
	}
  
  // 将G放回复用链表
	gfput(_g_.m.p.ptr(), gp)
	if locked {
		// The goroutine may have locked this thread because
		// it put it in an unusual kernel state. Kill it
		// rather than returning it to the thread pool.

		// Return to mstart, which will release the P and exit
		// the thread.
		if GOOS != "plan9" { // See golang.org/issue/22227.
			gogo(&_g_.m.g0.sched)
		} else {
			// Clear lockedExt on plan9 since we may end up re-using
			// this thread.
			_g_.m.lockedExt = 0
		}
	}
  
  // 重新进入调度循环
	schedule()
}
```

无论是`mcall`，`systemstack`，还是`gogo`都不会更新`g0.sched`栈线程。需要切换到g0栈时，直接从`g_sched+gobuf_sp`读取地址恢复SP。所以调用`goexit0/schedule`时，g0栈又从头开始，原调用堆栈全部失效，就算不返回也无所谓。

至此，单次任务完整结束，又回到查找待运行G任务的状态，循环往复。



### 8.4.3.找到可运行的G任务——findrunnable

回到`schedule`函数。

为了找到可以运行的G任务，`findrunnable`可谓费劲心机。本地队列，全局队列，网络任务(netpoll)，甚至是从其他P任务队列偷窃。所有的目的都是为了尽快完成所有任务，充分发挥多核并行能力。

```go
// Finds a runnable goroutine to execute.
// Tries to steal from other P's, get g from global queue, poll network.
func findrunnable() (gp *g, inheritTime bool) {
	_g_ := getg()

	// The conditions here and in handoffp must agree: if
	// findrunnable would return a G to run, handoffp must start
	// an M.

top:
	_p_ := _g_.m.p.ptr()
  // 垃圾回收
	if sched.gcwaiting != 0 {
		gcstopm()
		goto top
	}
	if _p_.runSafePointFn != 0 {
		runSafePointFn()
	}
  // fing是用来执行finalizer的goroutine
	if fingwait && fingwake {
		if gp := wakefing(); gp != nil {
			ready(gp, 0, true)
		}
	}
	if *cgo_yield != nil {
		asmcgocall(*cgo_yield, nil)
	}

	// local runq
  // 从本地队列获取
	if gp, inheritTime := runqget(_p_); gp != nil {
		return gp, inheritTime
	}

	// global runq
  // 从全局队列获取
	if sched.runqsize != 0 {
		lock(&sched.lock)
		gp := globrunqget(_p_, 0)
		unlock(&sched.lock)
		if gp != nil {
			return gp, false
		}
	}

	// Poll network.
	// This netpoll is only an optimization before we resort to stealing.
	// We can safely skip it if there are no waiters or a thread is blocked
	// in netpoll already. If there is any kind of logical race with that
	// blocked thread (e.g. it has already returned from netpoll, but does
	// not set lastpoll yet), this thread will do blocking netpoll below
	// anyway.
  // 检查netpoll任务
	if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Load64(&sched.lastpoll) != 0 {
		if list := netpoll(false); !list.empty() { // non-blocking
      // 返回多任务链表，将其他任务放入到全局队列
			gp := list.pop()
			injectglist(&list)
			casgstatus(gp, _Gwaiting, _Grunnable)
			if trace.enabled {
				traceGoUnpark(gp, 0)
			}
			return gp, false
		}
	}

	// Steal work from other P's.
	procs := uint32(gomaxprocs)
	if atomic.Load(&sched.npidle) == procs-1 {
		// Either GOMAXPROCS=1 or everybody, except for us, is idle already.
		// New work can appear from returning syscall/cgocall, network or timers.
		// Neither of that submits to local run queues, so no point in stealing.
		goto stop
	}
	// If number of spinning M's >= number of busy P's, block.
	// This is necessary to prevent excessive CPU consumption
	// when GOMAXPROCS>>1 but the program parallelism is low.
	if !_g_.m.spinning && 2*atomic.Load(&sched.nmspinning) >= procs-atomic.Load(&sched.npidle) {
		goto stop
	}
	if !_g_.m.spinning {
		_g_.m.spinning = true
		atomic.Xadd(&sched.nmspinning, 1)
	}
  // 随机挑一个P，偷些任务
	for i := 0; i < 4; i++ {
		for enum := stealOrder.start(fastrand()); !enum.done(); enum.next() {
			if sched.gcwaiting != 0 {
				goto top
			}
			stealRunNextG := i > 2 // first look for ready queues with more than 1 g
      // 如果尝试次数太多，连目标P.runnext都偷。
			if gp := runqsteal(_p_, allp[enum.position()], stealRunNextG); gp != nil {
				return gp, false
			}
		}
	}

stop:

	// We have nothing to do. If we're in the GC mark phase, can
	// safely scan and blacken objects, and have work to do, run
	// idle-time marking rather than give up the P.
  // 检查 GC MarkWorker
	if gcBlackenEnabled != 0 && _p_.gcBgMarkWorker != 0 && gcMarkWorkAvailable(_p_) {
		_p_.gcMarkWorkerMode = gcMarkWorkerIdleMode
		gp := _p_.gcBgMarkWorker.ptr()
		casgstatus(gp, _Gwaiting, _Grunnable)
		if trace.enabled {
			traceGoUnpark(gp, 0)
		}
		return gp, false
	}

	// wasm only:
	// If a callback returned and no other goroutine is awake,
	// then pause execution until a callback was triggered.
	if beforeIdle() {
		// At least one goroutine got woken.
		goto top
	}

	// Before we drop our P, make a snapshot of the allp slice,
	// which can change underfoot once we no longer block
	// safe-points. We don't need to snapshot the contents because
	// everything up to cap(allp) is immutable.
	allpSnapshot := allp

	// return P and block
	lock(&sched.lock)
  // 再次检查垃圾回收状态
	if sched.gcwaiting != 0 || _p_.runSafePointFn != 0 {
		unlock(&sched.lock)
		goto top
	}
  
  // 再次尝试全局队列
	if sched.runqsize != 0 {
		gp := globrunqget(_p_, 0)
		unlock(&sched.lock)
		return gp, false
	}
  
  // 释放当前P，取消自旋状态
	if releasep() != _p_ {
		throw("findrunnable: wrong p")
	}
	pidleput(_p_)
	unlock(&sched.lock)

	// Delicate dance: thread transitions from spinning to non-spinning state,
	// potentially concurrently with submission of new goroutines. We must
	// drop nmspinning first and then check all per-P queues again (with
	// #StoreLoad memory barrier in between). If we do it the other way around,
	// another thread can submit a goroutine after we've checked all run queues
	// but before we drop nmspinning; as the result nobody will unpark a thread
	// to run the goroutine.
	// If we discover new work below, we need to restore m.spinning as a signal
	// for resetspinning to unpark a new worker thread (because there can be more
	// than one starving goroutine). However, if after discovering new work
	// we also observe no idle Ps, it is OK to just park the current thread:
	// the system is fully loaded so no spinning threads are required.
	// Also see "Worker thread parking/unparking" comment at the top of the file.
	wasSpinning := _g_.m.spinning
	if _g_.m.spinning {
		_g_.m.spinning = false
		if int32(atomic.Xadd(&sched.nmspinning, -1)) < 0 {
			throw("findrunnable: negative nmspinning")
		}
	}

	// check all runqueues once again
  // 再次检查所有P任务队列
	for _, _p_ := range allpSnapshot {
		if !runqempty(_p_) {
			lock(&sched.lock)
      // 绑定一个空闲P，回到头部尝试偷取任务
			_p_ = pidleget()
			unlock(&sched.lock)
			if _p_ != nil {
				acquirep(_p_)
				if wasSpinning {
					_g_.m.spinning = true
					atomic.Xadd(&sched.nmspinning, 1)
				}
				goto top
			}
			break
		}
	}

	// Check for idle-priority GC work again.
	if gcBlackenEnabled != 0 && gcMarkWorkAvailable(nil) {
		lock(&sched.lock)
		_p_ = pidleget()
		if _p_ != nil && _p_.gcBgMarkWorker == 0 {
			pidleput(_p_)
			_p_ = nil
		}
		unlock(&sched.lock)
		if _p_ != nil {
			acquirep(_p_)
			if wasSpinning {
				_g_.m.spinning = true
				atomic.Xadd(&sched.nmspinning, 1)
			}
			// Go back to idle GC check.
			goto stop
		}
	}

	// poll network
  // 再次检查netpoll
	if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Xchg64(&sched.lastpoll, 0) != 0 {
		if _g_.m.p != 0 {
			throw("findrunnable: netpoll with p")
		}
		if _g_.m.spinning {
			throw("findrunnable: netpoll with spinning")
		}
		list := netpoll(true) // block until new work is available
		atomic.Store64(&sched.lastpoll, uint64(nanotime()))
		if !list.empty() {
			lock(&sched.lock)
			_p_ = pidleget()
			unlock(&sched.lock)
			if _p_ != nil {
				acquirep(_p_)
				gp := list.pop()
				injectglist(&list)
				casgstatus(gp, _Gwaiting, _Grunnable)
				if trace.enabled {
					traceGoUnpark(gp, 0)
				}
				return gp, false
			}
			injectglist(&list)
		}
	}
  // 一无所得，休眠
	stopm()
	goto top
}
```

我们顺着查找流程，依次查看不同优先级的获取方式。首先是本地队列，其中`P.runnext`优先级最高。详见`runqget`函数。

```go
// Get g from local runnable queue.
// If inheritTime is true, gp should inherit the remaining time in the
// current time slice. Otherwise, it should start a new time slice.
// Executed only by the owner P.
func runqget(_p_ *p) (gp *g, inheritTime bool) {
	// If there's a runnext, it's the next G to run.
  // 优先从 runnext 获取
  // 循环尝试cas。这里为什么用同步操作呢？因为可能有其他P从本地队列偷任务。
	for {
		next := _p_.runnext
		if next == 0 {
			break
		}
		if _p_.runnext.cas(next, 0) {
			return next.ptr(), true
		}
	}

  // 本地队列
	for {
		h := atomic.LoadAcq(&_p_.runqhead) // load-acquire, synchronize with other consumers
		t := _p_.runqtail
		if t == h {
			return nil, false
		}
    // 从头部提取
		gp := _p_.runq[h%uint32(len(_p_.runq))].ptr()
		if atomic.CasRel(&_p_.runqhead, h, h+1) { // cas-release, commits consume
			return gp, false
		}
	}
}
```

`runtext`不会影响`schedtick`计数，也就是说让`schedule`执行更多的任务才会去检查全局队列，所以才会有`inheritTime = true`的说法。

在检查全局队列时，除了返回一个可用G之外，还会批量转移一批到P本地队列里，毕竟不能每次加锁去操作全局队列，要通过加锁去做更多的事情。

检查全局队列的逻辑在`globrunqget`：

```go
// Try get a batch of G's from the global runnable queue.
// Sched must be locked.
func globrunqget(_p_ *p, max int32) *g {
	if sched.runqsize == 0 {
		return nil
	}

  // 将全局队列任务等分，计算最多能批量获取的任务数量
	n := sched.runqsize/gomaxprocs + 1
	if n > sched.runqsize {
		n = sched.runqsize
	}
	if max > 0 && n > max {
		n = max
	}
  
  // 不能超过 runq 数组长度的一半（128）
	if n > int32(len(_p_.runq))/2 {
		n = int32(len(_p_.runq)) / 2
	}

  // 调整计数
	sched.runqsize -= n

  // 返回第一个G任务，随后的才是要批量转移到本地的任务
	gp := sched.runq.pop()
	n--
  // 批量转移任务
	for ; n > 0; n-- {
		gp1 := sched.runq.pop()
		runqput(_p_, gp1, false)
	}
	return gp
}
```

只有当本地和全局队列都为空时，才会考虑去检查其他P任务队列。这个优先级最低，因为会影响目标P的执行（必须使用原子操作）。

最后，再看看”偷取“任务的函数：`runqsteal`

```go
// Steal half of elements from local runnable queue of p2
// and put onto local runnable queue of p.
// Returns one of the stolen elements (or nil if failed).
func runqsteal(_p_, p2 *p, stealRunNextG bool) *g {
	t := _p_.runqtail
  
  // 尝试从p2偷取一半任务存入p的本地队列
	n := runqgrab(p2, &_p_.runq, t, stealRunNextG)
	if n == 0 {
		return nil
	}
  
  // 返回尾部的G任务
	n--
	gp := _p_.runq[(t+n)%uint32(len(_p_.runq))].ptr()
	if n == 0 {
		return gp
	}
	h := atomic.LoadAcq(&_p_.runqhead) // load-acquire, synchronize with consumers
	if t-h+n >= uint32(len(_p_.runq)) {
		throw("runqsteal: runq overflow")
	}
  
  // 调整目标队列尾部状态
	atomic.StoreRel(&_p_.runqtail, t+n) // store-release, makes the item available for consumption
	return gp
}


// Grabs a batch of goroutines from _p_'s runnable queue into batch.
// Batch is a ring buffer starting at batchHead.
// Returns number of grabbed goroutines.
// Can be executed by any P.
func runqgrab(_p_ *p, batch *[256]guintptr, batchHead uint32, stealRunNextG bool) uint32 {
	for {
    // 计算批量转移任务数量
		h := atomic.LoadAcq(&_p_.runqhead) // load-acquire, synchronize with other consumers
		t := atomic.LoadAcq(&_p_.runqtail) // load-acquire, synchronize with the producer
		n := t - h
		n = n - n/2
    
    // 如果没有，那就尝试偷runnext吧
		if n == 0 {
			if stealRunNextG {
				// Try to steal from _p_.runnext.
				if next := _p_.runnext; next != 0 {
					if _p_.status == _Prunning {
						// Sleep to ensure that _p_ isn't about to run the g
						// we are about to steal.
						// The important use case here is when the g running
						// on _p_ ready()s another g and then almost
						// immediately blocks. Instead of stealing runnext
						// in this window, back off to give _p_ a chance to
						// schedule runnext. This will avoid thrashing gs
						// between different Ps.
						// A sync chan send/recv takes ~50ns as of time of
						// writing, so 3us gives ~50x overshoot.
						if GOOS != "windows" {
							usleep(3)
						} else {
							// On windows system timer granularity is
							// 1-15ms, which is way too much for this
							// optimization. So just yield.
							osyield()
						}
					}
					if !_p_.runnext.cas(next, 0) {
						continue
					}
					batch[batchHead%uint32(len(batch))] = next
					return 1
				}
			}
			return 0
		}
    
    // 数据异常，不可能超过一半值重试
		if n > uint32(len(_p_.runq)/2) { // read inconsistent h and t
			continue
		}
    
    // 转移任务
		for i := uint32(0); i < n; i++ {
			g := _p_.runq[(h+i)%uint32(len(_p_.runq))]
			batch[(batchHead+i)%uint32(len(batch))] = g
		}
    
    // 修改源P队列状态
    // 失败重试。因为没有修改源和目标队列位置状态，所以没有影响。
		if atomic.CasRel(&_p_.runqhead, h, h+n) { // cas-release, commits consume
			return n
		}
	}
}
```

这个就是著名的**Work-Stealing**算法。有空补充下。



### 8.4.4.lockedm和lockedg

当调度函数`schedule`检查到`locked`属性时，会适时移交，让正确的M去完成任务。

简单点说，就是`lockedm`会休眠，直到某人将`lockedg`交给它。而不幸拿到`lockedg`的M，则要将`lockedg`连同P一起传递给`lockedm`，还负责将其唤醒。至于它自己，则因失去P而被迫休眠，直到`wakep`带着新的P唤醒它。

详见上面的`schedule`中`stoplockedm`和`startlockedm`的调用方式。



## 8.5.连续栈

连续栈将调用堆栈(call stack)所有栈帧分配在一个连续内存空间。当空间不足时，另分配2x内存块，并拷贝当前栈全部数据，以避免分段栈链表结构在函数调用频繁时可能引发的切分热点问题。



结构示意图：

```bash
lo                   stackguard0                                                        hi
+--------------------+-------------------------------------------------------------------+
| StackGuard         |                                                                   |
+----------------------------------------------------------------------------------------+
                                                                                     <- SP
```

数据结构如下：

```go
// Stack describes a Go execution stack.
// The bounds of the stack are exactly [lo, hi),
// with no implicit data structures on either side.
type stack struct {
	lo uintptr
	hi uintptr
}

type g struct {
	// Stack parameters.
	// stack describes the actual stack memory: [stack.lo, stack.hi).
	// stackguard0 is the stack pointer compared in the Go stack growth prologue.
	// It is stack.lo+StackGuard normally, but can be StackPreempt to trigger a preemption.
	// stackguard1 is the stack pointer compared in the C stack growth prologue.
	// It is stack.lo+StackGuard on g0 and gsignal stacks.
	// It is ~0 on other goroutine stacks, to trigger a call to morestackc (and crash).
	stack       stack   // offset known to runtime/cgo
	stackguard0 uintptr // offset known to liblink
	stackguard1 uintptr // offset known to liblink

	_panic         *_panic // innermost panic - offset known to liblink
	_defer         *_defer // innermost defer
	m              *m      // current m; offset known to arm liblink
	sched          gobuf
	syscallsp      uintptr        // if status==Gsyscall, syscallsp = sched.sp to use during gc
	syscallpc      uintptr        // if status==Gsyscall, syscallpc = sched.pc to use during gc
	stktopsp       uintptr        // expected sp at top of stack, to check in traceback
	param          unsafe.Pointer // passed parameter on wakeup
	atomicstatus   uint32
	stackLock      uint32 // sigprof/scang lock; TODO: fold in to atomicstatus
	goid           int64
	schedlink      guintptr
	waitsince      int64      // approx time when the g become blocked
	waitreason     waitReason // if status==Gwaiting
	preempt        bool       // preemption signal, duplicates stackguard0 = stackpreempt
	paniconfault   bool       // panic (instead of crash) on unexpected fault address
	preemptscan    bool       // preempted g does scan for gc
	gcscandone     bool       // g has scanned stack; protected by _Gscan bit in status
	gcscanvalid    bool       // false at start of gc cycle, true if G has not run since last scan; TODO: remove?
	throwsplit     bool       // must not split stack
	raceignore     int8       // ignore race detection events
	sysblocktraced bool       // StartTrace has emitted EvGoInSyscall about this goroutine
	sysexitticks   int64      // cputicks when syscall has returned (for tracing)
	traceseq       uint64     // trace event sequencer
	tracelastp     puintptr   // last P emitted an event for this goroutine
	lockedm        muintptr
	sig            uint32
	writebuf       []byte
	sigcode0       uintptr
	sigcode1       uintptr
	sigpc          uintptr
	gopc           uintptr         // pc of go statement that created this goroutine
	ancestors      *[]ancestorInfo // ancestor information goroutine(s) that created this goroutine (only used if debug.tracebackancestors)
	startpc        uintptr         // pc of goroutine function
	racectx        uintptr
	waiting        *sudog         // sudog structures this g is waiting on (that have a valid elem ptr); in lock order
	cgoCtxt        []uintptr      // cgo traceback context
	labels         unsafe.Pointer // profiler labels
	timer          *timer         // cached timer for time.Sleep
	selectDone     uint32         // are we participating in a select and did someone win the race?

	// Per-G GC state

	// gcAssistBytes is this G's GC assist credit in terms of
	// bytes allocated. If this is positive, then the G has credit
	// to allocate gcAssistBytes bytes without assisting. If this
	// is negative, then the G must correct this by performing
	// scan work. We track this in bytes to make it fast to update
	// and check for debt in the malloc hot path. The assist ratio
	// determines how this corresponds to scan work debt.
	gcAssistBytes int64
}
```

其中，`stackguard0`是个非常重要的指针。在函数头部，编译器会插入一段指令，用它和SP寄存器进行比较，从而决定是否需要对栈空间扩容。另外，它还被用作抢占调度的标志。

栈。空间的初始分配发生在`newproc1`创建新G对象时。如下：(`newproc1`->`malg`)

```go
// 几个相关的常量
// 操作系统需要保留的区域，
_StackSystem = sys.GoosWindows*512*sys.PtrSize + sys.GoosPlan9*512 + sys.GoosDarwin*sys.GoarchArm*1024 + sys.GoosDarwin*sys.GoarchArm64*1024

// 默认栈大小
_StackMin = 2048

// StackGuard是一个警戒指针，用来判断栈容量是否需要扩张
_StackGuard = 880*sys.StackGuardMultiplier + _StackSystem

// Allocate a new g, with a stack big enough for stacksize bytes.
func malg(stacksize int32) *g {
	newg := new(g)
	if stacksize >= 0 {
		stacksize = round2(_StackSystem + stacksize)
		systemstack(func() {
			newg.stack = stackalloc(uint32(stacksize))
		})
		newg.stackguard0 = newg.stack.lo + _StackGuard
		newg.stackguard1 = ^uintptr(0)
	}
	return newg
}
```

在获取栈空间后，会立即设置`stackguard0`指针。



### 8.5.1.stackcache

因栈空间使用频繁，所以采取了和`cache/object`类似的做法，就是按大小分成几个等级进行缓存复用，当然也包括回收过多的闲置块。

以Linux为例，`_FixedStack`大小和`_StackMin`相同，`_NumStackOrders`等于4。

stack.go

```go
// The minimum stack size to allocate.
// The hackery here rounds FixedStack0 up to a power of 2.
_FixedStack0 = _StackMin + _StackSystem
_FixedStack1 = _FixedStack0 - 1
_FixedStack2 = _FixedStack1 | (_FixedStack1 >> 1)
_FixedStack3 = _FixedStack2 | (_FixedStack2 >> 2)
_FixedStack4 = _FixedStack3 | (_FixedStack3 >> 4)
_FixedStack5 = _FixedStack4 | (_FixedStack4 >> 8)
_FixedStack6 = _FixedStack5 | (_FixedStack5 >> 16)
_FixedStack  = _FixedStack6 + 1
```

malloc.go

```go
// Number of orders that get caching. Order 0 is FixedStack
// and each successive order is twice as large.
// We want to cache 2KB, 4KB, 8KB, and 16KB stacks. Larger stacks
// will be allocated directly.
// Since FixedStack is different on different systems, we
// must vary NumStackOrders to keep the same maximum cached size.
//   OS               | FixedStack | NumStackOrders
//   -----------------+------------+---------------
//   linux/darwin/bsd | 2KB        | 4
//   windows/32       | 4KB        | 3
//   windows/64       | 8KB        | 2
//   plan9            | 4KB        | 3
_NumStackOrders = 4 - sys.PtrSize/4*sys.GoosWindows - 1*sys.GoosPlan9
```

基于同样的性能考虑（无锁分配），栈空间被缓存在`Cache.stackcache`数组，且使用方法和`object`基本相同。

详见mcache.go

```go
// Per-thread (in Go, per-P) cache for small objects.
// No locking needed because it is per-thread (per-P).
//
// mcaches are allocated from non-GC'd memory, so any heap pointers
// must be specially handled.
//
//go:notinheap
type mcache struct {
	// The following members are accessed on every malloc,
	// so they are grouped here for better caching.
	next_sample uintptr // trigger heap sample after allocating this many bytes
	local_scan  uintptr // bytes of scannable heap allocated

	// Allocator cache for tiny objects w/o pointers.
	// See "Tiny allocator" comment in malloc.go.

	// tiny points to the beginning of the current tiny block, or
	// nil if there is no current tiny block.
	//
	// tiny is a heap pointer. Since mcache is in non-GC'd memory,
	// we handle it by clearing it in releaseAll during mark
	// termination.
	tiny             uintptr
	tinyoffset       uintptr
	local_tinyallocs uintptr // number of tiny allocs not counted in other stats

	// The rest is not accessed on every malloc.

	alloc [numSpanClasses]*mspan // spans to allocate from, indexed by spanClass

  // 关注这个
	stackcache [_NumStackOrders]stackfreelist

	// Local allocator stats, flushed during GC.
	local_largefree  uintptr                  // bytes freed for large objects (>maxsmallsize)
	local_nlargefree uintptr                  // number of frees for large objects (>maxsmallsize)
	local_nsmallfree [_NumSizeClasses]uintptr // number of frees for small objects (<=maxsmallsize)

	// flushGen indicates the sweepgen during which this mcache
	// was last flushed. If flushGen != mheap_.sweepgen, the spans
	// in this mcache are stale and need to the flushed so they
	// can be swept. This is done in acquirep.
	flushGen uint32
}

type stackfreelist struct {
	list gclinkptr // linked list of free stacks
	size uintptr   // total size of stacks in list
}
```

在获取栈空间时，会优先检查缓存链表。大空间直接从heap分配。

分配栈空间函数详见`stackalloc`：

```go
// stackalloc allocates an n byte stack.
//
// stackalloc must run on the system stack because it uses per-P
// resources and must not split the stack.
//
//go:systemstack
func stackalloc(n uint32) stack {
	// Stackalloc must be called on scheduler stack, so that we
	// never try to grow the stack during the code that stackalloc runs.
	// Doing so would cause a deadlock (issue 1547).
	thisg := getg()
	if thisg != thisg.m.g0 {
		throw("stackalloc not on scheduler stack")
	}
	if n&(n-1) != 0 {
		throw("stack size not a power of 2")
	}
	if stackDebug >= 1 {
		print("stackalloc ", n, "\n")
	}

	if debug.efence != 0 || stackFromSystem != 0 {
		n = uint32(round(uintptr(n), physPageSize))
		v := sysAlloc(uintptr(n), &memstats.stacks_sys)
		if v == nil {
			throw("out of memory (stackalloc)")
		}
		return stack{uintptr(v), uintptr(v) + uintptr(n)}
	}

	// Small stacks are allocated with a fixed-size free-list allocator.
	// If we need a stack of a bigger size, we fall back on allocating
	// a dedicated span.
	var v unsafe.Pointer
	if n < _FixedStack<<_NumStackOrders && n < _StackCacheSize {
    // 计算 order 等级
		order := uint8(0)
		n2 := n
		for n2 > _FixedStack {
			order++
			n2 >>= 1
		}
		var x gclinkptr
		c := thisg.m.mcache
    // 检查是否从缓存分配
		if stackNoCache != 0 || c == nil || thisg.m.preemptoff != "" {
			// c == nil can happen in the guts of exitsyscall or
			// procresize. Just get a stack from the global pool.
			// Also don't touch stackcache during gc
			// as it's flushed concurrently.
			lock(&stackpoolmu)
			x = stackpoolalloc(order)
			unlock(&stackpoolmu)
		} else {
      // 从对应链表提取复用空间
			x = c.stackcache[order].list
      // 提取失败，扩容后重试
			if x.ptr() == nil {
				stackcacherefill(c, order)
				x = c.stackcache[order].list
			}
      // 调整缓存链表
			c.stackcache[order].list = x.ptr().next
			c.stackcache[order].size -= uintptr(n)
		}
		v = unsafe.Pointer(x)
	} else {
		var s *mspan
		npage := uintptr(n) >> _PageShift
		log2npage := stacklog2(npage)

		// Try to get a stack from the large stack cache.
		lock(&stackLarge.lock)
		if !stackLarge.free[log2npage].isEmpty() {
			s = stackLarge.free[log2npage].first
			stackLarge.free[log2npage].remove(s)
		}
		unlock(&stackLarge.lock)

		if s == nil {
			// Allocate a new stack from the heap.
      // 大空间直接从heap分配
			s = mheap_.allocManual(npage, &memstats.stacks_inuse)
			if s == nil {
				throw("out of memory")
			}
			osStackAlloc(s)
			s.elemsize = uintptr(n)
		}
		v = unsafe.Pointer(s.base())
	}

	if raceenabled {
		racemalloc(v, uintptr(n))
	}
	if msanenabled {
		msanmalloc(v, uintptr(n))
	}
	if stackDebug >= 1 {
		print("  allocated ", v, "\n")
	}
	return stack{uintptr(v), uintptr(v) + uintptr(n)}
}
```

类似前文的内存分配器，然后我们再看看扩容过程怎么做，详见`stackcacherefill`：

```go
// stackcacherefill/stackcacherelease implement a global pool of stack segments.
// The pool is required to prevent unlimited growth of per-thread caches.
//
//go:systemstack
func stackcacherefill(c *mcache, order uint8) {
	if stackDebug >= 1 {
		print("stackcacherefill order=", order, "\n")
	}

	// Grab some stacks from the global cache.
	// Grab half of the allowed capacity (to prevent thrashing).
	var list gclinkptr
	var size uintptr
	lock(&stackpoolmu)
  // 提取一批复用空间
	for size < _StackCacheSize/2 {
    // 每次提取一个
		x := stackpoolalloc(order)
		x.ptr().next = list
		list = x
		size += _FixedStack << order
	}
	unlock(&stackpoolmu)
  
  // 保存到 cache.stackcache数组
	c.stackcache[order].list = list
	c.stackcache[order].size = size
}
```

然后，有个全局缓存`stackpool`似乎在充当`central`的角色。

```go
var stackpool [_NumStackOrders]mSpanList

// Allocates a stack from the free pool. Must be called with
// stackpoolmu held.
func stackpoolalloc(order uint8) gclinkptr {
  // 尝试从全局缓存获取
	list := &stackpool[order]
	s := list.first
  
  // 重新从heap获取span切分
	if s == nil {
		// no free stacks. Allocate another span worth.
		s = mheap_.allocManual(_StackCacheSize>>_PageShift, &memstats.stacks_inuse)
		if s == nil {
			throw("out of memory")
		}
		if s.allocCount != 0 {
			throw("bad allocCount")
		}
		if s.manualFreeList.ptr() != nil {
			throw("bad manualFreeList")
		}
		osStackAlloc(s)
		s.elemsize = _FixedStack << order
		for i := uintptr(0); i < _StackCacheSize; i += s.elemsize {
			x := gclinkptr(s.base() + i)
			x.ptr().next = s.manualFreeList
			s.manualFreeList = x
		}
		list.insert(s)
	}
  
  // 从链表返回一个空间
	x := s.manualFreeList
	if x.ptr() == nil {
		throw("span has no free stacks")
	}
	s.manualFreeList = x.ptr().next
	s.allocCount++
  
  // 如果当前链表已空，则移除span
	if s.manualFreeList.ptr() == nil {
		// all stacks in s are allocated.
		list.remove(s)
	}
	return x
}
```

堆空间分配算法，`allocManual`如下：

```go
func (h *mheap) allocManual(npage uintptr, stat *uint64) *mspan {
	lock(&h.lock)
	s := h.allocSpanLocked(npage, stat)
	if s != nil {
		s.state = mSpanManual
		s.manualFreeList = 0
		s.allocCount = 0
		s.spanclass = 0
		s.nelems = 0
		s.elemsize = 0
		s.limit = s.base() + s.npages<<_PageShift
		// Manually managed memory doesn't count toward heap_sys.
		memstats.heap_sys -= uint64(s.npages << _PageShift)
	}

	// This unlock acts as a release barrier. See mheap.alloc_m.
	unlock(&h.lock)

	return s
}
```

总结：栈内存也从arena区域分配，使用和对象分配相同的策略和算法，独立开两份代码。



### 8.5.2.morestack

执行函数前，需要为其准备好所需栈帧空间，此时是检查连续栈是否需要扩容的最佳时机。为此，编译器会在函数头部插入几条特殊指令，通过比较`stackguard0`和`SP`来决定是否进行扩容操作。

我们可以试个例子：

```go
package main

import (
    "fmt"
)

func test() {
    fmt.Println("hello")
}

func main() {
    test()
}
```

然后禁用内联进行编译，再反汇编。

```bash
[sysublackbear@centos-linux zhuowen.deng]$ go build -gcflags "-l" -o test test.go
[sysublackbear@centos-linux zhuowen.deng]$ go tool objdump -s "main\.test" test
TEXT main.test(SB) /home/sysublackbear/go/src/zhuowen.deng/test.go
  test.go:7		0x482070		64488b0c25f8ffffff	MOVQ FS:0xfffffff8, CX  // 当前G
  test.go:7		0x482079		483b6110		CMPQ 0x10(CX), SP  // G+0x10 指向 g.stackguard0，和SP比较
  test.go:7		0x48207d		7658			JBE 0x4820d7  // 如果SP <= stackguard0，则跳转值 0x4820d7
  test.go:7		0x48207f		4883ec48		SUBQ $0x48, SP
  test.go:7		0x482083		48896c2440		MOVQ BP, 0x40(SP)
  test.go:7		0x482088		488d6c2440		LEAQ 0x40(SP), BP
  test.go:8		0x48208d		0f57c0			XORPS X0, X0
  test.go:8		0x482090		0f11442430		MOVUPS X0, 0x30(SP)
  test.go:8		0x482095		488d05a4080100		LEAQ 0x108a4(IP), AX
  test.go:8		0x48209c		4889442430		MOVQ AX, 0x30(SP)
  test.go:8		0x4820a1		488d0528120400		LEAQ main.statictmp_0(SB), AX
  test.go:8		0x4820a8		4889442438		MOVQ AX, 0x38(SP)
  test.go:8		0x4820ad		488d442430		LEAQ 0x30(SP), AX
  test.go:8		0x4820b2		48890424		MOVQ AX, 0(SP)
  test.go:8		0x4820b6		48c744240801000000	MOVQ $0x1, 0x8(SP)
  test.go:8		0x4820bf		48c744241001000000	MOVQ $0x1, 0x10(SP)
  test.go:8		0x4820c8		e8e39dffff		CALL fmt.Println(SB)
  test.go:9		0x4820cd		488b6c2440		MOVQ 0x40(SP), BP
  test.go:9		0x4820d2		4883c448		ADDQ $0x48, SP
  test.go:9		0x4820d6		c3			RET
  test.go:7		0x4820d7		e894a1fcff		CALL runtime.morestack_noctxt(SB)  // 跳转到这里，执行morestack扩容
  test.go:7		0x4820dc		eb92			JMP main.test(SB)  // 扩容结束后，重新执行当前函数
```

这几条指令很简答。如果SP指针地址小于`stackguard0`（栈从高位地址向低位地址分配），那么显然已经溢出，这就需要扩容，否则当前和后续函数就无从分配栈帧内存。

细心一点，你会发现CMP指令并没将当前栈帧所需空间算上。假如SP大于`stackguard0`，但与其差值又小于当前栈帧大小呢？这显然不会跳转执行扩容操作，但又不能满足当前函数需求，难道只能眼看着堆栈溢出？

我们前面知道，在`stack.lo`和`stackguard0`之间尚有部分保留空间，所以适当”溢出“是允许的。

从stack.go里面的`_StackSmall = 128`我们可以知道，这段保留空间为128Bytes。

修改下测试代码，看看效果。

```go
package main

func test1() {
    var x [128]byte
    x[1] = 1
}

func test2() {
    var x [129]byte
    x[1] = 1
}

func main() {
    test1()
    test2()
}
```

```bash
[sysublackbear@centos-linux zhuowen.deng]$ go build -gcflags "-l" -o test test.go
[sysublackbear@centos-linux zhuowen.deng]$ go tool objdump -s "main\.test" test
```

很显然，如果当前栈帧是`SmallStack(0x80)`，那么就允许在 `[lo, stackguard0]`之间分配。



对栈进行扩容并不是件容易的事情，其中涉及很多内容。不过，在这里我们只须了解其基本过程和算法意图，无须深入到所有细节。

asm_amd64.s

```bash
// morestack but not preserving ctxt.
TEXT runtime·morestack_noctxt(SB),NOSPLIT,$0
	MOVL	$0, DX
	JMP	runtime·morestack(SB)

// Called during function prolog when more stack is needed.
//
// The traceback routines see morestack on a g0 as being
// the top of a stack (for example, morestack calling newstack
// calling the scheduler calling newm calling gc), so we must
// record an argument size. For that purpose, it has no arguments.
TEXT runtime·morestack(SB),NOSPLIT,$0-0
	// Cannot grow scheduler stack (m->g0).
	get_tls(CX)
	MOVQ	g(CX), BX
	MOVQ	g_m(BX), BX
	MOVQ	m_g0(BX), SI
	CMPQ	g(CX), SI
	JNE	3(PC)
	CALL	runtime·badmorestackg0(SB)
	CALL	runtime·abort(SB)

	// Cannot grow signal stack (m->gsignal).
	MOVQ	m_gsignal(BX), SI
	CMPQ	g(CX), SI
	JNE	3(PC)
	CALL	runtime·badmorestackgsignal(SB)
	CALL	runtime·abort(SB)

	// Called from f.
	// Set m->morebuf to f's caller.
	NOP	SP	// tell vet SP changed - stop checking offsets
	MOVQ	8(SP), AX	// f's caller's PC
	MOVQ	AX, (m_morebuf+gobuf_pc)(BX)
	LEAQ	16(SP), AX	// f's caller's SP
	MOVQ	AX, (m_morebuf+gobuf_sp)(BX)
	get_tls(CX)
	MOVQ	g(CX), SI
	MOVQ	SI, (m_morebuf+gobuf_g)(BX)

	// Set g->sched to context in f.
	MOVQ	0(SP), AX // f's PC
	MOVQ	AX, (g_sched+gobuf_pc)(SI)
	MOVQ	SI, (g_sched+gobuf_g)(SI)
	LEAQ	8(SP), AX // f's SP
	MOVQ	AX, (g_sched+gobuf_sp)(SI)
	MOVQ	BP, (g_sched+gobuf_bp)(SI)
	MOVQ	DX, (g_sched+gobuf_ctxt)(SI)

	// Call newstack on m->g0's stack.
	MOVQ	m_g0(BX), BX
	MOVQ	BX, g(CX)
	MOVQ	(g_sched+gobuf_sp)(BX), SP
	CALL	runtime·newstack(SB)
	CALL	runtime·abort(SB)	// crash if newstack returns
	RET
```

上面的代码有点复杂，基本过程就是分配一个2x大小的新栈，然后将数据拷贝过去，替换掉旧栈。当然，这期间需要对指针等内容做些调整。



再看`newstack`的实现：

```go
func newstack() {
	thisg := getg()
	// TODO: double check all gp. shouldn't be getg().
	if thisg.m.morebuf.g.ptr().stackguard0 == stackFork {
		throw("stack growth after fork")
	}
	if thisg.m.morebuf.g.ptr() != thisg.m.curg {
		print("runtime: newstack called from g=", hex(thisg.m.morebuf.g), "\n"+"\tm=", thisg.m, " m->curg=", thisg.m.curg, " m->g0=", thisg.m.g0, " m->gsignal=", thisg.m.gsignal, "\n")
		morebuf := thisg.m.morebuf
		traceback(morebuf.pc, morebuf.sp, morebuf.lr, morebuf.g.ptr())
		throw("runtime: wrong goroutine in newstack")
	}

	gp := thisg.m.curg

	if thisg.m.curg.throwsplit {
		// Update syscallsp, syscallpc in case traceback uses them.
		morebuf := thisg.m.morebuf
		gp.syscallsp = morebuf.sp
		gp.syscallpc = morebuf.pc
		pcname, pcoff := "(unknown)", uintptr(0)
		f := findfunc(gp.sched.pc)
		if f.valid() {
			pcname = funcname(f)
			pcoff = gp.sched.pc - f.entry
		}
		print("runtime: newstack at ", pcname, "+", hex(pcoff),
			" sp=", hex(gp.sched.sp), " stack=[", hex(gp.stack.lo), ", ", hex(gp.stack.hi), "]\n",
			"\tmorebuf={pc:", hex(morebuf.pc), " sp:", hex(morebuf.sp), " lr:", hex(morebuf.lr), "}\n",
			"\tsched={pc:", hex(gp.sched.pc), " sp:", hex(gp.sched.sp), " lr:", hex(gp.sched.lr), " ctxt:", gp.sched.ctxt, "}\n")

		thisg.m.traceback = 2 // Include runtime frames
		traceback(morebuf.pc, morebuf.sp, morebuf.lr, gp)
		throw("runtime: stack split at bad time")
	}

	morebuf := thisg.m.morebuf
	thisg.m.morebuf.pc = 0
	thisg.m.morebuf.lr = 0
	thisg.m.morebuf.sp = 0
	thisg.m.morebuf.g = 0

	// NOTE: stackguard0 may change underfoot, if another thread
	// is about to try to preempt gp. Read it just once and use that same
	// value now and below.
	preempt := atomic.Loaduintptr(&gp.stackguard0) == stackPreempt

	// Be conservative about where we preempt.
	// We are interested in preempting user Go code, not runtime code.
	// If we're holding locks, mallocing, or preemption is disabled, don't
	// preempt.
	// This check is very early in newstack so that even the status change
	// from Grunning to Gwaiting and back doesn't happen in this case.
	// That status change by itself can be viewed as a small preemption,
	// because the GC might change Gwaiting to Gscanwaiting, and then
	// this goroutine has to wait for the GC to finish before continuing.
	// If the GC is in some way dependent on this goroutine (for example,
	// it needs a lock held by the goroutine), that small preemption turns
	// into a real deadlock.
	if preempt {
		if thisg.m.locks != 0 || thisg.m.mallocing != 0 || thisg.m.preemptoff != "" || thisg.m.p.ptr().status != _Prunning {
			// Let the goroutine keep running for now.
			// gp->preempt is set, so it will be preempted next time.
			gp.stackguard0 = gp.stack.lo + _StackGuard
			gogo(&gp.sched) // never return
		}
	}

	if gp.stack.lo == 0 {
		throw("missing stack in newstack")
	}
	sp := gp.sched.sp
	if sys.ArchFamily == sys.AMD64 || sys.ArchFamily == sys.I386 || sys.ArchFamily == sys.WASM {
		// The call to morestack cost a word.
		sp -= sys.PtrSize
	}
	if stackDebug >= 1 || sp < gp.stack.lo {
		print("runtime: newstack sp=", hex(sp), " stack=[", hex(gp.stack.lo), ", ", hex(gp.stack.hi), "]\n",
			"\tmorebuf={pc:", hex(morebuf.pc), " sp:", hex(morebuf.sp), " lr:", hex(morebuf.lr), "}\n",
			"\tsched={pc:", hex(gp.sched.pc), " sp:", hex(gp.sched.sp), " lr:", hex(gp.sched.lr), " ctxt:", gp.sched.ctxt, "}\n")
	}
	if sp < gp.stack.lo {
		print("runtime: gp=", gp, ", goid=", gp.goid, ", gp->status=", hex(readgstatus(gp)), "\n ")
		print("runtime: split stack overflow: ", hex(sp), " < ", hex(gp.stack.lo), "\n")
		throw("runtime: split stack overflow")
	}

	if preempt {
		if gp == thisg.m.g0 {
			throw("runtime: preempt g0")
		}
		if thisg.m.p == 0 && thisg.m.locks == 0 {
			throw("runtime: g is running but p is not")
		}
		// Synchronize with scang.
		casgstatus(gp, _Grunning, _Gwaiting)
		if gp.preemptscan {
			for !castogscanstatus(gp, _Gwaiting, _Gscanwaiting) {
				// Likely to be racing with the GC as
				// it sees a _Gwaiting and does the
				// stack scan. If so, gcworkdone will
				// be set and gcphasework will simply
				// return.
			}
			if !gp.gcscandone {
				// gcw is safe because we're on the
				// system stack.
				gcw := &gp.m.p.ptr().gcw
				scanstack(gp, gcw)
				gp.gcscandone = true
			}
			gp.preemptscan = false
			gp.preempt = false
			casfrom_Gscanstatus(gp, _Gscanwaiting, _Gwaiting)
			// This clears gcscanvalid.
			casgstatus(gp, _Gwaiting, _Grunning)
			gp.stackguard0 = gp.stack.lo + _StackGuard
			gogo(&gp.sched) // never return
		}

		// Act like goroutine called runtime.Gosched.
		casgstatus(gp, _Gwaiting, _Grunning)
		gopreempt_m(gp) // never return
	}

	// Allocate a bigger segment and move the stack.
  // 扩张2倍
	oldsize := gp.stack.hi - gp.stack.lo
	newsize := oldsize * 2
	if newsize > maxstacksize {
		print("runtime: goroutine stack exceeds ", maxstacksize, "-byte limit\n")
		throw("stack overflow")
	}

	// The goroutine must be executing in order to call newstack,
	// so it must be Grunning (or Gscanrunning).
	casgstatus(gp, _Grunning, _Gcopystack)

	// The concurrent GC will not scan the stack while we are doing the copy since
	// the gp is in a Gcopystack status.
  // 拷贝栈数据后切换到新栈
	copystack(gp, newsize, true)
	if stackDebug >= 1 {
		print("stack grow done\n")
	}
  // 恢复执行
	casgstatus(gp, _Gcopystack, _Grunning)
	gogo(&gp.sched)
}

func copystack(gp *g, newsize uintptr, sync bool) {
	if gp.syscallsp != 0 {
		throw("stack growth not allowed in system call")
	}
	old := gp.stack
	if old.lo == 0 {
		throw("nil stackbase")
	}
	used := old.hi - gp.sched.sp

	// allocate new stack
  // 从缓存或堆分配新栈空间
	new := stackalloc(uint32(newsize))
  
  // 清零
	if stackPoisonCopy != 0 {
		fillstack(new, 0xfd)
	}
	if stackDebug >= 1 {
		print("copystack gp=", gp, " [", hex(old.lo), " ", hex(old.hi-used), " ", hex(old.hi), "]", " -> [", hex(new.lo), " ", hex(new.hi-used), " ", hex(new.hi), "]/", newsize, "\n")
	}

	// Compute adjustment.
  // 调整指针等操作 ... 
	var adjinfo adjustinfo
	adjinfo.old = old
	adjinfo.delta = new.hi - old.hi

	// Adjust sudogs, synchronizing with channel ops if necessary.
	ncopy := used
	if sync {
		adjustsudogs(gp, &adjinfo)
	} else {
		// sudogs can point in to the stack. During concurrent
		// shrinking, these areas may be written to. Find the
		// highest such pointer so we can handle everything
		// there and below carefully. (This shouldn't be far
		// from the bottom of the stack, so there's little
		// cost in handling everything below it carefully.)
		adjinfo.sghi = findsghi(gp, old)

		// Synchronize with channel ops and copy the part of
		// the stack they may interact with.
		ncopy -= syncadjustsudogs(gp, used, &adjinfo)
	}

	// Copy the stack (or the rest of it) to the new location
  // 拷贝数据到新栈空间
	memmove(unsafe.Pointer(new.hi-ncopy), unsafe.Pointer(old.hi-ncopy), ncopy)

	// Adjust remaining structures that have pointers into stacks.
	// We have to do most of these before we traceback the new
	// stack because gentraceback uses them.
	adjustctxt(gp, &adjinfo)
	adjustdefers(gp, &adjinfo)
	adjustpanics(gp, &adjinfo)
	if adjinfo.sghi != 0 {
		adjinfo.sghi += adjinfo.delta
	}

	// Swap out old stack for new one
  // 切换到新栈
	gp.stack = new
	gp.stackguard0 = new.lo + _StackGuard // NOTE: might clobber a preempt request
	gp.sched.sp = new.hi - used
	gp.stktopsp += adjinfo.delta

	// Adjust pointers in the new stack.
	gentraceback(^uintptr(0), ^uintptr(0), 0, gp, 0, nil, 0x7fffffff, adjustframe, noescape(unsafe.Pointer(&adjinfo)), 0)

	// free old stack
  // 将旧栈清零后释放
	if stackPoisonCopy != 0 {
		fillstack(old, 0xfc)
	}
	stackfree(old)
}
```





### 8.5.3.stackfree

释放栈空间的操作，依旧和回收object类似。

详见stack.go：

```go
// stackfree frees an n byte stack allocation at stk.
//
// stackfree must run on the system stack because it uses per-P
// resources and must not split the stack.
//
//go:systemstack
func stackfree(stk stack) {
	gp := getg()
	v := unsafe.Pointer(stk.lo)
	n := stk.hi - stk.lo
	if n&(n-1) != 0 {
		throw("stack not a power of 2")
	}
	if stk.lo+n < stk.hi {
		throw("bad stack size")
	}
	if stackDebug >= 1 {
		println("stackfree", v, n)
		memclrNoHeapPointers(v, n) // for testing, clobber stack data
	}
	if debug.efence != 0 || stackFromSystem != 0 {
		if debug.efence != 0 || stackFaultOnFree != 0 {
			sysFault(v, n)
		} else {
			sysFree(v, n, &memstats.stacks_sys)
		}
		return
	}
	if msanenabled {
		msanfree(v, n)
	}
  
  // 放回缓存链表
	if n < _FixedStack<<_NumStackOrders && n < _StackCacheSize {
    // 计算order等级
		order := uint8(0)
		n2 := n
		for n2 > _FixedStack {
			order++
			n2 >>= 1
		}
		x := gclinkptr(v)
		c := gp.m.mcache
		if stackNoCache != 0 || c == nil || gp.m.preemptoff != "" {
			lock(&stackpoolmu)
			stackpoolfree(x, order)
			unlock(&stackpoolmu)
		} else {
      // 如果缓存大小超出限制，则释放一些
			if c.stackcache[order].size >= _StackCacheSize {
				stackcacherelease(c, order)
			}
      // 放回缓存链表
			x.ptr().next = c.stackcache[order].list
			c.stackcache[order].list = x
			c.stackcache[order].size += n
		}
	} else {
		s := spanOfUnchecked(uintptr(v))
		if s.state != mSpanManual {
			println(hex(s.base()), v)
			throw("bad span state")
		}
		if gcphase == _GCoff {
			// Free the stack immediately if we're
			// sweeping.
			osStackFree(s)
      // 归还给heap
			mheap_.freeManual(s, &memstats.stacks_inuse)
		} else {
			// If the GC is running, we can't return a
			// stack span to the heap because it could be
			// reused as a heap span, and this state
			// change would race with GC. Add it to the
			// large stack cache instead.
			log2npage := stacklog2(s.npages)
			lock(&stackLarge.lock)
      // 如果正在垃圾回收期间，那么放到一个待处理队列，由垃圾回收器处理
			stackLarge.free[log2npage].insert(s)
			unlock(&stackLarge.lock)
		}
	}
}
```

回收的栈空间被放回对应的复用链表。如缓存过多，则转移一批到全局链表，或直接将自由的span归还给heap。

详见`stackcacherelease`：

```go
//go:systemstack
func stackcacherelease(c *mcache, order uint8) {
	if stackDebug >= 1 {
		print("stackcacherelease order=", order, "\n")
	}
	x := c.stackcache[order].list
	size := c.stackcache[order].size
	lock(&stackpoolmu)
  
  // 如果当前链表过大，则释放一半
	for size > _StackCacheSize/2 {
		y := x.ptr().next
    // 每次释放一个，它们可能属于不同的span
		stackpoolfree(x, order)
		x = y
		size -= _FixedStack << order
	}
	unlock(&stackpoolmu)
	c.stackcache[order].list = x
	c.stackcache[order].size = size
}


// Adds stack x to the free pool. Must be called with stackpoolmu held.
func stackpoolfree(x gclinkptr, order uint8) {
  // 找到所属span
	s := spanOfUnchecked(uintptr(x))
	if s.state != mSpanManual {
		throw("freeing stack not in a stack span")
	}
	if s.manualFreeList.ptr() == nil {
		// s will now have a free stack
		stackpool[order].insert(s)
	}
  // 添加到span.freelist
	x.ptr().next = s.manualFreeList
	s.manualFreeList = x
	s.allocCount--
  
  // 如果该span已收回全部空间，那么将其归还给heap
	if gcphase == _GCoff && s.allocCount == 0 {
		// Span is completely free. Return it to the heap
		// immediately if we're sweeping.
		//
		// If GC is active, we delay the free until the end of
		// GC to avoid the following type of situation:
		//
		// 1) GC starts, scans a SudoG but does not yet mark the SudoG.elem pointer
		// 2) The stack that pointer points to is copied
		// 3) The old stack is freed
		// 4) The containing span is marked free
		// 5) GC attempts to mark the SudoG.elem pointer. The
		//    marking fails because the pointer looks like a
		//    pointer into a free span.
		//
		// By not freeing, we prevent step #4 until GC is done.
		stackpool[order].remove(s)
		s.manualFreeList = 0
		osStackFree(s)
		mheap_.freeManual(s, &memstats.stacks_inuse)
	}
}
```

除了`morestack`调用导致`stackfree`操作外，垃圾回收对栈空间的处理也会发生此操作。

mgcmark.go

```go
func markroot(gcw *gcWork, i uint32) {
	// TODO(austin): This is a bit ridiculous. Compute and store
	// the bases in gcMarkRootPrepare instead of the counts.
	baseFlushCache := uint32(fixedRootCount)
	baseData := baseFlushCache + uint32(work.nFlushCacheRoots)
	baseBSS := baseData + uint32(work.nDataRoots)
	baseSpans := baseBSS + uint32(work.nBSSRoots)
	baseStacks := baseSpans + uint32(work.nSpanRoots)
	end := baseStacks + uint32(work.nStackRoots)

	// Note: if you add a case here, please also update heapdump.go:dumproots.
	switch {
	case baseFlushCache <= i && i < baseData:
		flushmcache(int(i - baseFlushCache))

	case baseData <= i && i < baseBSS:
		for _, datap := range activeModules() {
			markrootBlock(datap.data, datap.edata-datap.data, datap.gcdatamask.bytedata, gcw, int(i-baseData))
		}

	case baseBSS <= i && i < baseSpans:
		for _, datap := range activeModules() {
			markrootBlock(datap.bss, datap.ebss-datap.bss, datap.gcbssmask.bytedata, gcw, int(i-baseBSS))
		}

	case i == fixedRootFinalizers:
		for fb := allfin; fb != nil; fb = fb.alllink {
			cnt := uintptr(atomic.Load(&fb.cnt))
			scanblock(uintptr(unsafe.Pointer(&fb.fin[0])), cnt*unsafe.Sizeof(fb.fin[0]), &finptrmask[0], gcw, nil)
		}

	case i == fixedRootFreeGStacks:
		// Switch to the system stack so we can call
		// stackfree.
		systemstack(markrootFreeGStacks)

	case baseSpans <= i && i < baseStacks:
		// mark mspan.specials
		markrootSpans(gcw, int(i-baseSpans))

	default:
		// the rest is scanning goroutine stacks
		var gp *g
		if baseStacks <= i && i < end {
			gp = allgs[i-baseStacks]
		} else {
			throw("markroot: bad index")
		}

		// remember when we've first observed the G blocked
		// needed only to output in traceback
		status := readgstatus(gp) // We are not in a scan state
		if (status == _Gwaiting || status == _Gsyscall) && gp.waitsince == 0 {
			gp.waitsince = work.tstart
		}

		// scang must be done on the system stack in case
		// we're trying to scan our own stack.
		systemstack(func() {
			// If this is a self-scan, put the user G in
			// _Gwaiting to prevent self-deadlock. It may
			// already be in _Gwaiting if this is a mark
			// worker or we're in mark termination.
			userG := getg().m.curg
			selfScan := gp == userG && readgstatus(userG) == _Grunning
			if selfScan {
				casgstatus(userG, _Grunning, _Gwaiting)
				userG.waitreason = waitReasonGarbageCollectionScan
			}

			// TODO: scang blocks until gp's stack has
			// been scanned, which may take a while for
			// running goroutines. Consider doing this in
			// two phases where the first is non-blocking:
			// we scan the stacks we can and ask running
			// goroutines to scan themselves; and the
			// second blocks.
			scang(gp, gcw)

			if selfScan {
				casgstatus(userG, _Gwaiting, _Grunning)
			}
		})
	}
}
```

mstats.go

```go
func flushmcache(i int) {
	p := allp[i]
	c := p.mcache
	if c == nil {
		return
	}
	c.releaseAll()
	stackcache_clear(c)
}
```

mgc.go

```go
func gcMark(start_time int64) {
	if debug.allocfreetrace > 0 {
		tracegc()
	}

	if gcphase != _GCmarktermination {
		throw("in gcMark expecting to see gcphase as _GCmarktermination")
	}
	work.tstart = start_time

	// Check that there's no marking work remaining.
	if work.full != 0 || work.markrootNext < work.markrootJobs {
		print("runtime: full=", hex(work.full), " next=", work.markrootNext, " jobs=", work.markrootJobs, " nDataRoots=", work.nDataRoots, " nBSSRoots=", work.nBSSRoots, " nSpanRoots=", work.nSpanRoots, " nStackRoots=", work.nStackRoots, "\n")
		panic("non-empty mark queue after concurrent mark")
	}

	if debug.gccheckmark > 0 {
		// This is expensive when there's a large number of
		// Gs, so only do it if checkmark is also enabled.
		gcMarkRootCheck()
	}
	if work.full != 0 {
		throw("work.full != 0")
	}

	// Clear out buffers and double-check that all gcWork caches
	// are empty. This should be ensured by gcMarkDone before we
	// enter mark termination.
	//
	// TODO: We could clear out buffers just before mark if this
	// has a non-negligible impact on STW time.
	for _, p := range allp {
		// The write barrier may have buffered pointers since
		// the gcMarkDone barrier. However, since the barrier
		// ensured all reachable objects were marked, all of
		// these must be pointers to black objects. Hence we
		// can just discard the write barrier buffer.
		if debug.gccheckmark > 0 || throwOnGCWork {
			// For debugging, flush the buffer and make
			// sure it really was all marked.
			wbBufFlush1(p)
		} else {
			p.wbBuf.reset()
		}

		gcw := &p.gcw
		if !gcw.empty() {
			printlock()
			print("runtime: P ", p.id, " flushedWork ", gcw.flushedWork)
			if gcw.wbuf1 == nil {
				print(" wbuf1=<nil>")
			} else {
				print(" wbuf1.n=", gcw.wbuf1.nobj)
			}
			if gcw.wbuf2 == nil {
				print(" wbuf2=<nil>")
			} else {
				print(" wbuf2.n=", gcw.wbuf2.nobj)
			}
			print("\n")
			throw("P has cached GC work at end of mark termination")
		}
		// There may still be cached empty buffers, which we
		// need to flush since we're going to free them. Also,
		// there may be non-zero stats because we allocated
		// black after the gcMarkDone barrier.
		gcw.dispose()
	}

	throwOnGCWork = false

	cachestats()

	// Update the marked heap stat.
	memstats.heap_marked = work.bytesMarked

	// Update other GC heap size stats. This must happen after
	// cachestats (which flushes local statistics to these) and
	// flushallmcaches (which modifies heap_live).
	memstats.heap_live = work.bytesMarked
	memstats.heap_scan = uint64(gcController.scanWork)

	if trace.enabled {
		traceHeapAlloc()
	}
}
```

因垃圾回收需要，`stackcache_clear`会将所有cache缓存的栈空间归还给全局或heap。

```go
//go:systemstack
func stackcache_clear(c *mcache) {
	if stackDebug >= 1 {
		print("stackcache clear\n")
	}
	lock(&stackpoolmu)
	for order := uint8(0); order < _NumStackOrders; order++ {
		x := c.stackcache[order].list
		for x.ptr() != nil {
			y := x.ptr().next
			stackpoolfree(x, order)
			x = y
		}
		c.stackcache[order].list = 0
		c.stackcache[order].size = 0
	}
	unlock(&stackpoolmu)
}
```

以及栈收缩：`shrinkstack`除收回闲置栈空间外，还会收缩正在使用但被扩容过的栈以节约内存。

```go
func shrinkstack(gp *g) {
	gstatus := readgstatus(gp)
	if gp.stack.lo == 0 {
		throw("missing stack in shrinkstack")
	}
	if gstatus&_Gscan == 0 {
		throw("bad status in shrinkstack")
	}

	if debug.gcshrinkstackoff > 0 {
		return
	}
	f := findfunc(gp.startpc)
	if f.valid() && f.funcID == funcID_gcBgMarkWorker {
		// We're not allowed to shrink the gcBgMarkWorker
		// stack (see gcBgMarkWorker for explanation).
		return
	}

  // 收缩目标是一半大小
	oldsize := gp.stack.hi - gp.stack.lo
	newsize := oldsize / 2
	// Don't shrink the allocation below the minimum-sized stack
	// allocation.
	if newsize < _FixedStack {
		return
	}
	// Compute how much of the stack is currently in use and only
	// shrink the stack if gp is using less than a quarter of its
	// current stack. The currently used stack includes everything
	// down to the SP plus the stack guard space that ensures
	// there's room for nosplit functions.
  // 如果使用的空间超过1/4，则不收缩。
	avail := gp.stack.hi - gp.stack.lo
	if used := gp.stack.hi - gp.sched.sp + _StackLimit; used >= avail/4 {
		return
	}

	// We can't copy the stack if we're in a syscall.
	// The syscall might have pointers into the stack.
	if gp.syscallsp != 0 {
		return
	}
	if sys.GoosWindows != 0 && gp.m != nil && gp.m.libcallsp != 0 {
		return
	}

	if stackDebug > 0 {
		print("shrinking stack ", oldsize, "->", newsize, "\n")
	}

  // 用较小的栈替换
	copystack(gp, newsize, false)
}
```

最后就是`freeStackSpans`，它扫描全局队列`stackpool`和暂存队列`stackFreeQueue`，将那些空间已完全收回的`span`交还给`heap`。

stack.go

```go
// freeStackSpans frees unused stack spans at the end of GC.
func freeStackSpans() {
	lock(&stackpoolmu)

	// Scan stack pools for empty stack spans.
	for order := range stackpool {
		list := &stackpool[order]
		for s := list.first; s != nil; {
			next := s.next
			if s.allocCount == 0 {
				list.remove(s)
				s.manualFreeList = 0
				osStackFree(s)
				mheap_.freeManual(s, &memstats.stacks_inuse)
			}
			s = next
		}
	}

	unlock(&stackpoolmu)

	// Free large stack spans.
	lock(&stackLarge.lock)
	for i := range stackLarge.free {
		for s := stackLarge.free[i].first; s != nil; {
			next := s.next
			stackLarge.free[i].remove(s)
			osStackFree(s)
			mheap_.freeManual(s, &memstats.stacks_inuse)
			s = next
		}
	}
	unlock(&stackLarge.lock)
}
```

另外，调整P数量的`procresize`，将任务完成的G对象放回复用链表的`gfput`，同样会引发栈空间释放操作。只是流程和上述基本类似，不再赘述。



## 8.6.系统调用

为支持并发调度，Go专门对`syscall`，`cgo`进行了包装，以便在长时间阻塞时能切换执行其他任务。在标准库syscall包里，将系统调用函数分为`Syscall`和`RawSyscall`两类。比如`Getcwd`和`EpollCreate`函数：

src/syscall/zsyscall_linux_amd64.s

```go
func Getcwd(buf []byte) (n int, err error) {
	var _p0 unsafe.Pointer
	if len(buf) > 0 {
		_p0 = unsafe.Pointer(&buf[0])
	} else {
		_p0 = unsafe.Pointer(&_zero)
	}
	r0, _, e1 := Syscall(SYS_GETCWD, uintptr(_p0), uintptr(len(buf)), 0)
	n = int(r0)
	if e1 != 0 {
		err = errnoErr(e1)
	}
	return
}


func EpollCreate(size int) (fd int, err error) {
	r0, _, e1 := RawSyscall(SYS_EPOLL_CREATE, uintptr(size), 0, 0)
	fd = int(r0)
	if e1 != 0 {
		err = errnoErr(e1)
	}
	return
}
```

到底，`Syscall`和`RawSyscall`之间的区别在哪里呢？

src/syscall/asm_linux_amd64.s

```bash
TEXT ·Syscall(SB),NOSPLIT,$0-56
	BL	runtime·entersyscall(SB)
	MOVD	a1+8(FP), R0
	MOVD	a2+16(FP), R1
	MOVD	a3+24(FP), R2
	MOVD	$0, R3
	MOVD	$0, R4
	MOVD	$0, R5
	MOVD	trap+0(FP), R8	// syscall entry
	SVC
	CMN	$4095, R0
	BCC	ok
	MOVD	$-1, R4
	MOVD	R4, r1+32(FP)	// r1
	MOVD	ZR, r2+40(FP)	// r2
	NEG	R0, R0
	MOVD	R0, err+48(FP)	// errno
	BL	runtime·exitsyscall(SB)  // 关键的差异
	RET
ok:
	MOVD	R0, r1+32(FP)	// r1
	MOVD	R1, r2+40(FP)	// r2
	MOVD	ZR, err+48(FP)	// errno
	BL	runtime·exitsyscall(SB)
	RET
	
	
TEXT ·RawSyscall(SB),NOSPLIT,$0-56
	MOVD	a1+8(FP), R0
	MOVD	a2+16(FP), R1
	MOVD	a3+24(FP), R2
	MOVD	$0, R3
	MOVD	$0, R4
	MOVD	$0, R5
	MOVD	trap+0(FP), R8	// syscall entry
	SVC
	CMN	$4095, R0
	BCC	ok
	MOVD	$-1, R4
	MOVD	R4, r1+32(FP)	// r1
	MOVD	ZR, r2+40(FP)	// r2
	NEG	R0, R0
	MOVD	R0, err+48(FP)	// errno
	RET
ok:
	MOVD	R0, r1+32(FP)	// r1
	MOVD	R1, r2+40(FP)	// r2
	MOVD	ZR, err+48(FP)	// errno
	RET
```

最大的不同在于`Syscall`增加了`entersyscall/exitsyscall`，这就是允许调度的关键所在。

然后看下`exitsyscall`的实现：

proc.go

```go
// Standard syscall entry used by the go syscall library and normal cgo calls.
//
// This is exported via linkname to assembly in the syscall package.
//
//go:nosplit
//go:linkname entersyscall
func entersyscall() {
	reentersyscall(getcallerpc(), getcallersp())
}

func reentersyscall(pc, sp uintptr) {
	_g_ := getg()

	// Disable preemption because during this function g is in Gsyscall status,
	// but can have inconsistent g->sched, do not let GC observe it.
	_g_.m.locks++

	// Entersyscall must not call any function that might split/grow the stack.
	// (See details in comment above.)
	// Catch calls that might, by replacing the stack guard with something that
	// will trip any stack check and leaving a flag to tell newstack to die.
	_g_.stackguard0 = stackPreempt
	_g_.throwsplit = true

	// Leave SP around for GC and traceback.
  // 保存执行现场
	save(pc, sp)
	_g_.syscallsp = sp
	_g_.syscallpc = pc
	casgstatus(_g_, _Grunning, _Gsyscall)
	if _g_.syscallsp < _g_.stack.lo || _g_.stack.hi < _g_.syscallsp {
		systemstack(func() {
			print("entersyscall inconsistent ", hex(_g_.syscallsp), " [", hex(_g_.stack.lo), ",", hex(_g_.stack.hi), "]\n")
			throw("entersyscall")
		})
	}

	if trace.enabled {
		systemstack(traceGoSysCall)
		// systemstack itself clobbers g.sched.{pc,sp} and we might
		// need them later when the G is genuinely blocked in a
		// syscall
		save(pc, sp)
	}

  // 确保sysmon运行
	if atomic.Load(&sched.sysmonwait) != 0 {
		systemstack(entersyscall_sysmon)
		save(pc, sp)
	}

	if _g_.m.p.ptr().runSafePointFn != 0 {
		// runSafePointFn may stack split if run on this stack
		systemstack(runSafePointFn)
		save(pc, sp)
	}

  // 设置相关状态
	_g_.m.syscalltick = _g_.m.p.ptr().syscalltick
	_g_.sysblocktraced = true
	_g_.m.mcache = nil
	pp := _g_.m.p.ptr()
	pp.m = 0
	_g_.m.oldp.set(pp)
	_g_.m.p = 0
	atomic.Store(&pp.status, _Psyscall)
	if sched.gcwaiting != 0 {
		systemstack(entersyscall_gcwait)
		save(pc, sp)
	}

	_g_.m.locks--
}
```

监控线程`sysmon`对`syscall`非常重要，因为它负责将因系统调用而长时间阻塞的P抢回，用于执行其他任务。否则，整体性能会严重下降，甚至整个进程都会被冻结。

再看下`entersyscall_sysmon`的实现：

```go
func entersyscall_sysmon() {
	lock(&sched.lock)
	if atomic.Load(&sched.sysmonwait) != 0 {
		atomic.Store(&sched.sysmonwait, 0)
		notewakeup(&sched.sysmonnote)
	}
	unlock(&sched.lock)
}
```

某些系统调用本身就可以确定长时间阻塞（比如锁），那么它会选择执行`entersyscallblock`主动交出所关联的P。

```go
// The same as entersyscall(), but with a hint that the syscall is blocking.
//go:nosplit
func entersyscallblock() {
	_g_ := getg()

	_g_.m.locks++ // see comment in entersyscall
	_g_.throwsplit = true
	_g_.stackguard0 = stackPreempt // see comment in entersyscall
	_g_.m.syscalltick = _g_.m.p.ptr().syscalltick
	_g_.sysblocktraced = true
	_g_.m.p.ptr().syscalltick++

	// Leave SP around for GC and traceback.
	pc := getcallerpc()
	sp := getcallersp()
	save(pc, sp)
	_g_.syscallsp = _g_.sched.sp
	_g_.syscallpc = _g_.sched.pc
	if _g_.syscallsp < _g_.stack.lo || _g_.stack.hi < _g_.syscallsp {
		sp1 := sp
		sp2 := _g_.sched.sp
		sp3 := _g_.syscallsp
		systemstack(func() {
			print("entersyscallblock inconsistent ", hex(sp1), " ", hex(sp2), " ", hex(sp3), " [", hex(_g_.stack.lo), ",", hex(_g_.stack.hi), "]\n")
			throw("entersyscallblock")
		})
	}
	casgstatus(_g_, _Grunning, _Gsyscall)
	if _g_.syscallsp < _g_.stack.lo || _g_.stack.hi < _g_.syscallsp {
		systemstack(func() {
			print("entersyscallblock inconsistent ", hex(sp), " ", hex(_g_.sched.sp), " ", hex(_g_.syscallsp), " [", hex(_g_.stack.lo), ",", hex(_g_.stack.hi), "]\n")
			throw("entersyscallblock")
		})
	}

	systemstack(entersyscallblock_handoff)

	// Resave for traceback during blocked call.
	save(getcallerpc(), getcallersp())

	_g_.m.locks--
}

func entersyscallblock_handoff() {
	if trace.enabled {
		traceGoSysCall()
		traceGoSysBlock(getg().m.p.ptr())
	}
  // 释放P，让它去执行其他任务
	handoffp(releasep())
}

// Hands off P from syscall or locked M.
// Always runs without a P, so write barriers are not allowed.
//go:nowritebarrierrec
func handoffp(_p_ *p) {
	// handoffp must start an M in any situation where
	// findrunnable would return a G to run on _p_.

	// if it has local work, start it straight away
  // 如果P本地或全局有任务，直接唤醒某个M开始工作。
	if !runqempty(_p_) || sched.runqsize != 0 {
		startm(_p_, false)
		return
	}
	// if it has GC work, start it straight away
	if gcBlackenEnabled != 0 && gcMarkWorkAvailable(_p_) {
		startm(_p_, false)
		return
	}
	// no local work, check that there are no spinning/idle M's,
	// otherwise our help is not required
	if atomic.Load(&sched.nmspinning)+atomic.Load(&sched.npidle) == 0 && atomic.Cas(&sched.nmspinning, 0, 1) { // TODO: fast atomic
		startm(_p_, true)
		return
	}
	lock(&sched.lock)
	if sched.gcwaiting != 0 {
		_p_.status = _Pgcstop
		sched.stopwait--
		if sched.stopwait == 0 {
			notewakeup(&sched.stopnote)
		}
		unlock(&sched.lock)
		return
	}
	if _p_.runSafePointFn != 0 && atomic.Cas(&_p_.runSafePointFn, 1, 0) {
		sched.safePointFn(_p_)
		sched.safePointWait--
		if sched.safePointWait == 0 {
			notewakeup(&sched.safePointNote)
		}
	}
	if sched.runqsize != 0 {
		unlock(&sched.lock)
		startm(_p_, false)
		return
	}
	// If this is the last running P and nobody is polling network,
	// need to wakeup another M to poll network.
	if sched.npidle == uint32(gomaxprocs-1) && atomic.Load64(&sched.lastpoll) != 0 {
		unlock(&sched.lock)
		startm(_p_, false)
		return
	}
  // 没有任务就放回空闲队列
	pidleput(_p_)
	unlock(&sched.lock)
}
```

注意：从系统调用返回时，必须检查P是否依然可用，因为可能已被`sysmon`抢走。

```go
func exitsyscall() {
	_g_ := getg()

	_g_.m.locks++ // see comment in entersyscall
	if getcallersp() > _g_.syscallsp {
		throw("exitsyscall: syscall frame is no longer valid")
	}

	_g_.waitsince = 0
	oldp := _g_.m.oldp.ptr()
	_g_.m.oldp = 0
	if exitsyscallfast(oldp) {
		if _g_.m.mcache == nil {
			throw("lost mcache")
		}
		if trace.enabled {
			if oldp != _g_.m.p.ptr() || _g_.m.syscalltick != _g_.m.p.ptr().syscalltick {
				systemstack(traceGoStart)
			}
		}
		// There's a cpu for us, so we can run.
		_g_.m.p.ptr().syscalltick++
		// We need to cas the status and scan before resuming...
		casgstatus(_g_, _Gsyscall, _Grunning)

		// Garbage collector isn't running (since we are),
		// so okay to clear syscallsp.
		_g_.syscallsp = 0
		_g_.m.locks--
		if _g_.preempt {
			// restore the preemption request in case we've cleared it in newstack
			_g_.stackguard0 = stackPreempt
		} else {
			// otherwise restore the real _StackGuard, we've spoiled it in entersyscall/entersyscallblock
			_g_.stackguard0 = _g_.stack.lo + _StackGuard
		}
		_g_.throwsplit = false

		if sched.disable.user && !schedEnabled(_g_) {
			// Scheduling of this goroutine is disabled.
			Gosched()
		}

		return
	}

	_g_.sysexitticks = 0
	if trace.enabled {
		// Wait till traceGoSysBlock event is emitted.
		// This ensures consistency of the trace (the goroutine is started after it is blocked).
		for oldp != nil && oldp.syscalltick == _g_.m.syscalltick {
			osyield()
		}
		// We can't trace syscall exit right now because we don't have a P.
		// Tracing code can invoke write barriers that cannot run without a P.
		// So instead we remember the syscall exit time and emit the event
		// in execute when we have a P.
		_g_.sysexitticks = cputicks()
	}

	_g_.m.locks--

	// Call the scheduler.
	mcall(exitsyscall0)

	if _g_.m.mcache == nil {
		throw("lost mcache")
	}

	// Scheduler returned, so we're allowed to run now.
	// Delete the syscallsp information that we left for
	// the garbage collector during the system call.
	// Must wait until now because until gosched returns
	// we don't know for sure that the garbage collector
	// is not running.
	_g_.syscallsp = 0
	_g_.m.p.ptr().syscalltick++
	_g_.throwsplit = false
}
```

其中，快速退出`exitsyscallfast`是指能重新绑定原有或空闲的P，以继续当前G任务的执行。

```go
//go:nosplit
func exitsyscallfast(oldp *p) bool {
	_g_ := getg()

	// Freezetheworld sets stopwait but does not retake P's.
  // 当前处于STW状态，就不要继续了。
	if sched.stopwait == freezeStopWait {
		return false
	}

	// Try to re-acquire the last P.
  // 尝试关联原来的P
	if oldp != nil && oldp.status == _Psyscall && atomic.Cas(&oldp.status, _Psyscall, _Pidle) {
		// There's a cpu for us, so we can run.
		wirep(oldp)
		exitsyscallfast_reacquired()
		return true
	}

	// Try to get any other idle P.
  // 获取其他的
	if sched.pidle != 0 {
		var ok bool
		systemstack(func() {
			ok = exitsyscallfast_pidle()
			if ok && trace.enabled {
				if oldp != nil {
					// Wait till traceGoSysBlock event is emitted.
					// This ensures consistency of the trace (the goroutine is started after it is blocked).
					for oldp.syscalltick == _g_.m.syscalltick {
						osyield()
					}
				}
				traceGoSysExit(0)
			}
		})
		if ok {
			return true
		}
	}
	return false
}

func exitsyscallfast_pidle() bool {
	lock(&sched.lock)
	_p_ := pidleget()
  // 唤醒sysmon
	if _p_ != nil && atomic.Load(&sched.sysmonwait) != 0 {
		atomic.Store(&sched.sysmonwait, 0)
		notewakeup(&sched.sysmonnote)
	}
	unlock(&sched.lock)
  // 重新关联
	if _p_ != nil {
		acquirep(_p_)
		return true
	}
	return false
}
```

如果多次尝试绑定P却失败，那么只能将当前任务放入待运行队列。

再看看`exitsyscall0`的逻辑如下：

```go
// exitsyscall slow path on g0.
// Failed to acquire P, enqueue gp as runnable.
//
//go:nowritebarrierrec
func exitsyscall0(gp *g) {
	_g_ := getg()

  // 修改状态，解除和M的关联。
	casgstatus(gp, _Gsyscall, _Grunnable)
	dropg()
	lock(&sched.lock)
	var _p_ *p
  // 再次获取空闲P
	if schedEnabled(_g_) {
		_p_ = pidleget()
	}
	if _p_ == nil {
    // 获取失败，放回全局任务队列
		globrunqput(gp)
	} else if atomic.Load(&sched.sysmonwait) != 0 {
		atomic.Store(&sched.sysmonwait, 0)
		notewakeup(&sched.sysmonnote)
	}
	unlock(&sched.lock)
  
  // 再次检查P，以便执行当前任务
	if _p_ != nil {
		acquirep(_p_)
		execute(gp, false) // Never returns.
	}
	if _g_.m.lockedg != 0 {
		// Wait until another thread schedules gp and so m again.
		stoplockedm()
		execute(gp, false) // Never returns.
	}
  
  // 关联P失败，休眠当前M
	stopm()
	schedule() // Never returns.
}
```

`cgocall`也使用了通用的封装方式，因此它同样不受调度器的管理。

```go
// Call from Go to C.
//go:nosplit
func cgocall(fn, arg unsafe.Pointer) int32 {
	if !iscgo && GOOS != "solaris" && GOOS != "illumos" && GOOS != "windows" {
		throw("cgocall unavailable")
	}

	if fn == nil {
		throw("cgocall nil")
	}

	if raceenabled {
		racereleasemerge(unsafe.Pointer(&racecgosync))
	}

	mp := getg().m
	mp.ncgocall++
	mp.ncgo++

	// Reset traceback.
	mp.cgoCallers[0] = 0

	// Announce we are entering a system call
	// so that the scheduler knows to create another
	// M to run goroutines while we are in the
	// foreign code.
	//
	// The call to asmcgocall is guaranteed not to
	// grow the stack and does not allocate memory,
	// so it is safe to call while "in a system call", outside
	// the $GOMAXPROCS accounting.
	//
	// fn may call back into Go code, in which case we'll exit the
	// "system call", run the Go code (which may grow the stack),
	// and then re-enter the "system call" reusing the PC and SP
	// saved by entersyscall here.
	entersyscall()

	mp.incgo = true
	errno := asmcgocall(fn, arg)

	// Update accounting before exitsyscall because exitsyscall may
	// reschedule us on to a different M.
	mp.incgo = false
	mp.ncgo--

	exitsyscall()

	// Note that raceacquire must be called only after exitsyscall has
	// wired this M to a P.
	if raceenabled {
		raceacquire(unsafe.Pointer(&racecgosync))
	}

	// From the garbage collector's perspective, time can move
	// backwards in the sequence above. If there's a callback into
	// Go code, GC will see this function at the call to
	// asmcgocall. When the Go call later returns to C, the
	// syscall PC/SP is rolled back and the GC sees this function
	// back at the call to entersyscall. Normally, fn and arg
	// would be live at entersyscall and dead at asmcgocall, so if
	// time moved backwards, GC would see these arguments as dead
	// and then live. Prevent these undead arguments from crashing
	// GC by forcing them to stay live across this time warp.
	KeepAlive(fn)
	KeepAlive(arg)
	KeepAlive(mp)

	return errno
}
```



## 8.7.监控

监控总结：

+ 释放闲置超过5分钟的span物理内存；
+ 如果超过2分钟没有垃圾回收，则强制执行；
+ 将长时间未处理的netpoll结果添加到任务队列；
+ 向长时间运行的G任务发出抢占调度；
+ 收回因syscall而长时间阻塞的P。

在进入垃圾回收状态时，`sysmon`会自动进入休眠，所以我们才会在`syscall`里看到很多唤醒指令。另外，`startTheWorld`也会做唤醒处理。保证监控线程正常运行，对内存分配，垃圾回收和并发调度都非常重要。

`startTheWorldWithSema`逻辑如下：

```go
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
	p1 := procresize(procs)
	sched.gcwaiting = 0
	if sched.sysmonwait != 0 {
		sched.sysmonwait = 0
		notewakeup(&sched.sysmonnote)
	}
	unlock(&sched.lock)

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
	if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 {
		wakep()
	}

	releasem(mp)

	return startTime
}
```

再看对`syscall`和`preempt`的处理：

```go
func sysmon() {
	lock(&sched.lock)
	sched.nmsys++
	checkdead()
	unlock(&sched.lock)

	lasttrace := int64(0)
	idle := 0 // how many cycles in succession we had not wokeup somebody
	delay := uint32(0)
	for {
		if idle == 0 { // start with 20us sleep...
			delay = 20
		} else if idle > 50 { // start doubling the sleep after 1ms...
			delay *= 2
		}
		if delay > 10*1000 { // up to 10ms
			delay = 10 * 1000
		}
		usleep(delay)
    // STW时休眠sysmon
		if debug.schedtrace <= 0 && (sched.gcwaiting != 0 || atomic.Load(&sched.npidle) == uint32(gomaxprocs)) {
			lock(&sched.lock)
			if atomic.Load(&sched.gcwaiting) != 0 || atomic.Load(&sched.npidle) == uint32(gomaxprocs) {
        // 设置休眠标志，休眠（有个超时，苏醒保障）
				atomic.Store(&sched.sysmonwait, 1)
				unlock(&sched.lock)
				// Make wake-up period small enough
				// for the sampling to be correct.
				maxsleep := forcegcperiod / 2
				shouldRelax := true
				if osRelaxMinNS > 0 {
					next := timeSleepUntil()
					now := nanotime()
					if next-now < osRelaxMinNS {
						shouldRelax = false
					}
				}
				if shouldRelax {
					osRelax(true)
				}
				notetsleep(&sched.sysmonnote, maxsleep)
				if shouldRelax {
					osRelax(false)
				}
				lock(&sched.lock)
        // 唤醒后重置状态，继续执行
				atomic.Store(&sched.sysmonwait, 0)
				noteclear(&sched.sysmonnote)
				idle = 0
				delay = 20
			}
			unlock(&sched.lock)
		}
		// trigger libc interceptors if needed
		if *cgo_yield != nil {
			asmcgocall(*cgo_yield, nil)
		}
		// poll network if not polled for more than 10ms
		lastpoll := int64(atomic.Load64(&sched.lastpoll))
		now := nanotime()
    // 获取超过10ms的netpoll结果
		if netpollinited() && lastpoll != 0 && lastpoll+10*1000*1000 < now {
			atomic.Cas64(&sched.lastpoll, uint64(lastpoll), uint64(now))
			list := netpoll(false) // non-blocking - returns list of goroutines
			if !list.empty() {
				// Need to decrement number of idle locked M's
				// (pretending that one more is running) before injectglist.
				// Otherwise it can lead to the following situation:
				// injectglist grabs all P's but before it starts M's to run the P's,
				// another M returns from syscall, finishes running its G,
				// observes that there is no work to do and no other running M's
				// and reports deadlock.
				incidlelocked(-1)
				injectglist(&list)
				incidlelocked(1)
			}
		}
		// retake P's blocked in syscalls
		// and preempt long running G's
    // 抢夺syscall长时间阻塞的P
    // 向长时间运行的G发出抢占调度
		if retake(now) != 0 {
			idle = 0
		} else {
			idle++
		}
		// check if we need to force a GC
		if t := (gcTrigger{kind: gcTriggerTime, now: now}); t.test() && atomic.Load(&forcegc.idle) != 0 {
			lock(&forcegc.lock)
			forcegc.idle = 0
			var list gList
			list.push(forcegc.g)
			injectglist(&list)
			unlock(&forcegc.lock)
		}
		if debug.schedtrace > 0 && lasttrace+int64(debug.schedtrace)*1000000 <= now {
			lasttrace = now
			schedtrace(debug.scheddetail > 0)
		}
	}
}
```

对于超时的统计，专门有个`pdesc`的全局变量用于保存`sysmon`运行统计信息，据此来判断`syscall`和G是否超时

。

```go
// forcePreemptNS is the time slice given to a G before it is
// preempted.
const forcePreemptNS = 10 * 1000 * 1000 // 10ms

func retake(now int64) uint32 {
	n := 0
	// Prevent allp slice changes. This lock will be completely
	// uncontended unless we're already stopping the world.
	lock(&allpLock)
	// We can't use a range loop over allp because we may
	// temporarily drop the allpLock. Hence, we need to re-fetch
	// allp each time around the loop.
  // 遍历P
	for i := 0; i < len(allp); i++ {
		_p_ := allp[i]
		if _p_ == nil {
			// This can happen if procresize has grown
			// allp but not yet created new Ps.
			continue
		}
		pd := &_p_.sysmontick
		s := _p_.status
		sysretake := false
		if s == _Prunning || s == _Psyscall {
			// Preempt G if it's running for too long.
      // 更新G运行统计信息
			t := int64(_p_.schedtick)
			if int64(pd.schedtick) != t {
				pd.schedtick = uint32(t)
				pd.schedwhen = now
			} else if pd.schedwhen+forcePreemptNS <= now {
        // 发出抢占调度
				preemptone(_p_)
				// In case of syscall, preemptone() doesn't
				// work, because there is no M wired to P.
				sysretake = true
			}
		}
    // P处于syscall模式
		if s == _Psyscall {
			// Retake P from syscall if it's there for more than 1 sysmon tick (at least 20us).
      // 更新syscall统计信息
			t := int64(_p_.syscalltick)
			if !sysretake && int64(pd.syscalltick) != t {
				pd.syscalltick = uint32(t)
				pd.syscallwhen = now
				continue
			}
			// On the one hand we don't want to retake Ps if there is no other work to do,
			// but on the other hand we want to retake them eventually
			// because they can prevent the sysmon thread from deep sleep.
      // 检查是否有其他任务需要P，是否超出时间限制，是否有必要抢夺P
			if runqempty(_p_) && atomic.Load(&sched.nmspinning)+atomic.Load(&sched.npidle) > 0 && pd.syscallwhen+10*1000*1000 > now {
				continue
			}
			// Drop allpLock so we can take sched.lock.
			unlock(&allpLock)
			// Need to decrement number of idle locked M's
			// (pretending that one more is running) before the CAS.
			// Otherwise the M from which we retake can exit the syscall,
			// increment nmidle and report deadlock.
			incidlelocked(-1)
      // 抢夺P
			if atomic.Cas(&_p_.status, s, _Pidle) {
				if trace.enabled {
					traceGoSysBlock(_p_)
					traceProcStop(_p_)
				}
				n++
				_p_.syscalltick++
				handoffp(_p_)
			}
			incidlelocked(1)
			lock(&allpLock)
		}
	}
	unlock(&allpLock)
	return uint32(n)
}
```

**抢占调度**

golang的抢占调度做得比较简单：并不是类似”抢占式多任务操作系统“那种。因为Go调度器并没有真正意义上的时间片概念，只是在目标G上设置一个抢占标志，当该任务调用某个函数时，被编译器安插的指令就会检查这个标志，从而决定是否暂停当前任务。

```go
func preemptone(_p_ *p) bool {
	mp := _p_.m.ptr()
	if mp == nil || mp == getg().m {
		return false
	}
	gp := mp.curg
	if gp == nil || gp == mp.g0 {
		return false
	}

	gp.preempt = true

	// Every call in a go routine checks for stack overflow by
	// comparing the current stack pointer to gp->stackguard0.
	// Setting gp->stackguard0 to StackPreempt folds
	// preemption into the normal stack overflow check.
	gp.stackguard0 = stackPreempt
	return true
}
```

有两个标志，实际起作用的是`G.stackguard0`。`G.preempt`只是后备，以便在`stackguard0`做回溢出检查标志时，依然可用`preempt`恢复抢占状态。

这个标志是由编译器插入的指令（`morestack`）。当它调用`newstack`扩容时会检查抢占标志，并决定是否暂停当前任务，当然这发生在实际扩容之前。

```go
func newstack() {
	thisg := getg()
	// TODO: double check all gp. shouldn't be getg().
	if thisg.m.morebuf.g.ptr().stackguard0 == stackFork {
		throw("stack growth after fork")
	}
	if thisg.m.morebuf.g.ptr() != thisg.m.curg {
		print("runtime: newstack called from g=", hex(thisg.m.morebuf.g), "\n"+"\tm=", thisg.m, " m->curg=", thisg.m.curg, " m->g0=", thisg.m.g0, " m->gsignal=", thisg.m.gsignal, "\n")
		morebuf := thisg.m.morebuf
		traceback(morebuf.pc, morebuf.sp, morebuf.lr, morebuf.g.ptr())
		throw("runtime: wrong goroutine in newstack")
	}

	gp := thisg.m.curg

	if thisg.m.curg.throwsplit {
		// Update syscallsp, syscallpc in case traceback uses them.
		morebuf := thisg.m.morebuf
		gp.syscallsp = morebuf.sp
		gp.syscallpc = morebuf.pc
		pcname, pcoff := "(unknown)", uintptr(0)
		f := findfunc(gp.sched.pc)
		if f.valid() {
			pcname = funcname(f)
			pcoff = gp.sched.pc - f.entry
		}
		print("runtime: newstack at ", pcname, "+", hex(pcoff),
			" sp=", hex(gp.sched.sp), " stack=[", hex(gp.stack.lo), ", ", hex(gp.stack.hi), "]\n",
			"\tmorebuf={pc:", hex(morebuf.pc), " sp:", hex(morebuf.sp), " lr:", hex(morebuf.lr), "}\n",
			"\tsched={pc:", hex(gp.sched.pc), " sp:", hex(gp.sched.sp), " lr:", hex(gp.sched.lr), " ctxt:", gp.sched.ctxt, "}\n")

		thisg.m.traceback = 2 // Include runtime frames
		traceback(morebuf.pc, morebuf.sp, morebuf.lr, gp)
		throw("runtime: stack split at bad time")
	}

	morebuf := thisg.m.morebuf
	thisg.m.morebuf.pc = 0
	thisg.m.morebuf.lr = 0
	thisg.m.morebuf.sp = 0
	thisg.m.morebuf.g = 0

	// NOTE: stackguard0 may change underfoot, if another thread
	// is about to try to preempt gp. Read it just once and use that same
	// value now and below.
	preempt := atomic.Loaduintptr(&gp.stackguard0) == stackPreempt

	// Be conservative about where we preempt.
	// We are interested in preempting user Go code, not runtime code.
	// If we're holding locks, mallocing, or preemption is disabled, don't
	// preempt.
	// This check is very early in newstack so that even the status change
	// from Grunning to Gwaiting and back doesn't happen in this case.
	// That status change by itself can be viewed as a small preemption,
	// because the GC might change Gwaiting to Gscanwaiting, and then
	// this goroutine has to wait for the GC to finish before continuing.
	// If the GC is in some way dependent on this goroutine (for example,
	// it needs a lock held by the goroutine), that small preemption turns
	// into a real deadlock.
	if preempt {
    // 如果M持有锁，或者正在进行内存分配，垃圾回收等操作，不抢占，留待下次
		if thisg.m.locks != 0 || thisg.m.mallocing != 0 || thisg.m.preemptoff != "" || thisg.m.p.ptr().status != _Prunning {
			// Let the goroutine keep running for now.
			// gp->preempt is set, so it will be preempted next time.
      // stackguard0恢复溢出检查用途，下次用G.preempt恢复
			gp.stackguard0 = gp.stack.lo + _StackGuard
			gogo(&gp.sched) // never return
		}
	}

	if gp.stack.lo == 0 {
		throw("missing stack in newstack")
	}
	sp := gp.sched.sp
	if sys.ArchFamily == sys.AMD64 || sys.ArchFamily == sys.I386 || sys.ArchFamily == sys.WASM {
		// The call to morestack cost a word.
		sp -= sys.PtrSize
	}
	if stackDebug >= 1 || sp < gp.stack.lo {
		print("runtime: newstack sp=", hex(sp), " stack=[", hex(gp.stack.lo), ", ", hex(gp.stack.hi), "]\n",
			"\tmorebuf={pc:", hex(morebuf.pc), " sp:", hex(morebuf.sp), " lr:", hex(morebuf.lr), "}\n",
			"\tsched={pc:", hex(gp.sched.pc), " sp:", hex(gp.sched.sp), " lr:", hex(gp.sched.lr), " ctxt:", gp.sched.ctxt, "}\n")
	}
	if sp < gp.stack.lo {
		print("runtime: gp=", gp, ", goid=", gp.goid, ", gp->status=", hex(readgstatus(gp)), "\n ")
		print("runtime: split stack overflow: ", hex(sp), " < ", hex(gp.stack.lo), "\n")
		throw("runtime: split stack overflow")
	}

	if preempt {
		if gp == thisg.m.g0 {
			throw("runtime: preempt g0")
		}
		if thisg.m.p == 0 && thisg.m.locks == 0 {
			throw("runtime: g is running but p is not")
		}
		// Synchronize with scang.
		casgstatus(gp, _Grunning, _Gwaiting)
    // 垃圾回收本身也算一次抢占，忽略本次抢占调度
		if gp.preemptscan {
			for !castogscanstatus(gp, _Gwaiting, _Gscanwaiting) {
				// Likely to be racing with the GC as
				// it sees a _Gwaiting and does the
				// stack scan. If so, gcworkdone will
				// be set and gcphasework will simply
				// return.
			}
			if !gp.gcscandone {
				// gcw is safe because we're on the
				// system stack.
				gcw := &gp.m.p.ptr().gcw
				scanstack(gp, gcw)
				gp.gcscandone = true
			}
			gp.preemptscan = false
			gp.preempt = false
			casfrom_Gscanstatus(gp, _Gscanwaiting, _Gwaiting)
			// This clears gcscanvalid.
			casgstatus(gp, _Gwaiting, _Grunning)
			gp.stackguard0 = gp.stack.lo + _StackGuard
			gogo(&gp.sched) // never return
		}

		// Act like goroutine called runtime.Gosched.
    // 开始抢占调度，将当前G放回队列，让M执行其他任务
		casgstatus(gp, _Gwaiting, _Grunning)
		gopreempt_m(gp) // never return
	}

	// Allocate a bigger segment and move the stack.
	oldsize := gp.stack.hi - gp.stack.lo
	newsize := oldsize * 2
	if newsize > maxstacksize {
		print("runtime: goroutine stack exceeds ", maxstacksize, "-byte limit\n")
		throw("stack overflow")
	}

	// The goroutine must be executing in order to call newstack,
	// so it must be Grunning (or Gscanrunning).
	casgstatus(gp, _Grunning, _Gcopystack)

	// The concurrent GC will not scan the stack while we are doing the copy since
	// the gp is in a Gcopystack status.
	copystack(gp, newsize, true)
	if stackDebug >= 1 {
		print("stack grow done\n")
	}
	casgstatus(gp, _Gcopystack, _Grunning)
	gogo(&gp.sched)
}
```

再看下`gopreempt_m`和`goschedImpl`的实现：

```go
func gopreempt_m(gp *g) {
	if trace.enabled {
		traceGoPreempt()
	}
	goschedImpl(gp)
}


func goschedImpl(gp *g) {
	status := readgstatus(gp)
	if status&^_Gscan != _Grunning {
		dumpgstatus(gp)
		throw("bad g status")
	}
	casgstatus(gp, _Grunning, _Grunnable)
	dropg()
	lock(&sched.lock)
	globrunqput(gp)
	unlock(&sched.lock)

	schedule()
}
```

