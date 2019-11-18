# 12.缓存池

设计对象缓存池，除避免内存分配操作开销外，更多的是为了避免分配大量临时对象对垃圾回收器造成负面影响。只是有一个问题需要解决，就是如何在多线程共享的情况下，解决同步锁带来的性能弊端，尤其是在高并发情形下。

因Go goroutine机制对线程的抽象，我们以往基于LTS的方案统统无法实施。就算`runtime`对我们开放线程访问接口也未必有用。因为G可能在中途被调度给其他线程，甚至你设置了LTS的线程会滚回闲置队列休眠。

为此，官方提供了一个深入`runtime`内核运作机制的`sync.Pool`。其算法已被内存分配，垃圾回收和调度器所使用，算是得到验证的成熟高效体系。



## 12.1.初始化

用于提供本地缓存对象分配的`poolLocal`类似内存分配器的`cache`，总是和P绑定，为当前工作线程提供快速无锁分配。而`Pool`则管理多个`P/poolLocal`。

**sync/pool.go**

```go
type Pool struct {
	noCopy noCopy

	local     unsafe.Pointer // [P]poolLocal数组指针
	localSize uintptr        // 数组内poolLocal的数量

	victim     unsafe.Pointer // local from previous cycle
	victimSize uintptr        // size of victims array

	// New optionally specifies a function to generate
	// a value when Get would otherwise return nil.
	// It may not be changed concurrently with calls to Get.
	New func() interface{}  // 新建对象函数
}

// Local per-P Pool appendix.
type poolLocalInternal struct {
	private interface{} // 私有缓存区
	shared  poolChain   // 可共享缓存区
}

type poolLocal struct {
	poolLocalInternal

	// Prevents false sharing on widespread platforms with
	// 128 mod (cache line size) = 0 .
	pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}
```

`Pool`用`local`和`localSize`维护一个动态`poolLocal`数组。无论是`Get`，还是`Put`操作都会通过`pin`来返回与当前`P`绑定的`poolLocal`对象，这里面就有初始化的关键。

**pool.go**

```go
// pin pins the current goroutine to P, disables preemption and
// returns poolLocal pool for the P and the P's id.
// Caller must call runtime_procUnpin() when done with the pool.
func (p *Pool) pin() (*poolLocal, int) {
  // 返回当前P.id
	pid := runtime_procPin()
	// In pinSlow we store to local and then to localSize, here we load in opposite order.
	// Since we've disabled preemption, GC cannot happen in between.
	// Thus here we must observe local at least as large localSize.
	// We can observe a newer/larger local, it is fine (we must observe its zero-initialized-ness).
	s := atomic.LoadUintptr(&p.localSize) // load-acquire
	l := p.local                          // load-consume
  
  // 如果P.id没有超出数组索引限制，则直接返回
  // 这是考虑到 procresize / GOMAXPROCS 的影响。
	if uintptr(pid) < s {
		return indexLocal(l, pid), pid
	}
  
  // 没有结果时，会涉及全局加锁操作
  // 比如重新分配数组内存，添加到全局列表
	return p.pinSlow()
}
```

`pinSlow`逻辑如下：

**pool.go**

```go
var (
	allPoolsMu Mutex

	// allPools is the set of pools that have non-empty primary
	// caches. Protected by either 1) allPoolsMu and pinning or 2)
	// STW.
	allPools []*Pool

	// oldPools is the set of pools that may have non-empty victim
	// caches. Protected by STW.
	oldPools []*Pool
)


func (p *Pool) pinSlow() (*poolLocal, int) {
	// Retry under the mutex.
	// Can not lock the mutex while pinned.
	runtime_procUnpin()
  
 	// 加锁
	allPoolsMu.Lock()
	defer allPoolsMu.Unlock()
	pid := runtime_procPin()
	// poolCleanup won't be called while we are pinned.
  // 再次检查是否符合条件，可能中途已被其他线程调用
	s := p.localSize
	l := p.local
	if uintptr(pid) < s {
		return indexLocal(l, pid), pid
	}
  
  // 如果数组为空，新建
  // 将其添加到allPools，垃圾回收器以此获取所有Pool实例
	if p.local == nil {
		allPools = append(allPools, p)
	}
	// If GOMAXPROCS changes between GCs, we re-allocate the array and lose the old one.
  
  // 根据P数量创建slice
	size := runtime.GOMAXPROCS(0)
	local := make([]poolLocal, size)
  
  // 将底层数组起始指针保存到Pool.local,并设置P.localSize
	atomic.StorePointer(&p.local, unsafe.Pointer(&local[0])) // store-release
	atomic.StoreUintptr(&p.localSize, uintptr(size))         // store-release
  
  // 返回本次所需的poolLocal
	return &local[pid], pid
}
```

至于`indexLocal`操作，如下：

**pool.go**

```go
func indexLocal(l unsafe.Pointer, i int) *poolLocal {
	lp := unsafe.Pointer(uintptr(l) + uintptr(i)*unsafe.Sizeof(poolLocal{}))
	return (*poolLocal)(lp)
}
```



## 12.2.Get/Set操作

和调度器对`P.runq`队列的处理方式类似。每个`poolLocal`有两个缓存区域：其中区域`private`完全私有，无须任何锁操作，优先级最高；另一区域`share`，允许被其他`poolLocal`访问，用来平衡调度缓存对象，无锁队列操作。

**pool.go**

```go
// Get selects an arbitrary item from the Pool, removes it from the
// Pool, and returns it to the caller.
// Get may choose to ignore the pool and treat it as empty.
// Callers should not assume any relation between values passed to Put and
// the values returned by Get.
//
// If Get would otherwise return nil and p.New is non-nil, Get returns
// the result of calling p.New.
func (p *Pool) Get() interface{} {
	if race.Enabled {
		race.Disable()
	}
  // 返回poolLocal
	l, pid := p.pin()
  
  // 优先从private选择
	x := l.private
	l.private = nil
	if x == nil {
		// Try to pop the head of the local shard. We prefer
		// the head over the tail for temporal locality of
		// reuse.
    // 从shared尾部提取缓存对象
		x, _ = l.shared.popHead()
    
    // 如果提取失败，则需要获取新的缓存对象
		if x == nil {
			x = p.getSlow(pid)
		}
	}
	runtime_procUnpin()
	if race.Enabled {
		race.Enable()
		if x != nil {
			race.Acquire(poolRaceAddr(x))
		}
	}
  
  // 如果还是不行，直接新创建对象进行分配
	if x == nil && p.New != nil {
		x = p.New()
	}
	return x
}
```

如果从本地获取缓存失败，则考虑从其他的`poolLocal`借调一个过来。如果实在不行，则调用`New`函数新建（最终手段）。

`getSlow`实现如下：

```go
func (p *Pool) getSlow(pid int) interface{} {
	// See the comment in pin regarding ordering of the loads.
	size := atomic.LoadUintptr(&p.localSize) // load-acquire
	locals := p.local                        // load-consume
	// Try to steal one element from other procs.
  // 从其他poolLocal偷取一个缓存对象
	for i := 0; i < int(size); i++ {
    // 获取目标poolLocal，且保证不是自身
		l := indexLocal(locals, (pid+i+1)%int(size))
    // 从shared列表提取一个对象
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

	// Mark the victim cache as empty for future gets don't bother
	// with it.
	atomic.StoreUintptr(&p.victimSize, 0)

	return nil
}
```

**注意：**`Get`操作后，缓存对象彻底与`Pool`失去引用关联，需要自行`Put`放回。



至于`Put`操作，就更简单了，无须考虑不同`poolLocal`之间的平衡调度。

**pool.go**

```go
// Put adds x to the pool.
func (p *Pool) Put(x interface{}) {
	if x == nil {
		return
	}
	if race.Enabled {
		if fastrand()%4 == 0 {
			// Randomly drop x on floor.
			return
		}
		race.ReleaseMerge(poolRaceAddr(x))
		race.Disable()
	}
  
  // 获取poolLocal
	l, _ := p.pin()
  
  // 优先放入private
	if l.private == nil {
		l.private = x
		x = nil
	}
  
  // 放入share
	if x != nil {
		l.shared.pushHead(x)
	}
	runtime_procUnpin()
	if race.Enabled {
		race.Enable()
	}
}
```



## 12.3.清理

借助垃圾回收机制，我们无须考虑Pool收缩问题，我们看看官方是怎么做的。

**mgc.go**

```go
// gcStart starts the GC. It transitions from _GCoff to _GCmark (if
// debug.gcstoptheworld == 0) or performs all of GC (if
// debug.gcstoptheworld != 0).
//
// This may return without performing this transition in some cases,
// such as when called on a system stack or with locks held.
func gcStart(trigger gcTrigger) {
	// Since this is called from malloc and malloc is called in
	// the guts of a number of libraries that might be holding
	// locks, don't attempt to start GC in non-preemptible or
	// potentially unstable situations.
	mp := acquirem()
	if gp := getg(); gp == mp.g0 || mp.locks > 1 || mp.preemptoff != "" {
		releasem(mp)
		return
	}
	releasem(mp)
	mp = nil

	// Pick up the remaining unswept/not being swept spans concurrently
	//
	// This shouldn't happen if we're being invoked in background
	// mode since proportional sweep should have just finished
	// sweeping everything, but rounding errors, etc, may leave a
	// few spans unswept. In forced mode, this is necessary since
	// GC can be forced at any point in the sweeping cycle.
	//
	// We check the transition condition continuously here in case
	// this G gets delayed in to the next GC cycle.
	for trigger.test() && sweepone() != ^uintptr(0) {
		sweep.nbgsweep++
	}

	// Perform GC initialization and the sweep termination
	// transition.
	semacquire(&work.startSema)
	// Re-check transition condition under transition lock.
	if !trigger.test() {
		semrelease(&work.startSema)
		return
	}

	// For stats, check if this GC was forced by the user.
	work.userForced = trigger.kind == gcTriggerCycle

	// In gcstoptheworld debug mode, upgrade the mode accordingly.
	// We do this after re-checking the transition condition so
	// that multiple goroutines that detect the heap trigger don't
	// start multiple STW GCs.
	mode := gcBackgroundMode
	if debug.gcstoptheworld == 1 {
		mode = gcForceMode
	} else if debug.gcstoptheworld == 2 {
		mode = gcForceBlockMode
	}

	// Ok, we're doing it! Stop everybody else
	semacquire(&worldsema)

	if trace.enabled {
		traceGCStart()
	}

	// Check that all Ps have finished deferred mcache flushes.
	for _, p := range allp {
		if fg := atomic.Load(&p.mcache.flushGen); fg != mheap_.sweepgen {
			println("runtime: p", p.id, "flushGen", fg, "!= sweepgen", mheap_.sweepgen)
			throw("p mcache not flushed")
		}
	}

	gcBgMarkStartWorkers()

	systemstack(gcResetMarkState)

	work.stwprocs, work.maxprocs = gomaxprocs, gomaxprocs
	if work.stwprocs > ncpu {
		// This is used to compute CPU time of the STW phases,
		// so it can't be more than ncpu, even if GOMAXPROCS is.
		work.stwprocs = ncpu
	}
	work.heap0 = atomic.Load64(&memstats.heap_live)
	work.pauseNS = 0
	work.mode = mode

	now := nanotime()
	work.tSweepTerm = now
	work.pauseStart = now
	if trace.enabled {
		traceGCSTWStart(1)
	}
	systemstack(stopTheWorldWithSema)
	// Finish sweep before we start concurrent scan.
	systemstack(func() {
		finishsweep_m()
	})
	// clearpools before we start the GC. If we wait they memory will not be
	// reclaimed until the next GC cycle.
	clearpools()

	work.cycles++

	gcController.startCycle()
	work.heapGoal = memstats.next_gc

	// In STW mode, disable scheduling of user Gs. This may also
	// disable scheduling of this goroutine, so it may block as
	// soon as we start the world again.
	if mode != gcBackgroundMode {
		schedEnableUser(false)
	}

	// Enter concurrent mark phase and enable
	// write barriers.
	//
	// Because the world is stopped, all Ps will
	// observe that write barriers are enabled by
	// the time we start the world and begin
	// scanning.
	//
	// Write barriers must be enabled before assists are
	// enabled because they must be enabled before
	// any non-leaf heap objects are marked. Since
	// allocations are blocked until assists can
	// happen, we want enable assists as early as
	// possible.
	setGCPhase(_GCmark)

	gcBgMarkPrepare() // Must happen before assist enable.
	gcMarkRootPrepare()

	// Mark all active tinyalloc blocks. Since we're
	// allocating from these, they need to be black like
	// other allocations. The alternative is to blacken
	// the tiny block on every allocation from it, which
	// would slow down the tiny allocator.
	gcMarkTinyAllocs()

	// At this point all Ps have enabled the write
	// barrier, thus maintaining the no white to
	// black invariant. Enable mutator assists to
	// put back-pressure on fast allocating
	// mutators.
	atomic.Store(&gcBlackenEnabled, 1)

	// Assists and workers can start the moment we start
	// the world.
	gcController.markStartTime = now

	// Concurrent mark.
	systemstack(func() {
		now = startTheWorldWithSema(trace.enabled)
		work.pauseNS += now - work.pauseStart
		work.tMark = now
	})
	// In STW mode, we could block the instant systemstack
	// returns, so don't do anything important here. Make sure we
	// block rather than returning to user code.
	if mode != gcBackgroundMode {
		Gosched()
	}

	semrelease(&work.startSema)
}
```

重点，我们看下`clearpools`代码：

```go
func clearpools() {
	// clear sync.Pools
	if poolcleanup != nil {
		poolcleanup()
	}

	// Clear central sudog cache.
	// Leave per-P caches alone, they have strictly bounded size.
	// Disconnect cached list before dropping it on the floor,
	// so that a dangling ref to one entry does not pin all of them.
	lock(&sched.sudoglock)
	var sg, sgnext *sudog
	for sg = sched.sudogcache; sg != nil; sg = sgnext {
		sgnext = sg.next
		sg.next = nil
	}
	sched.sudogcache = nil
	unlock(&sched.sudoglock)

	// Clear central defer pools.
	// Leave per-P pools alone, they have strictly bounded size.
	lock(&sched.deferlock)
	for i := range sched.deferpool {
		// disconnect cached list before dropping it on the floor,
		// so that a dangling ref to one entry does not pin all of them.
		var d, dlink *_defer
		for d = sched.deferpool[i]; d != nil; d = dlink {
			dlink = d.link
			d.link = nil
		}
		sched.deferpool[i] = nil
	}
	unlock(&sched.deferlock)
}




// Hooks for other packages

var poolcleanup func()

//go:linkname sync_runtime_registerPoolCleanup sync.runtime_registerPoolCleanup
func sync_runtime_registerPoolCleanup(f func()) {
	poolcleanup = f
}
```

`poolcleanup`函数需要额外注册。

**pool.go**

```go
func init() {
	runtime_registerPoolCleanup(poolCleanup)
}
```

真正的目标是`poolCleanup`。此时正处于STW状态，所以无须加锁操作。

```go
func poolCleanup() {
	// This function is called with the world stopped, at the beginning of a garbage collection.
	// It must not allocate and probably should not call any runtime functions.

	// Because the world is stopped, no pool user can be in a
	// pinned section (in effect, this has all Ps pinned).

	// Drop victim caches from all pools.
	for _, p := range oldPools {
    // 清理操作
		p.victim = nil
		p.victimSize = 0
	}

	// Move primary cache to victim cache.
	for _, p := range allPools {
    // 清理操作
		p.victim = p.local
		p.victimSize = p.localSize
    
    // 设置Pool.local = nil，除了解除所引用的数组空间外
    // 还让Pool.pinSlow方法将其重新添加到allPools
		p.local = nil
		p.localSize = 0
	}

	// The pools with non-empty primary caches now have non-empty
	// victim caches and no pools have primary caches.
	oldPools, allPools = allPools, nil
}
```



## 12.4.堆栈逃逸

### 12.4.1.关于堆和栈

栈 可以简单得理解成一次函数调用内部申请到的内存，它们会随着函数的返回把内存还给系统。

```go
func F() {
	temp := make([]int, 0, 20)
	...
}
```

类似于上面代码里面的`temp`变量，只是内函数内部申请的临时变量，并不会作为返回值返回，它就是被编译器申请到栈里面。

**申请到栈内存好处：函数返回直接释放，不会引起垃圾回收，对性能没有影响。**

再来看看堆得情况之一如下代码：

```go
func F() []int{
	a := make([]int, 0, 20)
	return a
}
```

而上面这段代码，申请的代码一模一样，但是申请后作为返回值返回了，编译器会认为变量之后还会被使用，当函数返回之后并不会将其内存归还，那么它就会被申请到 堆 上面了。

**申请到堆上面的内存才会引起垃圾回收，如果这个过程（特指垃圾回收不断被触发）过于高频就会导致 gc 压力过大，程序性能出问题。**

我们再看看如下几个例子：

```go
func F() {
	a := make([]int, 0, 20)     // 栈 空间小
	b := make([]int, 0, 20000) // 堆 空间过大
 
	l := 20
	c := make([]int, 0, l) // 堆 动态分配不定空间
}
```

+ 像是 b 这种 即使是临时变量，申请过大也会在堆上面申请。
+ 对于 c 编译器对于这种**不定长度**的申请方式，也会在堆上面申请，即使申请的长度很短。



### 12.4.2.逃逸分析（Escape analysis）

所谓逃逸分析（Escape analysis）是指由编译器决定内存分配的位置，不需要程序员指定。

在函数中申请一个新的对象：

- 如果分配在栈中，则函数执行结束可自动将内存回收；
- 如果分配在堆中，则函数执行结束可交给GC（垃圾回收）处理。

注意，对于函数外部没有引用的对象，也有可能放到堆中，比如内存过大超过栈的存储能力。



### 12.4.3.逃逸场景（其实就是讨论什么情况下会分配到堆中）

#### 12.4.3.1.指针逃逸

Go可以返回局部变量指针，这其实是一个典型的变量逃逸案例，示例代码如下：

```go
package main

type Student struct {
    Name string
    Age  int
}

func StudentRegister(name string, age int) *Student {
    s := new(Student) //局部变量s逃逸到堆

    s.Name = name
    s.Age = age

    return s
}

func main() {
    StudentRegister("Jim", 18)
}
```

虽然，在函数`StudentRegister()`内部`s`为局部变量，其值通过函数返回值返回，`s`本身为一指针，其指向的内存地址不会是栈而是堆，这就是典型的逃逸案例。

我们可以通过终端运行命令查看逃逸分析日志：

```bash
go build -gcflags=-m
```

可见在`StudentRegister()`函数中，也即代码第9行显示`”escapes to heap”`，代表该行内存分配发生了逃逸现象。



#### 12.4.3.2.栈空间不足逃逸（空间开辟过大）

```go
package main

func Slice() {
    s := make([]int, 10000, 10000)

    for index, _ := range s {
        s[index] = index
    }
}

func main() {
    Slice()
}
```

分析如下：

+ 当切片长度扩大到10000时就会逃逸。

+ **本质是，当栈空间不足以存放当前对象时或无法判断当前切片长度时会将对象分配到堆中。（看上面的c变量）**



#### 12.4.3.动态类型逃逸（不确定长度大小）

很多函数参数为interface类型，比如fmt.Println(a …interface{})，编译期间很难确定其参数的具体类型，也能产生逃逸。

如下代码所示：

```go
package main

import "fmt"

func main() {
    s := "Escape"
    fmt.Println(s)
}
```



#### 12.4.4.闭包引用对象逃逸

Fibonacci数列的函数：

```go
package main

import "fmt"

func Fibonacci() func() int {
    a, b := 0, 1
    return func() int {
        a, b = b, a+b
        return a
    }
}

func main() {
    f := Fibonacci()

    for i := 0; i < 10; i++ {
        fmt.Printf("Fibonacci: %d\n", f())
    }
}
```



### 12.5.逃逸分析的作用是？

1. 逃逸分析的好处是为了减少gc的压力，不逃逸的对象分配在栈上，当函数返回时就回收了资源，不需要gc标记清除。
2. 逃逸分析完后可以确定哪些变量可以分配在栈上，栈的分配比堆快，性能好(逃逸的局部变量会在堆上分配 ,而没有发生逃逸的则有编译器在栈上分配)。
3. 同步消除，如果你定义的对象的方法上有同步锁，但在运行时，却只有一个线程在访问，此时逃逸分析后的机器码，会去掉同步锁运行。



### 12.6.总结

- 栈上分配内存比在堆中分配内存有更高的效率
- 栈上分配的内存不需要GC处理
- 堆上分配的内存使用完毕会交给GC处理
- 逃逸分析目的是决定内分配地址是栈还是堆
- 逃逸分析在编译阶段完成。



所以，我们提一个问题：**函数传递指针真的比传值效率高吗？**

**答案**：我们知道传递指针可以减少底层值的拷贝，可以提高效率，但是如果拷贝的数据量小，由于指针传递会产生逃逸，可能会使用堆，也可能会增加GC的负担，所以传递指针不一定是高效的。

