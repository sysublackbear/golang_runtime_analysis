# 9.通道



## 9.1.通道的结构

同步和异步的区别，在于是否有缓冲槽。

**chan.go**

```go
type hchan struct {
	qcount   uint           // 缓冲槽有效数据项数量
	dataqsiz uint           // 缓冲槽大小（可存储数据项数量）
	buf      unsafe.Pointer // 缓冲槽指针
	elemsize uint16         // 数据项大小
	closed   uint32
	elemtype *_type // 数据项类型
	sendx    uint   // 缓冲槽发送位置索引
	recvx    uint   // 缓冲槽接收位置索引
	recvq    waitq  // 接收者等待队列（由sudog组成的链表）
	sendq    waitq  // 发送者等待队列（由sudog组成的链表）

	// lock protects all fields in hchan, as well as several
	// fields in sudogs blocked on this channel.
	//
	// Do not change another G's status while holding this lock
	// (in particular, do not ready a G), as this can deadlock
	// with stack shrinking.
	lock mutex
}

type waitq struct {
	first *sudog
	last  *sudog
}


func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// 数据项不能超过64kb（这时候用指针更合适一些）
	if elem.size >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	if hchanSize%maxAlign != 0 || elem.align > maxAlign {
		throw("makechan: bad alignment")
	}

	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
  // 缓冲槽大小检查
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	// Hchan does not contain pointers interesting for GC when elements stored in buf do not contain pointers.
	// buf points into the same allocation, elemtype is persistent.
	// SudoG's are referenced from their owning thread so they can't be collected.
	// TODO(dvyukov,rlh): Rethink when collector can move allocated objects.
  // 受垃圾回收器限制，指针类型缓冲槽须单独分配内存
	var c *hchan
	switch {
	case mem == 0:
		// Queue or element size is zero.
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		// Race detector uses this location for synchronization.
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
		// Elements do not contain pointers.
		// Allocate hchan and buf in one call.
    // 因为缓冲槽大小固定，所以可一次性分配内存
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
    // 调整缓冲槽起始指针
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// Elements contain pointers.
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

  // 设置属性
	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.size, "; elemalg=", elem.alg, "; dataqsiz=", size, "\n")
	}
	return c
}
```





## 9.2.收发

 必须对channel收发双方（G）进行包装，因为它们要携带数据项，并存储相关状态。

**runtime2.go**

```go
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
	param          unsafe.Pointer // 传递唤醒参数
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


type sudog struct {
	// The following fields are protected by the hchan.lock of the
	// channel this sudog is blocking on. shrinkstack depends on
	// this for sudogs involved in channel ops.

	g *g

	// isSelect indicates g is participating in a select, so
	// g.selectDone must be CAS'd to win the wake-up race.
	isSelect bool
	next     *sudog
	prev     *sudog
	elem     unsafe.Pointer // 数据存储空间指针

	// The following fields are never accessed concurrently.
	// For channels, waitlink is only accessed by g.
	// For semaphores, all fields (including the ones above)
	// are only accessed when holding a semaRoot lock.

	acquiretime int64
	releasetime int64
	ticket      uint32
	parent      *sudog // semaRoot binary tree
	waitlink    *sudog // g.waiting list or semaRoot
	waittail    *sudog // semaRoot
	c           *hchan // channel
}

// sudog也实现了自己的二级缓存复用体系。
type p struct {
	id          int32
	status      uint32 // one of pidle/prunning/...
	link        puintptr
	schedtick   uint32     // incremented on every scheduler call
	syscalltick uint32     // incremented on every system call
	sysmontick  sysmontick // last tick observed by sysmon
	m           muintptr   // back-link to associated m (nil if idle)
	mcache      *mcache
	raceprocctx uintptr

	deferpool    [5][]*_defer // pool of available defer structs of different sizes (see panic.go)
	deferpoolbuf [5][32]*_defer

	// Cache of goroutine ids, amortizes accesses to runtime·sched.goidgen.
	goidcache    uint64
	goidcacheend uint64

	// Queue of runnable goroutines. Accessed without lock.
	runqhead uint32
	runqtail uint32
	runq     [256]guintptr
	// runnext, if non-nil, is a runnable G that was ready'd by
	// the current G and should be run next instead of what's in
	// runq if there's time remaining in the running G's time
	// slice. It will inherit the time left in the current time
	// slice. If a set of goroutines is locked in a
	// communicate-and-wait pattern, this schedules that set as a
	// unit and eliminates the (potentially large) scheduling
	// latency that otherwise arises from adding the ready'd
	// goroutines to the end of the run queue.
	runnext guintptr

	// Available G's (status == Gdead)
	gFree struct {
		gList
		n int32
	}

  sudogcache []*sudog   // 在procresize new(p)时指向 sudogbuf
	sudogbuf   [128]*sudog

	tracebuf traceBufPtr

	// traceSweep indicates the sweep events should be traced.
	// This is used to defer the sweep start event until a span
	// has actually been swept.
	traceSweep bool
	// traceSwept and traceReclaimed track the number of bytes
	// swept and reclaimed by sweeping in the current sweep loop.
	traceSwept, traceReclaimed uintptr

	palloc persistentAlloc // per-P to avoid mutex

	_ uint32 // Alignment for atomic fields below

	// Per-P GC state
	gcAssistTime         int64    // Nanoseconds in assistAlloc
	gcFractionalMarkTime int64    // Nanoseconds in fractional mark worker (atomic)
	gcBgMarkWorker       guintptr // (atomic)
	gcMarkWorkerMode     gcMarkWorkerMode

	// gcMarkWorkerStartTime is the nanotime() at which this mark
	// worker started.
	gcMarkWorkerStartTime int64

	// gcw is this P's GC work buffer cache. The work buffer is
	// filled by write barriers, drained by mutator assists, and
	// disposed on certain GC state transitions.
	gcw gcWork

	// wbBuf is this P's GC write barrier buffer.
	//
	// TODO: Consider caching this in the running G.
	wbBuf wbBuf

	runSafePointFn uint32 // if 1, run sched.safePointFn at next safe point

	pad cpu.CacheLinePad
}

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

	pidle      puintptr // idle p's
	npidle     uint32
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

收发函数实现细节如下：

**proc.go**

```go
//go:nosplit
// 新建或者复用现有的sudog
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
  
  // 如果本地缓存为空
	if len(pp.sudogcache) == 0 {
		lock(&sched.sudoglock)
		// First, try to grab a batch from central cache.
    // 从全局缓存转移一批到本地
		for len(pp.sudogcache) < cap(pp.sudogcache)/2 && sched.sudogcache != nil {
      // 一个个转移到本地（上面已经上锁了）
			s := sched.sudogcache
			sched.sudogcache = s.next
			s.next = nil
			pp.sudogcache = append(pp.sudogcache, s)
		}
		unlock(&sched.sudoglock)
		// If the central cache is empty, allocate a new one.
    // 如果失败，则新建
		if len(pp.sudogcache) == 0 {
			pp.sudogcache = append(pp.sudogcache, new(sudog))
		}
	}
  // 从尾部提取，并调整本地缓存
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

//go:nosplit
func releaseSudog(s *sudog) {
	if s.elem != nil {
		throw("runtime: sudog with non-nil elem")
	}
	if s.isSelect {
		throw("runtime: sudog with non-false isSelect")
	}
	if s.next != nil {
		throw("runtime: sudog with non-nil next")
	}
	if s.prev != nil {
		throw("runtime: sudog with non-nil prev")
	}
	if s.waitlink != nil {
		throw("runtime: sudog with non-nil waitlink")
	}
	if s.c != nil {
		throw("runtime: sudog with non-nil c")
	}
	gp := getg()
	if gp.param != nil {
		throw("runtime: releaseSudog with non-nil gp.param")
	}
	mp := acquirem() // avoid rescheduling to another P
	pp := mp.p.ptr()
  // 如果本地缓存已满
	if len(pp.sudogcache) == cap(pp.sudogcache) {
		// Transfer half of local cache to the central cache.
    // 转移一半到全局
		var first, last *sudog
		for len(pp.sudogcache) > cap(pp.sudogcache)/2 {
			n := len(pp.sudogcache)
			p := pp.sudogcache[n-1]
			pp.sudogcache[n-1] = nil
			pp.sudogcache = pp.sudogcache[:n-1]
			if first == nil {
				first = p
			} else {
				last.next = p
			}
			last = p
		}
    // 将提取的链表挂到全局
		lock(&sched.sudoglock)
		last.next = sched.sudogcache
		sched.sudogcache = first
		unlock(&sched.sudoglock)
	}
	pp.sudogcache = append(pp.sudogcache, s)
	releasem(mp)
}
```



### 9.2.1.同步收发算法

同步模式的关键是找到匹配的接收或发送方，找到则直接拷贝数据；找不到就将自身打包后放入等待队列，由另一方复制数据并唤醒。

在同步模式下，channel的作用仅是维护发送和接收者队列，数据复制与channel无关。另外在唤醒后，需要验证唤醒者身份，以此决定是否有实际的数据传递。

**chan.go**

**注**：由于同步收发和异步收发的代码都写在一起了，所以对于异步收发的算法，依然从下面这段逻辑开始分析。

```go
// entry point for c <- x from compiled code
//go:nosplit
func chansend1(c *hchan, elem unsafe.Pointer) {
	chansend(c, elem, true, getcallerpc())
}

// 参数eq是数据项指针
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	if c == nil {
		if !block {
			return false
		}
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	if debugChan {
		print("chansend: chan=", c, "\n")
	}

	if raceenabled {
		racereadpc(c.raceaddr(), callerpc, funcPC(chansend))
	}

	// Fast path: check for failed non-blocking operation without acquiring the lock.
	//
	// After observing that the channel is not closed, we observe that the channel is
	// not ready for sending. Each of these observations is a single word-sized read
	// (first c.closed and second c.recvq.first or c.qcount depending on kind of channel).
	// Because a closed channel cannot transition from 'ready for sending' to
	// 'not ready for sending', even if the channel is closed between the two observations,
	// they imply a moment between the two when the channel was both not yet closed
	// and not ready for sending. We behave as if we observed the channel at that moment,
	// and report that the send cannot proceed.
	//
	// It is okay if the reads are reordered here: if we observe that the channel is not
	// ready for sending and then observe that it is not closed, that implies that the
	// channel wasn't closed during the first observation.
  // 1.block是false
  // 2.管道是否被关闭（c.closed == 0)
  // 3.缓冲槽大小为0，且并没有接收者（不能重复投递？因为前面已经有一个阻塞在排队？）
  // 4.缓冲槽大小大于0，且管道已经满了
	if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
		(c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
		return false
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)

	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

  // 从等待队列获取接收者
	if sg := c.recvq.dequeue(); sg != nil {
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)   // 同步发送的逻辑
		return true
	}

  // 缓冲槽未满，先将数据复制到缓冲槽
	if c.qcount < c.dataqsiz {
		// Space is available in the channel buffer. Enqueue the element to send.
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
    // 将数据复制到缓冲槽
		typedmemmove(c.elemtype, qp, ep)
    
    // 调整缓冲槽队列索引和数据项计数
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0  // 唤醒队列
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}

	if !block {
		unlock(&c.lock)
		return false
	}

	// Block on the channel. Some receiver will complete our operation for us.
  // 如果没有接收者，则打包成sudog
	gp := getg()
	mysg := acquireSudog()  // 新建，或从缓存获取复用sudog对象
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
  
  // 将发送sudog放入等待队列，休眠，等待被接收者唤醒
	c.sendq.enqueue(mysg)
	goparkunlock(&c.lock, waitReasonChanSend, traceEvGoBlockSend, 3)
	// Ensure the value being sent is kept alive until the
	// receiver copies it out. The sudog has a pointer to the
	// stack object, but sudogs aren't considered as roots of the
	// stack tracer.
	KeepAlive(ep)

	// someone woke us up.
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
  // 被唤醒，检查是否被closechan唤醒
  // 此时数据已被接收者复制，无须再做处理
	gp.waiting = nil
	if gp.param == nil {
		if c.closed == 0 {  // 检查是否被closechan唤醒
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
  // 将sudog放回复用缓存
	releaseSudog(mysg)
	return true
}


func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if raceenabled {
		if c.dataqsiz == 0 {
			racesync(c, sg)
		} else {
			// Pretend we go through the buffer, even though
			// we copy directly. Note that we need to increment
			// the head/tail locations only when raceenabled.
			qp := chanbuf(c, c.recvx)
			raceacquire(qp)
			racerelease(qp)
			raceacquireg(sg.g, qp)
			racereleaseg(sg.g, qp)
			c.recvx++
			if c.recvx == c.dataqsiz {
				c.recvx = 0
			}
			c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
		}
	}
  
  // 主体逻辑从下面开始
	if sg.elem != nil {
		sendDirect(c.elemtype, sg, ep)  // 直接用memmove将数据项复制给接收者
		sg.elem = nil
	}
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
  // 唤醒接收者
	goready(gp, skip+1)
}


func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
	// src is on our stack, dst is a slot on another stack.

	// Once we read sg.elem out of sg, it will no longer
	// be updated if the destination's stack gets copied (shrunk).
	// So make sure that no preemption points can happen between read & use.
  // 直接进行内存"搬迁"
  // 如果目标地址的栈发生了栈收缩，当我们读出了 sg.elem 后
  // 就不能修改真正的 dst 位置的值了
  // 因此需要在读和写之前加上一个屏障
	dst := sg.elem
	typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)
	// No need for cgo write barrier checks because dst is always
	// Go memory.
	memmove(dst, src, t.size)
}


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
	if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 {
		wakep()
	}
	releasem(mp)
}
```

其中`waitq`的入队和出队逻辑如下：

```go
// 直接塞到队尾
func (q *waitq) enqueue(sgp *sudog) {
	sgp.next = nil
	x := q.last
	if x == nil {
		sgp.prev = nil
		q.first = sgp
		q.last = sgp
		return
	}
	sgp.prev = x
	x.next = sgp
	q.last = sgp
}

func (q *waitq) dequeue() *sudog {
	for {
		sgp := q.first
		if sgp == nil {
			return nil
		}
		y := sgp.next
		if y == nil {
			q.first = nil
			q.last = nil
		} else {
			y.prev = nil
			q.first = y
			sgp.next = nil // mark as removed (see dequeueSudog)
		}

		// if a goroutine was put on this queue because of a
		// select, there is a small window between the goroutine
		// being woken up by a different case and it grabbing the
		// channel locks. Once it has the lock
		// it removes itself from the queue, so we won't see it after that.
		// We use a flag in the G struct to tell us when someone
		// else has won the race to signal this goroutine but the goroutine
		// hasn't removed itself from the queue yet.
    // 如果这个sudog被select选中，则跳过
		if sgp.isSelect && !atomic.Cas(&sgp.g.selectDone, 0, 1) {
			continue
		}

		return sgp
	}
}
```







接收代码和发送几乎一致，差别在于谁先进入等待队列，谁负责唤醒。编译器会将不同语法翻译成不同的函数调用。

**chan.go**

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

func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	// raceenabled: don't need to check ep, as it is always on the stack
	// or is new memory allocated by reflect.

	if debugChan {
		print("chanrecv: chan=", c, "\n")
	}

  // 管道为空的异常
	if c == nil {
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	// Fast path: check for failed non-blocking operation without acquiring the lock.
	//
	// After observing that the channel is not ready for receiving, we observe that the
	// channel is not closed. Each of these observations is a single word-sized read
	// (first c.sendq.first or c.qcount, and second c.closed).
	// Because a channel cannot be reopened, the later observation of the channel
	// being not closed implies that it was also not closed at the moment of the
	// first observation. We behave as if we observed the channel at that moment
	// and report that the receive cannot proceed.
	//
	// The order of operations is important here: reversing the operations can lead to
	// incorrect behavior when racing with a close.
  // 1.block为false
  // 2.缓冲槽大小为空，且发送者等待队列为空
	if !block && (c.dataqsiz == 0 && c.sendq.first == nil ||
		c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0) &&
		atomic.Load(&c.closed) == 0 {
		return
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)

  // 管道未被关闭，且有效数据项为0
	if c.closed != 0 && c.qcount == 0 {
		if raceenabled {
			raceacquire(c.raceaddr())
		}
		unlock(&c.lock)
		if ep != nil {
			// 缓冲槽清0
      typedmemclr(c.elemtype, ep)
		}
		return true, false
	}

  // 从等待队列获取发送者
	if sg := c.sendq.dequeue(); sg != nil {
		// Found a waiting sender. If buffer is size 0, receive value
		// directly from sender. Otherwise, receive from head of queue
		// and add sender's value to the tail of the queue (both map to
		// the same buffer slot because the queue is full).
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)  // 同步接收逻辑
		return true, true
	}

	if c.qcount > 0 {
		// Receive directly from queue
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
    // 清零，调整缓冲槽队列索引及计数
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

	if !block {
		unlock(&c.lock)
		return false, false
	}

	// no sender available: block on this channel.
  // 如果没有发送者，打包成sudog
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
  
  // 放入等待队列，休眠，等待被发送者唤醒
	c.recvq.enqueue(mysg)
	goparkunlock(&c.lock, waitReasonChanReceive, traceEvGoBlockRecv, 3)

	// someone woke us up
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
  
  // 被唤醒
  // 数据已被发送者复制过来
	gp.waiting = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	closed := gp.param == nil
	gp.param = nil
	mysg.c = nil
  
  // 将sudog放回复用缓存
	releaseSudog(mysg)
	return true, !closed
}


func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if c.dataqsiz == 0 {  // 同步管道，直接从缓冲区复制数据
		if raceenabled {
			racesync(c, sg)
		}
		if ep != nil {
			// copy data from sender
      // 从发送者复制数据
			recvDirect(c.elemtype, sg, ep)
		}
	} else {
		// Queue is full. Take the item at the
		// head of the queue. Make the sender enqueue
		// its item at the tail of the queue. Since the
		// queue is full, those are both the same slot.
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
			raceacquireg(sg.g, qp)
			racereleaseg(sg.g, qp)
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
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
	goready(gp, skip+1)
}

func recvDirect(t *_type, sg *sudog, dst unsafe.Pointer) {
	// dst is on our stack or the heap, src is on another stack.
	// The channel is locked, so src will not move during this
	// operation.
	src := sg.elem
	typeBitsBulkBarrier(t, uintptr(dst), uintptr(src), t.size)
  // 从缓冲槽复制数据项
	memmove(dst, src, t.size)
}
```



### 9.2.2.异步收发算法

异步模式围绕缓冲槽进行。当有空位时，发送者向槽中复制数据；有数据后，接收者从槽中获取数据。双方都有唤醒排队的另一方继续工作的责任。

**chan.go**

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	if c == nil {
		if !block {
			return false
		}
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	if debugChan {
		print("chansend: chan=", c, "\n")
	}

	if raceenabled {
		racereadpc(c.raceaddr(), callerpc, funcPC(chansend))
	}

	// Fast path: check for failed non-blocking operation without acquiring the lock.
	//
	// After observing that the channel is not closed, we observe that the channel is
	// not ready for sending. Each of these observations is a single word-sized read
	// (first c.closed and second c.recvq.first or c.qcount depending on kind of channel).
	// Because a closed channel cannot transition from 'ready for sending' to
	// 'not ready for sending', even if the channel is closed between the two observations,
	// they imply a moment between the two when the channel was both not yet closed
	// and not ready for sending. We behave as if we observed the channel at that moment,
	// and report that the send cannot proceed.
	//
	// It is okay if the reads are reordered here: if we observe that the channel is not
	// ready for sending and then observe that it is not closed, that implies that the
	// channel wasn't closed during the first observation.
	if !block && c.closed == 0 && ((c.dataqsiz == 0 && c.recvq.first == nil) ||
		(c.dataqsiz > 0 && c.qcount == c.dataqsiz)) {
		return false
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)

	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

  // 现在缓冲槽不为空，唤醒某个排队的接收者从槽中获取数据
	if sg := c.recvq.dequeue(); sg != nil {
		// Found a waiting receiver. We pass the value we want to send
		// directly to the receiver, bypassing the channel buffer (if any).
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

	if c.qcount < c.dataqsiz {
		// Space is available in the channel buffer. Enqueue the element to send.
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
    // 将数据复制到缓冲槽
		typedmemmove(c.elemtype, qp, ep)
    
    // 调整缓冲槽队列索引和数据项计数
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}

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
  // 打包成sudog
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
  
  // 放入发送者等待队列，休眠。等待有空位时被唤醒。
	c.sendq.enqueue(mysg)
	goparkunlock(&c.lock, waitReasonChanSend, traceEvGoBlockSend, 3)
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
	if gp.param == nil {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
  
  // 将sudog放回复用缓存
	releaseSudog(mysg)
	return true
}
```

发送须有缓冲槽空位，而接收则需要槽中有可用数据项。

**chan.go**

```go
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

	// Fast path: check for failed non-blocking operation without acquiring the lock.
	//
	// After observing that the channel is not ready for receiving, we observe that the
	// channel is not closed. Each of these observations is a single word-sized read
	// (first c.sendq.first or c.qcount, and second c.closed).
	// Because a channel cannot be reopened, the later observation of the channel
	// being not closed implies that it was also not closed at the moment of the
	// first observation. We behave as if we observed the channel at that moment
	// and report that the receive cannot proceed.
	//
	// The order of operations is important here: reversing the operations can lead to
	// incorrect behavior when racing with a close.
	if !block && (c.dataqsiz == 0 && c.sendq.first == nil ||
		c.dataqsiz > 0 && atomic.Loaduint(&c.qcount) == 0) &&
		atomic.Load(&c.closed) == 0 {
		return
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)

	if c.closed != 0 && c.qcount == 0 {
		if raceenabled {
			raceacquire(c.raceaddr())
		}
		unlock(&c.lock)
		if ep != nil {
			typedmemclr(c.elemtype, ep)
		}
		return true, false
	}

  // 现在有空位了，唤醒某个排队的发送者向槽中发送数据
	if sg := c.sendq.dequeue(); sg != nil {
		// Found a waiting sender. If buffer is size 0, receive value
		// directly from sender. Otherwise, receive from head of queue
		// and add sender's value to the tail of the queue (both map to
		// the same buffer slot because the queue is full).
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

	if c.qcount > 0 {
		// Receive directly from queue
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
    // 从缓冲槽复制数据项
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
    
    // 清零。调整缓冲槽队列索引及计数
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

	if !block {
		unlock(&c.lock)
		return false, false
	}

	// no sender available: block on this channel.
  // 打包成sudog
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
  
  // 放入接收等待队列，休眠。等待有数据项时被唤醒。
	c.recvq.enqueue(mysg)
	goparkunlock(&c.lock, waitReasonChanReceive, traceEvGoBlockRecv, 3)

	// someone woke us up
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
  
  // 将sudog放回复用缓存
	releaseSudog(mysg)
	return true, !closed
}
```

示意图如下：

![image-20191116190244597](/Users/dengzhuowen/Library/Application Support/typora-user-images/image-20191116190244597.png)

**注意：**`chansend`和`chanrecv`都需要对管道进行加锁。





### 9.2.3.关闭管道

关闭操作将所有排队者唤醒，并通过`chan.closed`，`g.param`参数告知由`close`发出。

+ 向`closed channel`发送数据，触发`panic`；
+ 从`closed channel`读取数据，返回零值；
+ 无论收发，`nil channel`都会阻塞。

**chan.go**

```go
func closechan(c *hchan) {
	if c == nil {
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock)
  // 不能重复关闭
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}

	if raceenabled {
		callerpc := getcallerpc()
		racewritepc(c.raceaddr(), callerpc, funcPC(closechan))
		racerelease(c.raceaddr())
	}

  // 设置关闭标志
	c.closed = 1

	var glist gList

	// release all readers
  // 释放所有接收者
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
    // 这个参数表明唤醒者是closechan
		gp.param = nil
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}

	// release all writers (they will panic)
  // 释放所有发送者
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
    // closechan唤醒
		gp.param = nil
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



### 9.2.4.选择

选择模式(select)是从多个channel里随机选出可用的那个，编译器会将相关语句翻译成具体的函数调用。

我们测试下：

**test.go**

```go
package main

import "fmt"

func main() {
    c1, c2 := make(chan int), make(chan int, 2)

    select {
    case c1 <- 1:
        fmt.Println(0x11)
    case <- c2:
        fmt.Println(0x22)
    default:
        fmt.Println(0xff)
    }
}
```

然后进行反汇编：

```bash
[sysublackbear@centos-linux zhuowen.deng]$ go build -o test test.go
[sysublackbear@centos-linux zhuowen.deng]$ go tool objdump -s "main\.main" test
TEXT main.main(SB) /home/sysublackbear/go/src/zhuowen.deng/test.go
  test.go:5		0x4841b0		64488b0c25f8ffffff	MOVQ FS:0xfffffff8, CX
  test.go:5		0x4841b9		488d842448ffffff	LEAQ 0xffffff48(SP), AX
  test.go:5		0x4841c1		483b4110		CMPQ 0x10(CX), AX
  test.go:5		0x4841c5		0f8605020000		JBE 0x4843d0
  test.go:5		0x4841cb		4881ec38010000		SUBQ $0x138, SP
  test.go:5		0x4841d2		4889ac2430010000	MOVQ BP, 0x130(SP)
  test.go:5		0x4841da		488dac2430010000	LEAQ 0x130(SP), BP
  test.go:6		0x4841e2		488d0537fe0000		LEAQ 0xfe37(IP), AX
  test.go:6		0x4841e9		48890424		MOVQ AX, 0(SP)
  test.go:6		0x4841ed		48c744240800000000	MOVQ $0x0, 0x8(SP)
  test.go:6		0x4841f6		e845eef7ff		CALL runtime.makechan(SB)  // make(c1)
  test.go:6		0x4841fb		488b442410		MOVQ 0x10(SP), AX
  test.go:6		0x484200		4889442440		MOVQ AX, 0x40(SP)
  test.go:6		0x484205		488d0d14fe0000		LEAQ 0xfe14(IP), CX
  test.go:6		0x48420c		48890c24		MOVQ CX, 0(SP)
  test.go:6		0x484210		48c744240802000000	MOVQ $0x2, 0x8(SP)
  test.go:6		0x484219		e822eef7ff		CALL runtime.makechan(SB)  // make(c2)
  test.go:6		0x48421e		488b442410		MOVQ 0x10(SP), AX
  test.go:6		0x484223		4889442438		MOVQ AX, 0x38(SP)
  test.go:9		0x484228		48c744243001000000	MOVQ $0x1, 0x30(SP)
  test.go:8		0x484231		48c744247800000000	MOVQ $0x0, 0x78(SP)
  test.go:8		0x48423a		488dbc2480000000	LEAQ 0x80(SP), DI
  test.go:8		0x484242		0f57c0			XORPS X0, X0
  test.go:8		0x484245		488d7ff0		LEAQ -0x10(DI), DI
  test.go:8		0x484249		48896c24f0		MOVQ BP, -0x10(SP)
  test.go:8		0x48424e		488d6c24f0		LEAQ -0x10(SP), BP
  test.go:8		0x484253		e822c3fcff		CALL 0x45057a
  test.go:8		0x484258		488b6d00		MOVQ 0(BP), BP
  test.go:8		0x48425c		488d4c2478		LEAQ 0x78(SP), CX
  test.go:8		0x484261		48890c24		MOVQ CX, 0(SP)
  test.go:8		0x484265		48c7442408b8000000	MOVQ $0xb8, 0x8(SP)
  test.go:8		0x48426e		c744241003000000	MOVL $0x3, 0x10(SP)
  test.go:8		0x484276		e8c515fbff		CALL runtime.newselect(SB)  // newselect
  test.go:9		0x48427b		488d442478		LEAQ 0x78(SP), AX
  test.go:9		0x484280		48890424		MOVQ AX, 0(SP)
  test.go:9		0x484284		488b4c2440		MOVQ 0x40(SP), CX
  test.go:9		0x484289		48894c2408		MOVQ CX, 0x8(SP)
  test.go:9		0x48428e		488d4c2430		LEAQ 0x30(SP), CX
  test.go:9		0x484293		48894c2410		MOVQ CX, 0x10(SP)
  test.go:9		0x484298		e8d316fbff		CALL runtime.selectsend(SB)  // case c1
  test.go:11		0x48429d		488d442478		LEAQ 0x78(SP), AX
  test.go:11		0x4842a2		48890424		MOVQ AX, 0(SP)
  test.go:11		0x4842a6		488b4c2438		MOVQ 0x38(SP), CX
  test.go:11		0x4842ab		48894c2408		MOVQ CX, 0x8(SP)
  test.go:11		0x4842b0		0f57c0			XORPS X0, X0
  test.go:11		0x4842b3		0f11442410		MOVUPS X0, 0x10(SP)
  test.go:11		0x4842b8		e8a317fbff		CALL runtime.selectrecv(SB)  // case c2
  test.go:13		0x4842bd		488d442478		LEAQ 0x78(SP), AX
  test.go:13		0x4842c2		48890424		MOVQ AX, 0(SP)
  test.go:13		0x4842c6		e8a518fbff		CALL runtime.selectdefault(SB)  // case default
  test.go:8		0x4842cb		488d442478		LEAQ 0x78(SP), AX
  test.go:8		0x4842d0		48890424		MOVQ AX, 0(SP)
  test.go:8		0x4842d4		e8971bfbff		CALL runtime.selectgo(SB)  // selectgo
  test.go:8		0x4842d9		488b442408		MOVQ 0x8(SP), AX
  test.go:9		0x4842de		4885c0			TESTQ AX, AX
  test.go:9		0x4842e1		0f84a2000000		JE 0x484389
  test.go:11		0x4842e7		4883f801		CMPQ $0x1, AX
  test.go:11		0x4842eb		745a			JE 0x484347
  test.go:13		0x4842ed		4883f802		CMPQ $0x2, AX
  test.go:13		0x4842f1		0f85d7000000		JNE 0x4843ce
  test.go:14		0x4842f7		0f57c0			XORPS X0, X0
  test.go:14		0x4842fa		0f11442448		MOVUPS X0, 0x48(SP)
  test.go:14		0x4842ff		488d05da010100		LEAQ 0x101da(IP), AX
  test.go:14		0x484306		4889442448		MOVQ AX, 0x48(SP)
  test.go:14		0x48430b		488d0556190400		LEAQ main.statictmp_2(SB), AX
  test.go:14		0x484312		4889442450		MOVQ AX, 0x50(SP)
  test.go:14		0x484317		488d442448		LEAQ 0x48(SP), AX
  test.go:14		0x48431c		48890424		MOVQ AX, 0(SP)
  test.go:14		0x484320		48c744240801000000	MOVQ $0x1, 0x8(SP)
  test.go:14		0x484329		48c744241001000000	MOVQ $0x1, 0x10(SP)
  test.go:14		0x484332		e8b99cffff		CALL fmt.Println(SB)
  test.go:14		0x484337		488bac2430010000	MOVQ 0x130(SP), BP
  test.go:14		0x48433f		4881c438010000		ADDQ $0x138, SP
  test.go:14		0x484346		c3			RET
  test.go:12		0x484347		0f57c0			XORPS X0, X0
  test.go:12		0x48434a		0f11442458		MOVUPS X0, 0x58(SP)
  test.go:12		0x48434f		488d058a010100		LEAQ 0x1018a(IP), AX
  test.go:12		0x484356		4889442458		MOVQ AX, 0x58(SP)
  test.go:12		0x48435b		488d05fe180400		LEAQ main.statictmp_1(SB), AX
  test.go:12		0x484362		4889442460		MOVQ AX, 0x60(SP)
  test.go:12		0x484367		488d442458		LEAQ 0x58(SP), AX
  test.go:12		0x48436c		48890424		MOVQ AX, 0(SP)
  test.go:12		0x484370		48c744240801000000	MOVQ $0x1, 0x8(SP)
  test.go:12		0x484379		48c744241001000000	MOVQ $0x1, 0x10(SP)
  test.go:12		0x484382		e8699cffff		CALL fmt.Println(SB)
  test.go:11		0x484387		ebae			JMP 0x484337
  test.go:10		0x484389		0f57c0			XORPS X0, X0
  test.go:10		0x48438c		0f11442468		MOVUPS X0, 0x68(SP)
  test.go:10		0x484391		488d0548010100		LEAQ 0x10148(IP), AX
  test.go:10		0x484398		4889442468		MOVQ AX, 0x68(SP)
  test.go:10		0x48439d		488d05b4180400		LEAQ main.statictmp_0(SB), AX
  test.go:10		0x4843a4		4889442470		MOVQ AX, 0x70(SP)
  test.go:10		0x4843a9		488d442468		LEAQ 0x68(SP), AX
  test.go:10		0x4843ae		48890424		MOVQ AX, 0(SP)
  test.go:10		0x4843b2		48c744240801000000	MOVQ $0x1, 0x8(SP)
  test.go:10		0x4843bb		48c744241001000000	MOVQ $0x1, 0x10(SP)
  test.go:10		0x4843c4		e8279cffff		CALL fmt.Println(SB)
  test.go:9		0x4843c9		e969ffffff		JMP 0x484337
  test.go:9		0x4843ce		0f0b			UD2
  test.go:5		0x4843d0		e86b9cfcff		CALL runtime.morestack_noctxt(SB)
  test.go:5		0x4843d5		e9d6fdffff		JMP main.main(SB)
```



完整的select对象由"header + [n]scase"组成，完全是C不定长结构体的风格。

**select.go**

```go
type scase struct {
	c           *hchan         // chan
	elem        unsafe.Pointer // data element
	kind        uint16
	pc          uintptr // race pc (for race detector / msan)
	releasetime int64
}

type runtimeSelect struct {
	dir selectDir
	typ unsafe.Pointer // channel type (not used here)
	ch  *hchan         // channel
	val unsafe.Pointer // ptr to data (SendDir) or ptr to receive buffer (RecvDir)
}
```

在处理好`select`对象后，还需要初始化`scase`。过程并不复杂，依ncase确定位置，设置相关参数。依照`case channel`操作方式，可分为`send`，`recv`和`default`三种。

```go
//go:linkname reflect_rselect reflect.rselect
func reflect_rselect(cases []runtimeSelect) (int, bool) {
	if len(cases) == 0 {
		block()
	}
	sel := make([]scase, len(cases))
	order := make([]uint16, 2*len(cases))
	for i := range cases {
		rc := &cases[i]
		switch rc.dir {
		case selectDefault:
			sel[i] = scase{kind: caseDefault}
		case selectSend:
			sel[i] = scase{kind: caseSend, c: rc.ch, elem: rc.val}
		case selectRecv:
			sel[i] = scase{kind: caseRecv, c: rc.ch, elem: rc.val}
		}
		if raceenabled || msanenabled {
			selectsetpc(&sel[i])
		}
	}

	return selectgo(&sel[0], &order[0], len(cases))
}
```

选择算法的逻辑比较繁琐，可以慢慢看。

**select.go**

```go
func selectgo(cas0 *scase, order0 *uint16, ncases int) (int, bool) {
	if debugSelect {
		print("select: cas0=", cas0, "\n")
	}

  // 为访问方便，将scase封装成slice
	cas1 := (*[1 << 16]scase)(unsafe.Pointer(cas0))
	order1 := (*[1 << 17]uint16)(unsafe.Pointer(order0))

  
  // pollorder: 对scases序号洗牌，乱序
  // lockorder: 按channel地址顺序排序，select里面所有的channel列表
	scases := cas1[:ncases:ncases]
	pollorder := order1[:ncases:ncases]
	lockorder := order1[ncases:][:ncases:ncases]

	// Replace send/receive cases involving nil channels with
	// caseNil so logic below can assume non-nil channel.
	for i := range scases {
		cas := &scases[i]
		if cas.c == nil && cas.kind != caseDefault {
			*cas = scase{}
		}
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
		for i := 0; i < ncases; i++ {
			scases[i].releasetime = -1
		}
	}

	// The compiler rewrites selects that statically have
	// only 0 or 1 cases plus default into simpler constructs.
	// The only way we can end up with such small sel.ncase
	// values here is for a larger select in which most channels
	// have been nilled out. The general code handles those
	// cases correctly, and they are rare enough not to bother
	// optimizing (and needing to test).

	// generate permuted order
  // 对序号洗牌，乱序
	for i := 1; i < ncases; i++ {
		j := fastrandn(uint32(i + 1))
		pollorder[i] = pollorder[j]
		pollorder[j] = uint16(i)
	}

	// sort the cases by Hchan address to get the locking order.
	// simple heap sort, to guarantee n log n time and constant stack footprint.
	for i := 0; i < ncases; i++ {
		j := i
		// Start with the pollorder to permute cases on the same channel.
		c := scases[pollorder[i]].c
		for j > 0 && scases[lockorder[(j-1)/2]].c.sortkey() < c.sortkey() {
			k := (j - 1) / 2
			lockorder[j] = lockorder[k]
			j = k
		}
		lockorder[j] = pollorder[i]
	}
	for i := ncases - 1; i >= 0; i-- {
		o := lockorder[i]
		c := scases[o].c
		lockorder[i] = lockorder[0]
		j := 0
		for {
			k := j*2 + 1
			if k >= i {
				break
			}
			if k+1 < i && scases[lockorder[k]].c.sortkey() < scases[lockorder[k+1]].c.sortkey() {
				k++
			}
			if c.sortkey() < scases[lockorder[k]].c.sortkey() {
				lockorder[j] = lockorder[k]
				j = k
				continue
			}
			break
		}
		lockorder[j] = o
	}

	if debugSelect {
		for i := 0; i+1 < ncases; i++ {
			if scases[lockorder[i]].c.sortkey() > scases[lockorder[i+1]].c.sortkey() {
				print("i=", i, " x=", lockorder[i], " y=", lockorder[i+1], "\n")
				throw("select: broken sort")
			}
		}
	}

	// lock all the channels involved in the select
  // 锁定全部channel
	sellock(scases, lockorder)

	var (
		gp     *g
		sg     *sudog
		c      *hchan
		k      *scase
		sglist *sudog
		sgnext *sudog
		qp     unsafe.Pointer
		nextp  **sudog
	)

loop:
  
  // 1: 查找已准备好的case
	// pass 1 - look for something already waiting
	var dfli int
	var dfl *scase
	var casi int
	var cas *scase
	var recvOK bool
	for i := 0; i < ncases; i++ {
    // 从乱序的pollorder中获取，这就是select随机选择的关键
		casi = int(pollorder[i])
		cas = &scases[casi]
		c = cas.c

		switch cas.kind {
		case caseNil:
			continue

		case caseRecv:
			sg = c.sendq.dequeue()
			if sg != nil {  // 有发送者
				goto recv  // 同步
			}
			if c.qcount > 0 {  // 缓冲槽有数据
				goto bufrecv  // 异步
			}
			if c.closed != 0 {  // 关闭
				goto rclose
			}

		case caseSend:
			if raceenabled {
				racereadpc(c.raceaddr(), cas.pc, chansendpc)
			}
			if c.closed != 0 {
				goto sclose
			}
			sg = c.recvq.dequeue()
			if sg != nil {
				goto send
			}
			if c.qcount < c.dataqsiz {
				goto bufsend
			}

		case caseDefault:
			dfli = casi
			dfl = cas
		}
	}

  // 如果没有准备好的case，尝试执行default
	if dfl != nil {
		selunlock(scases, lockorder)
		casi = dfli
		cas = dfl
		goto retc
	}

  // 2: 如果没有任何准备好的case，将当前select G打包成sudog，放到所有 channel 排队列表，等待唤醒
	// pass 2 - enqueue on all chans
	gp = getg()
	if gp.waiting != nil {
		throw("gp.waiting != nil")
	}
	nextp = &gp.waiting
	for _, casei := range lockorder {
		casi = int(casei)
		cas = &scases[casi]
		if cas.kind == caseNil {
			continue
		}
		c = cas.c
    // 打包成sudog
    // 每个case的sudog都不同
		sg := acquireSudog()
		sg.g = gp
		sg.isSelect = true
		// No stack splits between assigning elem and enqueuing
		// sg on gp.waiting where copystack can find it.
		sg.elem = cas.elem
		sg.releasetime = 0
		if t0 != 0 {
			sg.releasetime = -1
		}
		sg.c = c
		// Construct waiting list in lock order.
    // 全部sudog被放入gp.waiting链表
    // 此链表顺序同pollorder，后面以此识别是哪个case唤醒的。
		*nextp = sg
		nextp = &sg.waitlink

    // 根据case类型，决定放入发送或接收者排队列表
		switch cas.kind {
		case caseRecv:
			c.recvq.enqueue(sg)

		case caseSend:
			c.sendq.enqueue(sg)
		}
	}

	// wait for someone to wake us up
  // 休眠select G,直到某个case channel活动后，从排队列表将其提取并唤醒
	gp.param = nil
	gopark(selparkcommit, nil, waitReasonSelect, traceEvGoBlockSelect, 1)

  // 被唤醒
	sellock(scases, lockorder)

	gp.selectDone = 0
	sg = (*sudog)(gp.param)
	gp.param = nil

	// pass 3 - dequeue from unsuccessful chans
	// otherwise they stack up on quiet channels
	// record the successful case, if any.
	// We singly-linked up the SudoGs in lock order.
  // 3.找出是哪个case唤醒select G的
	casi = -1
	cas = nil
  // 使用第二步准备好的gp.waiting链表
	sglist = gp.waiting
	// Clear all elem before unlinking from gp.waiting.
	for sg1 := gp.waiting; sg1 != nil; sg1 = sg1.waitlink {
		sg1.isSelect = false
		sg1.elem = nil
		sg1.c = nil
	}
	gp.waiting = nil

	for _, casei := range lockorder {
    // 同样使用pollorder，所以和gp.waiting顺序一致
		k = &scases[casei]
		if k.kind == caseNil {
			continue
		}
		if sglist.releasetime > 0 {
			k.releasetime = sglist.releasetime
		}
		if sg == sglist {
			// sg has already been dequeued by the G that woke us up.
      // 匹配
			casi = int(casei)
			cas = k
		} else {
      // 不匹配，将sudog从channel排队列表移除
			c = k.c
			if k.kind == caseSend {
				c.sendq.dequeueSudoG(sglist)
			} else {
				c.recvq.dequeueSudoG(sglist)
			}
		}
    // 利用循环清理掉所有排队sudog
		sgnext = sglist.waitlink
		sglist.waitlink = nil
		releaseSudog(sglist)
		sglist = sgnext
	}

  // 没找到匹配，可能被意外唤醒，重新开始
	if cas == nil {
		// We can wake up with gp.param == nil (so cas == nil)
		// when a channel involved in the select has been closed.
		// It is easiest to loop and re-run the operation;
		// we'll see that it's now closed.
		// Maybe some day we can signal the close explicitly,
		// but we'd have to distinguish close-on-reader from close-on-writer.
		// It's easiest not to duplicate the code and just recheck above.
		// We know that something closed, and things never un-close,
		// so we won't block again.
		goto loop
	}

	c = cas.c

	if debugSelect {
		print("wait-return: cas0=", cas0, " c=", c, " cas=", cas, " kind=", cas.kind, "\n")
	}

	if cas.kind == caseRecv {
		recvOK = true
	}

	if raceenabled {
		if cas.kind == caseRecv && cas.elem != nil {
			raceWriteObjectPC(c.elemtype, cas.elem, cas.pc, chanrecvpc)
		} else if cas.kind == caseSend {
			raceReadObjectPC(c.elemtype, cas.elem, cas.pc, chansendpc)
		}
	}
	if msanenabled {
		if cas.kind == caseRecv && cas.elem != nil {
			msanwrite(cas.elem, c.elemtype.size)
		} else if cas.kind == caseSend {
			msanread(cas.elem, c.elemtype.size)
		}
	}

  // 找到目标，解锁，退出
	selunlock(scases, lockorder)
	goto retc

// 异步接收
bufrecv:
	// can receive from buffer
	if raceenabled {
		if cas.elem != nil {
			raceWriteObjectPC(c.elemtype, cas.elem, cas.pc, chanrecvpc)
		}
		raceacquire(chanbuf(c, c.recvx))
		racerelease(chanbuf(c, c.recvx))
	}
	if msanenabled && cas.elem != nil {
		msanwrite(cas.elem, c.elemtype.size)
	}
	recvOK = true
	qp = chanbuf(c, c.recvx)
	if cas.elem != nil {
		typedmemmove(c.elemtype, cas.elem, qp)
	}
	typedmemclr(c.elemtype, qp)
	c.recvx++
	if c.recvx == c.dataqsiz {
		c.recvx = 0
	}
	c.qcount--
	selunlock(scases, lockorder)
	goto retc

// 异步发送
bufsend:
	// can send to buffer
	if raceenabled {
		raceacquire(chanbuf(c, c.sendx))
		racerelease(chanbuf(c, c.sendx))
		raceReadObjectPC(c.elemtype, cas.elem, cas.pc, chansendpc)
	}
	if msanenabled {
		msanread(cas.elem, c.elemtype.size)
	}
	typedmemmove(c.elemtype, chanbuf(c, c.sendx), cas.elem)
	c.sendx++
	if c.sendx == c.dataqsiz {
		c.sendx = 0
	}
	c.qcount++
	selunlock(scases, lockorder)
	goto retc

// 同步接收
recv:
	// can receive from sleeping sender (sg)
	recv(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	if debugSelect {
		print("syncrecv: cas0=", cas0, " c=", c, "\n")
	}
	recvOK = true
	goto retc

rclose:
	// read at end of closed channel
	selunlock(scases, lockorder)
	recvOK = false
	if cas.elem != nil {
		typedmemclr(c.elemtype, cas.elem)
	}
	if raceenabled {
		raceacquire(c.raceaddr())
	}
	goto retc

// 同步发送
send:
	// can send to a sleeping receiver (sg)
	if raceenabled {
		raceReadObjectPC(c.elemtype, cas.elem, cas.pc, chansendpc)
	}
	if msanenabled {
		msanread(cas.elem, c.elemtype.size)
	}
	send(c, sg, cas.elem, func() { selunlock(scases, lockorder) }, 2)
	if debugSelect {
		print("syncsend: cas0=", cas0, " c=", c, "\n")
	}
	goto retc

retc:
	if cas.releasetime > 0 {
		blockevent(cas.releasetime-t0, 1)
	}
	return casi, recvOK

sclose:
	// send on closed channel
	selunlock(scases, lockorder)
	panic(plainError("send on closed channel"))
}
```

该过程概括为：

1. 利用`pollorder`生成随机序列；
2. 对所有的`channel`进行上锁操作（`lockorder`)，即上面的`sellock(scases, lockorder)`；
3. 从可用的`case`对应的`channel`中随机挑选一个执行；
4. 如果都不可用，如果有`default case`，尝试执行`default case`；
5. 如果没有任何准备好的`case`，将当前`select G`打包成`sudog`，放到所有`channel`排队列表(`lockorder`)，进入休眠，等待被某个管道唤醒；
6. 唤醒过后，遍历所有的`channel`的列表，找出到底是哪个`case`唤醒当前`select G`的，顺带清理放在其他`channel`队列中排队的元素；
7. 如果还是没有找到可用`case`，有可能是被意外唤醒，则重新有第二步开始；
8. 找到可用`case`，进行解锁操作，退出。

我们看下上锁和解锁的函数：

**select.go**

```go
func sellock(scases []scase, lockorder []uint16) {
	var c *hchan
	for _, o := range lockorder {
		c0 := scases[o].c
    // 如果和前一channel地址不同，则加锁
    // lockorder的作用就是避免对同一channel重复加锁
		if c0 != nil && c0 != c {
			c = c0
			lock(&c.lock)
		}
	}
}


func lock(l *mutex) {
	gp := getg()
	if gp.m.locks < 0 {
		throw("runtime·lock: lock count")
	}
	gp.m.locks++  // 锁采用引用计数的方式

	// Speculative grab for lock.
	if atomic.Casuintptr(&l.key, 0, locked) {
		return
	}
	semacreate(gp.m)

	// On uniprocessor's, no point spinning.
	// On multiprocessors, spin for ACTIVE_SPIN attempts.
	spin := 0
	if ncpu > 1 {
		spin = active_spin
	}
Loop:  // 自旋
	for i := 0; ; i++ {
		v := atomic.Loaduintptr(&l.key)
		if v&locked == 0 {
			// Unlocked. Try to lock.
			if atomic.Casuintptr(&l.key, v, v|locked) {
				return
			}
			i = 0
		}
		if i < spin {
			procyield(active_spin_cnt)
		} else if i < spin+passive_spin {
			osyield()
		} else {
			// Someone else has it.
			// l->waitm points to a linked list of M's waiting
			// for this lock, chained through m->nextwaitm.
			// Queue this M.
			for {
				gp.m.nextwaitm = muintptr(v &^ locked)
				if atomic.Casuintptr(&l.key, v, uintptr(unsafe.Pointer(gp.m))|locked) {
					break
				}
				v = atomic.Loaduintptr(&l.key)
				if v&locked == 0 {
					continue Loop
				}
			}
			if v&locked != 0 {
				// Queued. Wait.
				semasleep(-1)
				i = 0
			}
		}
	}
}


func selunlock(scases []scase, lockorder []uint16) {
	// We must be very careful here to not touch sel after we have unlocked
	// the last lock, because sel can be freed right after the last unlock.
	// Consider the following situation.
	// First M calls runtime·park() in runtime·selectgo() passing the sel.
	// Once runtime·park() has unlocked the last lock, another M makes
	// the G that calls select runnable again and schedules it for execution.
	// When the G runs on another M, it locks all the locks and frees sel.
	// Now if the first M touches sel, it will access freed memory.
	for i := len(scases) - 1; i >= 0; i-- {
		c := scases[lockorder[i]].c
		if c == nil {
			break
		}
    // 避免重复解锁
		if i > 0 && c == scases[lockorder[i-1]].c {
			continue // will unlock it on the next iteration
		}
		unlock(&c.lock)
	}
}


func unlock(l *mutex) {
	gp := getg()
	var mp *m
	for {
		v := atomic.Loaduintptr(&l.key)
		if v == locked {
			if atomic.Casuintptr(&l.key, locked, 0) {
				break
			}
		} else {
			// Other M's are waiting for the lock.
			// Dequeue an M.
			mp = muintptr(v &^ locked).ptr()
			if atomic.Casuintptr(&l.key, v, uintptr(mp.nextwaitm)) {
				// Dequeued an M.  Wake it.
				semawakeup(mp)
				break
			}
		}
	}
	gp.m.locks--
	if gp.m.locks < 0 {
		throw("runtime·unlock: lock count")
	}
	if gp.m.locks == 0 && gp.preempt { // restore the preemption request in case we've cleared it in newstack
		gp.stackguard0 = stackPreempt
	}
}
```

