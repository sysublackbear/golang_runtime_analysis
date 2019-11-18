# 11.析构

析构的主要用途是在对象被垃圾回收时执行一个关联函数，效果如同OOP里的析构方法。

示例：

**test.go**

```go
package main

import (
  "fmt"
	"runtime"
  "time"
)

func main() {
  x := 123
  runtime.SetFinalizer(&x, func(x *int) {
    fmt.Println("x finalizer.")
  })
  
  runtime.GC()
  time.Sleep(time.Minute)
}
```



## 11.1.设置

首先得为目标对象关联一个析构函数。`SetFinalizer`会通过接口内部的类型信息对目标对象和`finalizer`函数（参数变量，类型等）做出检查，确保它们符合要求。

**mfinal.go**

```go
func SetFinalizer(obj interface{}, finalizer interface{}) {
	if debug.sbrk != 0 {
		// debug.sbrk never frees memory, so no finalizers run
		// (and we don't have the data structures to record them).
		return
	}
  // 从接口获取类型和对象指针
	e := efaceOf(&obj)
	etyp := e._type
	if etyp == nil {
		throw("runtime.SetFinalizer: first argument is nil")
	}
	if etyp.kind&kindMask != kindPtr {
		throw("runtime.SetFinalizer: first argument is " + etyp.string() + ", not pointer")
	}
	ot := (*ptrtype)(unsafe.Pointer(etyp))
	if ot.elem == nil {
		throw("nil elem type!")
	}

	// find the containing object
  // 忽略nil对象
	base, _, _ := findObject(uintptr(e.data), 0, 0)

	if base == 0 {
		// 0-length objects are okay.
		if e.data == unsafe.Pointer(&zerobase) {
			return
		}

		// Global initializers might be linker-allocated.
		//	var Foo = &Object{}
		//	func main() {
		//		runtime.SetFinalizer(Foo, nil)
		//	}
		// The relevant segments are: noptrdata, data, bss, noptrbss.
		// We cannot assume they are in any order or even contiguous,
		// due to external linking.
		for datap := &firstmoduledata; datap != nil; datap = datap.next {
			if datap.noptrdata <= uintptr(e.data) && uintptr(e.data) < datap.enoptrdata ||
				datap.data <= uintptr(e.data) && uintptr(e.data) < datap.edata ||
				datap.bss <= uintptr(e.data) && uintptr(e.data) < datap.ebss ||
				datap.noptrbss <= uintptr(e.data) && uintptr(e.data) < datap.enoptrbss {
				return
			}
		}
		throw("runtime.SetFinalizer: pointer not in allocated block")
	}

	if uintptr(e.data) != base {
		// As an implementation detail we allow to set finalizers for an inner byte
		// of an object if it could come from tiny alloc (see mallocgc for details).
		if ot.elem == nil || ot.elem.ptrdata != 0 || ot.elem.size >= maxTinySize {
			throw("runtime.SetFinalizer: pointer not at beginning of allocated block")
		}
	}

  // 获取finalizer函数信息
	f := efaceOf(&finalizer)
	ftyp := f._type
  
  // 如果finalizer = nil，则移除析构函数
	if ftyp == nil {
		// switch to system stack and remove finalizer
		systemstack(func() {
			removefinalizer(e.data)
		})
		return
	}

  // 确保 finalizer 是函数
	if ftyp.kind&kindMask != kindFunc {
		throw("runtime.SetFinalizer: second argument is " + ftyp.string() + ", not a function")
	}
  
  // 检查finalizer参数数量及其类型
	ft := (*functype)(unsafe.Pointer(ftyp))
	if ft.dotdotdot() {
		throw("runtime.SetFinalizer: cannot pass " + etyp.string() + " to finalizer " + ftyp.string() + " because dotdotdot")
	}
	if ft.inCount != 1 {
		throw("runtime.SetFinalizer: cannot pass " + etyp.string() + " to finalizer " + ftyp.string())
	}
	fint := ft.in()[0]
	switch {
	case fint == etyp:
		// ok - same type
    // okarg相同类型
		goto okarg
	case fint.kind&kindMask == kindPtr:
		if (fint.uncommon() == nil || etyp.uncommon() == nil) && (*ptrtype)(unsafe.Pointer(fint)).elem == ot.elem {
			// ok - not same type, but both pointers,
			// one or the other is unnamed, and same element type, so assignable.
			goto okarg
		}
	case fint.kind&kindMask == kindInterface:
		ityp := (*interfacetype)(unsafe.Pointer(fint))
		if len(ityp.mhdr) == 0 {
			// ok - satisfies empty interface
			goto okarg
		}
		if _, ok := assertE2I2(ityp, *efaceOf(&obj)); ok {
			goto okarg
		}
	}
  // 检查结果错误，抛出异常
	throw("runtime.SetFinalizer: cannot pass " + etyp.string() + " to finalizer " + ftyp.string())
okarg:
	// compute size needed for return parameters
  // 计算返回参数大小
	nret := uintptr(0)
	for _, t := range ft.out() {
		nret = round(nret, uintptr(t.align)) + uintptr(t.size)
	}
	nret = round(nret, sys.PtrSize)

	// make sure we have a finalizer goroutine
  // 确保finalizer goroutine运行
	createfing()

  // 不能重复设置finalizer函数
	systemstack(func() {
		if !addfinalizer(e.data, (*funcval)(f.data), nret, fint, ot) {
			throw("runtime.SetFinalizer: finalizer already set")
		}
	})
}
```

析构函数会被打包成`specialfinalizer`对象。

**mheap.go**

```go
//go:notinheap
type special struct {
	next   *special // linked list in span
	offset uint16   // span offset of object
	kind   byte     // kind of special
}

type specialfinalizer struct {
	special special
	fn      *funcval // May be a heap pointer.
	nret    uintptr
	fint    *_type   // May be a heap pointer, but always live.
	ot      *ptrtype // May be a heap pointer, but always live.
}

// Adds a finalizer to the object p. Returns true if it succeeded.
func addfinalizer(p unsafe.Pointer, f *funcval, nret uintptr, fint *_type, ot *ptrtype) bool {
	lock(&mheap_.speciallock)
  // 从固定分配器创建specialfinalizer
	s := (*specialfinalizer)(mheap_.specialfinalizeralloc.alloc())
	unlock(&mheap_.speciallock)
	s.special.kind = _KindSpecialFinalizer
	s.fn = f
	s.nret = nret
	s.fint = fint
	s.ot = ot
  
  // 添加（注意，使用了匿名嵌入字段）
	if addspecial(p, &s.special) {
		// This is responsible for maintaining the same
		// GC-related invariants as markrootSpans in any
		// situation where it's possible that markrootSpans
		// has already run but mark termination hasn't yet.
		if gcphase != _GCoff {
			base, _, _ := findObject(uintptr(p), 0, 0)
			mp := acquirem()
			gcw := &mp.p.ptr().gcw
			// Mark everything reachable from the object
			// so it's retained for the finalizer.
			scanobject(base, gcw)
			// Mark the finalizer itself, since the
			// special isn't part of the GC'd heap.
			scanblock(uintptr(unsafe.Pointer(&s.fn)), sys.PtrSize, &oneptrmask[0], gcw, nil)
			releasem(mp)
		}
		return true
	}

	// There was an old finalizer
  // 已经有finalizer，释放当前specialfinalizer
	lock(&mheap_.speciallock)
	mheap_.specialfinalizeralloc.free(unsafe.Pointer(s))
	unlock(&mheap_.speciallock)
	return false
}
```

最终，`specialfinalizer`被保存到`span.specials`链表。这里的算法很有意思，利用目标对象再`span`的地址偏移量作为去重和排序条件。如此，单个循环就可以完成去重判断和有序添加操作。

**mheap.go**

```go
type mspan struct {
	next *mspan     // next span in list, or nil if none
	prev *mspan     // previous span in list, or nil if none
	list *mSpanList // For debugging. TODO: Remove.

	startAddr uintptr // address of first byte of span aka s.base()
	npages    uintptr // number of pages in span

	manualFreeList gclinkptr // list of free objects in mSpanManual spans

	// freeindex is the slot index between 0 and nelems at which to begin scanning
	// for the next free object in this span.
	// Each allocation scans allocBits starting at freeindex until it encounters a 0
	// indicating a free object. freeindex is then adjusted so that subsequent scans begin
	// just past the newly discovered free object.
	//
	// If freeindex == nelem, this span has no free objects.
	//
	// allocBits is a bitmap of objects in this span.
	// If n >= freeindex and allocBits[n/8] & (1<<(n%8)) is 0
	// then object n is free;
	// otherwise, object n is allocated. Bits starting at nelem are
	// undefined and should never be referenced.
	//
	// Object n starts at address n*elemsize + (start << pageShift).
	freeindex uintptr
	// TODO: Look up nelems from sizeclass and remove this field if it
	// helps performance.
	nelems uintptr // number of object in the span.

	// Cache of the allocBits at freeindex. allocCache is shifted
	// such that the lowest bit corresponds to the bit freeindex.
	// allocCache holds the complement of allocBits, thus allowing
	// ctz (count trailing zero) to use it directly.
	// allocCache may contain bits beyond s.nelems; the caller must ignore
	// these.
	allocCache uint64

	// allocBits and gcmarkBits hold pointers to a span's mark and
	// allocation bits. The pointers are 8 byte aligned.
	// There are three arenas where this data is held.
	// free: Dirty arenas that are no longer accessed
	//       and can be reused.
	// next: Holds information to be used in the next GC cycle.
	// current: Information being used during this GC cycle.
	// previous: Information being used during the last GC cycle.
	// A new GC cycle starts with the call to finishsweep_m.
	// finishsweep_m moves the previous arena to the free arena,
	// the current arena to the previous arena, and
	// the next arena to the current arena.
	// The next arena is populated as the spans request
	// memory to hold gcmarkBits for the next GC cycle as well
	// as allocBits for newly allocated spans.
	//
	// The pointer arithmetic is done "by hand" instead of using
	// arrays to avoid bounds checks along critical performance
	// paths.
	// The sweep will free the old allocBits and set allocBits to the
	// gcmarkBits. The gcmarkBits are replaced with a fresh zeroed
	// out memory.
	allocBits  *gcBits
	gcmarkBits *gcBits

	// sweep generation:
	// if sweepgen == h->sweepgen - 2, the span needs sweeping
	// if sweepgen == h->sweepgen - 1, the span is currently being swept
	// if sweepgen == h->sweepgen, the span is swept and ready to use
	// if sweepgen == h->sweepgen + 1, the span was cached before sweep began and is still cached, and needs sweeping
	// if sweepgen == h->sweepgen + 3, the span was swept and then cached and is still cached
	// h->sweepgen is incremented by 2 after every GC

	sweepgen    uint32
	divMul      uint16     // for divide by elemsize - divMagic.mul
	baseMask    uint16     // if non-0, elemsize is a power of 2, & this will get object allocation base
	allocCount  uint16     // number of allocated objects
	spanclass   spanClass  // size class and noscan (uint8)
	state       mSpanState // mspaninuse etc
	needzero    uint8      // needs to be zeroed before allocation
	divShift    uint8      // for divide by elemsize - divMagic.shift
	divShift2   uint8      // for divide by elemsize - divMagic.shift2
	scavenged   bool       // whether this span has had its pages released to the OS
	elemsize    uintptr    // computed from sizeclass or from npages
	limit       uintptr    // end of data in span
	speciallock mutex      // guards specials list
	specials    *special   // 按偏移量对链表进行排序
}


func addspecial(p unsafe.Pointer, s *special) bool {
  // 找到目标对象所属span
	span := spanOfHeap(uintptr(p))
	if span == nil {
		throw("addspecial on invalid pointer")
	}

	// Ensure that the span is swept.
	// Sweeping accesses the specials list w/o locks, so we have
	// to synchronize with it. And it's just much safer.
	mp := acquirem()
	span.ensureSwept()

  // 计算地址偏移量
	offset := uintptr(p) - span.base()
	kind := s.kind

	lock(&span.speciallock)

	// Find splice point, check for existing record.
  // 遍历span.specials链表
  // 通过偏移量和_KindSpecialFinalizer检查是否已设置finalizer
	t := &span.specials
	for {
		x := *t
		if x == nil {
			break
		}
    // 已设置了，返回false
		if offset == uintptr(x.offset) && kind == x.kind {
			unlock(&span.speciallock)
			releasem(mp)
			return false // already exists
		}
    // 因为span.specials按offset排序，所以没必要超出范围检查，直接退出循环。
		if offset < uintptr(x.offset) || (offset == uintptr(x.offset) && kind < x.kind) {
			break
		}
		t = &x.next
	}

	// Splice in record, fill in offset.
  // 利用上面的循环中断，将special插入到链表合适的地方，保持有序
	s.offset = uint16(offset)
	s.next = *t
	*t = s
	unlock(&span.speciallock)
	releasem(mp)

	return true
}


```

移除操作也只需要用偏移量遍历`span.specials`链表即可。

**mheap.go**

```go
func removespecial(p unsafe.Pointer, kind uint8) *special {
  // 查找所属span
	span := spanOfHeap(uintptr(p))
	if span == nil {
		throw("removespecial on invalid pointer")
	}

	// Ensure that the span is swept.
	// Sweeping accesses the specials list w/o locks, so we have
	// to synchronize with it. And it's just much safer.
	mp := acquirem()
	span.ensureSwept()

  // 计算地址偏移量
	offset := uintptr(p) - span.base()

	lock(&span.speciallock)
  // 遍历span的链表，移除special
	t := &span.specials
	for {
		s := *t
		if s == nil {
			break
		}
		// This function is used for finalizers only, so we don't check for
		// "interior" specials (p must be exactly equal to s->offset).
		if offset == uintptr(s.offset) && kind == s.kind {
			*t = s.next
			unlock(&span.speciallock)
			releasem(mp)
			return s
		}
		t = &s.next
	}
	unlock(&span.speciallock)
	releasem(mp)
	return nil
}
```



## 11.2.清理

垃圾清理操作在处理span时会检查`specials`链表，将不可达对象的`finalizer`函数添加到一个特定待执行队列。

**mgcsweep.go**

```go
func (s *mspan) sweep(preserve bool) bool {
	// It's critical that we enter this function with preemption disabled,
	// GC must not start while we are in the middle of this function.
	_g_ := getg()
	if _g_.m.locks == 0 && _g_.m.mallocing == 0 && _g_ != _g_.m.g0 {
		throw("mspan.sweep: m is not locked")
	}
	sweepgen := mheap_.sweepgen
	if s.state != mSpanInUse || s.sweepgen != sweepgen-1 {
		print("mspan.sweep: state=", s.state, " sweepgen=", s.sweepgen, " mheap.sweepgen=", sweepgen, "\n")
		throw("mspan.sweep: bad span state")
	}

	if trace.enabled {
		traceGCSweepSpan(s.npages * _PageSize)
	}

	atomic.Xadd64(&mheap_.pagesSwept, int64(s.npages))

	spc := s.spanclass
	size := s.elemsize
	res := false

	c := _g_.m.mcache
	freeToHeap := false

	// The allocBits indicate which unmarked objects don't need to be
	// processed since they were free at the end of the last GC cycle
	// and were not allocated since then.
	// If the allocBits index is >= s.freeindex and the bit
	// is not marked then the object remains unallocated
	// since the last GC.
	// This situation is analogous to being on a freelist.

	// Unlink & free special records for any objects we're about to free.
	// Two complications here:
	// 1. An object can have both finalizer and profile special records.
	//    In such case we need to queue finalizer for execution,
	//    mark the object as live and preserve the profile special.
	// 2. A tiny object can have several finalizers setup for different offsets.
	//    If such object is not marked, we need to queue all finalizers at once.
	// Both 1 and 2 are possible at the same time.
	specialp := &s.specials
	special := *specialp
	for special != nil {
		// A finalizer can be set for an inner byte of an object, find object beginning.
    // 利用偏移量计算出目标对象地址
		objIndex := uintptr(special.offset) / size
		p := s.base() + objIndex*size
		mbits := s.markBitsForIndex(objIndex)
		if !mbits.isMarked() {
			// This object is not marked and has at least one special record.
			// Pass 1: see if it has at least one finalizer.
			hasFin := false
			endOffset := p - s.base() + size
			for tmp := special; tmp != nil && uintptr(tmp.offset) < endOffset; tmp = tmp.next {
				if tmp.kind == _KindSpecialFinalizer {
					// Stop freeing of object if it has a finalizer.
          // 重新将目标对象标记，避免被清理
          // 为了让finalizer正确执行，必须延长目标对象生命周期
					mbits.setMarkedNonAtomic()
					hasFin = true
					break
				}
			}
			// Pass 2: queue all finalizers _or_ handle profile record.
      // 检查回收标记
      // 如果没有标记，那么属于可回收对象，准备执行finalizer
			for special != nil && uintptr(special.offset) < endOffset {
				// Find the exact byte for which the special was setup
				// (as opposed to object beginning).
				p := s.base() + uintptr(special.offset)
				if special.kind == _KindSpecialFinalizer || !hasFin {
					// Splice out special record.
					y := special
          
          // 调整span.specials链表
					special = special.next
					*specialp = special
          
          // 释放special，将finalizer放入待执行队列
          // 下次回收该对象时，已经没有finalizer需要处理了
					freespecial(y, unsafe.Pointer(p), size)
				} else {
					// This is profile record, but the object has finalizers (so kept alive).
					// Keep special record.
					specialp = &special.next
					special = *specialp
				}
			}
		} else {
			// object is still live: keep special record
			specialp = &special.next
			special = *specialp
		}
	}

	if debug.allocfreetrace != 0 || debug.clobberfree != 0 || raceenabled || msanenabled {
		// Find all newly freed objects. This doesn't have to
		// efficient; allocfreetrace has massive overhead.
		mbits := s.markBitsForBase()
		abits := s.allocBitsForIndex(0)
		for i := uintptr(0); i < s.nelems; i++ {
			if !mbits.isMarked() && (abits.index < s.freeindex || abits.isMarked()) {
				x := s.base() + i*s.elemsize
				if debug.allocfreetrace != 0 {
					tracefree(unsafe.Pointer(x), size)
				}
				if debug.clobberfree != 0 {
					clobberfree(unsafe.Pointer(x), size)
				}
				if raceenabled {
					racefree(unsafe.Pointer(x), size)
				}
				if msanenabled {
					msanfree(unsafe.Pointer(x), size)
				}
			}
			mbits.advance()
			abits.advance()
		}
	}

	// Count the number of free objects in this span.
	nalloc := uint16(s.countAlloc())
	if spc.sizeclass() == 0 && nalloc == 0 {
		s.needzero = 1
		freeToHeap = true
	}
	nfreed := s.allocCount - nalloc
	if nalloc > s.allocCount {
		print("runtime: nelems=", s.nelems, " nalloc=", nalloc, " previous allocCount=", s.allocCount, " nfreed=", nfreed, "\n")
		throw("sweep increased allocation count")
	}

	s.allocCount = nalloc
	wasempty := s.nextFreeIndex() == s.nelems
	s.freeindex = 0 // reset allocation index to start of span.
	if trace.enabled {
		getg().m.p.ptr().traceReclaimed += uintptr(nfreed) * s.elemsize
	}

	// gcmarkBits becomes the allocBits.
	// get a fresh cleared gcmarkBits in preparation for next GC
	s.allocBits = s.gcmarkBits
	s.gcmarkBits = newMarkBits(s.nelems)

	// Initialize alloc bits cache.
	s.refillAllocCache(0)

	// We need to set s.sweepgen = h.sweepgen only when all blocks are swept,
	// because of the potential for a concurrent free/SetFinalizer.
	// But we need to set it before we make the span available for allocation
	// (return it to heap or mcentral), because allocation code assumes that a
	// span is already swept if available for allocation.
	if freeToHeap || nfreed == 0 {
		// The span must be in our exclusive ownership until we update sweepgen,
		// check for potential races.
		if s.state != mSpanInUse || s.sweepgen != sweepgen-1 {
			print("mspan.sweep: state=", s.state, " sweepgen=", s.sweepgen, " mheap.sweepgen=", sweepgen, "\n")
			throw("mspan.sweep: bad span state after sweep")
		}
		// Serialization point.
		// At this point the mark bits are cleared and allocation ready
		// to go so release the span.
		atomic.Store(&s.sweepgen, sweepgen)
	}

	if nfreed > 0 && spc.sizeclass() != 0 {
		c.local_nsmallfree[spc.sizeclass()] += uintptr(nfreed)
		res = mheap_.central[spc].mcentral.freeSpan(s, preserve, wasempty)
		// mcentral.freeSpan updates sweepgen
	} else if freeToHeap {
		// Free large span to heap

		// NOTE(rsc,dvyukov): The original implementation of efence
		// in CL 22060046 used sysFree instead of sysFault, so that
		// the operating system would eventually give the memory
		// back to us again, so that an efence program could run
		// longer without running out of memory. Unfortunately,
		// calling sysFree here without any kind of adjustment of the
		// heap data structures means that when the memory does
		// come back to us, we have the wrong metadata for it, either in
		// the mspan structures or in the garbage collection bitmap.
		// Using sysFault here means that the program will run out of
		// memory fairly quickly in efence mode, but at least it won't
		// have mysterious crashes due to confused memory reuse.
		// It should be possible to switch back to sysFree if we also
		// implement and then call some kind of mheap.deleteSpan.
		if debug.efence > 0 {
			s.limit = 0 // prevent mlookup from finding this span
			sysFault(unsafe.Pointer(s.base()), size)
		} else {
			mheap_.freeSpan(s, true)
		}
		c.local_nlargefree++
		c.local_largefree += size
		res = true
	}
	if !res {
		// The span has been swept and is still in-use, so put
		// it on the swept in-use list.
		mheap_.sweepSpans[sweepgen/2%2].push(s)
	}
	return res
}
```

再看`freespecial`实现：

**mheap.go**

```go
func freespecial(s *special, p unsafe.Pointer, size uintptr) {
	switch s.kind {
	case _KindSpecialFinalizer:
    // addfinalizer创建的原本就是specialfinalizer，它匿名嵌入special
    // 这里是转换回原来的结构体
		sf := (*specialfinalizer)(unsafe.Pointer(s))
    
    // 将函数放入待运行队列
		queuefinalizer(p, sf.fn, sf.nret, sf.fint, sf.ot)
		lock(&mheap_.speciallock)
    // 释放specialfinalizer对象
		mheap_.specialfinalizeralloc.free(unsafe.Pointer(sf))
		unlock(&mheap_.speciallock)
	case _KindSpecialProfile:
		sp := (*specialprofile)(unsafe.Pointer(s))
		mProf_Free(sp.b, size)
		lock(&mheap_.speciallock)
		mheap_.specialprofilealloc.free(unsafe.Pointer(sp))
		unlock(&mheap_.speciallock)
	default:
		throw("bad special kind")
		panic("not reached")
	}
}
```

从清理操作对持有`finalizer`不可达对象的态度可以看出，析构函数会延长对象的生命周期，直到下一次垃圾回收才会真正被清理。其根本理由就是，`finalizer`函数执行时可能会访问目标对象，比如释放目标对象持有的相关资源等等。

另外，`finalizer`的执行依赖于垃圾清理操作，我们无法确定其准确执行时间。且不保证在进程退出前，一定会得到执行。因此，不能用`finalizer`去执行类似`flush cache`操作。



## 11.3.执行

### 11.3.1.执行队列

析构函数相关信息从`special`解包后，被重新打包成`finalizer`，然后被存储到一个由数组封装而成的`finblock`容器（块）里。多个`finblock`串成链表，形成队列。看上去很像前文垃圾回收器里的`gcWork`高性能缓存队列的做法。

**mfinal.go**

```go
type finalizer struct {
	fn   *funcval       // function to call (may be a heap pointer)
	arg  unsafe.Pointer // ptr to object (may be a heap pointer)
	nret uintptr        // bytes of return values from fn
	fint *_type         // type of first argument of fn
	ot   *ptrtype       // type of ptr to object (may be a heap pointer)
}

type finblock struct {
	alllink *finblock
	next    *finblock
	cnt     uint32
	_       int32
	fin     [(_FinBlockSize - 2*sys.PtrSize - 2*4) / unsafe.Sizeof(finalizer{})]finalizer
}
```

另外，还有几个全局变量用来管理`finblock`。

**mfinal.go**

```go
var finlock mutex  // protects the following variables
var fing *g        // goroutine that runs finalizers
var finq *finblock // 待运行finalizer队列（链表）
var finc *finblock // 提供finblock缓存复用对象
var finptrmask [_FinBlockSize / sys.PtrSize / 8]byte
var fingwait bool
var fingwake bool
var allfin *finblock // 所有finblock列表
```



### 11.3.2.执行

向队列添加析构函数的过程，基本上就是对`finblock`的操作。每次都向待执行队列的第一个容器`finq`添加，直到装满后将`finq`换成新的`finblock`块。

**mfinal.go**

```go
func queuefinalizer(p unsafe.Pointer, fn *funcval, nret uintptr, fint *_type, ot *ptrtype) {
	if gcphase != _GCoff {
		// Currently we assume that the finalizer queue won't
		// grow during marking so we don't have to rescan it
		// during mark termination. If we ever need to lift
		// this assumption, we can do it by adding the
		// necessary barriers to queuefinalizer (which it may
		// have automatically).
		throw("queuefinalizer during GC")
	}

	lock(&finlock)
  // finq是链表的第一个finblock，也是当前操作的目标
  // 如果为空，或者内部数组已满，则重新获取finblock替换finq
	if finq == nil || finq.cnt == uint32(len(finq.fin)) {
    // 如果复用缓存已空，申请新内存
		if finc == nil {
      // 有关persistent，请参考内存分配相关章节
			finc = (*finblock)(persistentalloc(_FinBlockSize, 0, &memstats.gc_sys))
      // 添加到allfin链表
			finc.alllink = allfin
			allfin = finc
			if finptrmask[0] == 0 {
				// Build pointer mask for Finalizer array in block.
				// Check assumptions made in finalizer1 array above.
				if (unsafe.Sizeof(finalizer{}) != 5*sys.PtrSize ||
					unsafe.Offsetof(finalizer{}.fn) != 0 ||
					unsafe.Offsetof(finalizer{}.arg) != sys.PtrSize ||
					unsafe.Offsetof(finalizer{}.nret) != 2*sys.PtrSize ||
					unsafe.Offsetof(finalizer{}.fint) != 3*sys.PtrSize ||
					unsafe.Offsetof(finalizer{}.ot) != 4*sys.PtrSize) {
					throw("finalizer out of sync")
				}
				for i := range finptrmask {
					finptrmask[i] = finalizer1[i%len(finalizer1)]
				}
			}
		}
    // 从复用链表头部提取finblock,并调整finc链表
		block := finc
		finc = block.next
    
    // 将新的finblock挂到finq待执行队列头部
		block.next = finq
		finq = block
	}
  
  // finq.cnt记录了finblock内部数组使用位置索引。
	f := &finq.fin[finq.cnt]
	atomic.Xadd(&finq.cnt, +1) // Sync with markroots
  
  // 设置相关属性
	f.fn = fn
	f.nret = nret
	f.fint = fint
	f.ot = ot
	f.arg = p
  
  // 设置fing唤醒标志。
	fingwake = true
	unlock(&finlock)
}
```

执行队列准备好以后，须由专门的`fing goroutine`负责执行。在`SetFinalizer`里我们就看到过`createfing`函数调用。

**mfinal.go**

```go
func createfing() {
	// start the finalizer goroutine exactly once
  // 确保仅执行一次
	if fingCreate == 0 && atomic.Cas(&fingCreate, 0, 1) {
		go runfinq()
	}
}
```

**mfinal.go**

```go
var fing *g        // goroutine that runs finalizers
var fingwait bool  // 休眠标记
var fingwake bool  // 唤醒标记

// This is the goroutine that runs all of the finalizers
func runfinq() {
	var (
		frame    unsafe.Pointer
		framecap uintptr
	)

	for {
		lock(&finlock)
    
    // 置换运行队列
    // 因为是并发，所以在执行runfinq时不能影响新的添加操作
		fb := finq
		finq = nil
    
    // 如果队列为空，则进入休眠
		if fb == nil {
      // 设置全局变量flag
			gp := getg()
			fing = gp
      
      // 设置休眠标志，休眠
			fingwait = true
			goparkunlock(&finlock, waitReasonFinalizerWait, traceEvGoBlock, 1)
      
      // 唤醒后重新检查队列
			continue
		}
		unlock(&finlock)
		if raceenabled {
			racefingo()
		}
    
    // 遍历finq链表
		for fb != nil {
      // 遍历finblock内部数组
			for i := fb.cnt; i > 0; i-- {
        // 获取并执行finalizer
				f := &fb.fin[i-1]

				framesz := unsafe.Sizeof((interface{})(nil)) + f.nret
				if framecap < framesz {
					// The frame does not contain pointers interesting for GC,
					// all not yet finalized objects are stored in finq.
					// If we do not mark it as FlagNoScan,
					// the last finalized object is not collected.
					frame = mallocgc(framesz, nil, true)
					framecap = framesz
				}

				if f.fint == nil {
					throw("missing type in runfinq")
				}
				// frame is effectively uninitialized
				// memory. That means we have to clear
				// it before writing to it to avoid
				// confusing the write barrier.
				*(*[2]uintptr)(frame) = [2]uintptr{}
				switch f.fint.kind & kindMask {
				case kindPtr:
					// direct use of pointer
					*(*unsafe.Pointer)(frame) = f.arg
				case kindInterface:
					ityp := (*interfacetype)(unsafe.Pointer(f.fint))
					// set up with empty interface
					(*eface)(frame)._type = &f.ot.typ
					(*eface)(frame).data = f.arg
					if len(ityp.mhdr) != 0 {
						// convert to interface with methods
						// this conversion is guaranteed to succeed - we checked in SetFinalizer
						*(*iface)(frame) = assertE2I(ityp, *(*eface)(frame))
					}
				default:
					throw("bad kind in runfinq")
				}
				fingRunning = true
				reflectcall(nil, unsafe.Pointer(f.fn), frame, uint32(framesz), uint32(framesz))
				fingRunning = false

				// Drop finalizer queue heap references
				// before hiding them from markroot.
				// This also ensures these will be
				// clear if we reuse the finalizer.
				f.fn = nil
				f.arg = nil
				f.ot = nil
				atomic.Store(&fb.cnt, i-1)
			}
			next := fb.next
			lock(&finlock)
      
      // 将当前已完成任务的finalizer对象放回finc复用缓存
			fb.next = finc
			finc = fb
			unlock(&finlock)
			fb = next
		}
	}
}
```

一路走来，已记不清`runtime`创建了多少类似fing这样的以死循环方式工作的goroutine了。好在像`fing`这样的都是按需创建的。



上面代码，循环遍历所有`finblock`，执行其中的析构函数。要说有所不同，就是它们会在同一个G栈串行执行。剩余问题是，当`fing`执行完毕进入休眠后，由谁来唤醒呢？要知道`queuefinalizer`仅仅设置了`fingwake`标志。



还记得调度循环里四处查找可用任务的`findrunnable`函数吗？没错，`fing`也是它要寻找的目标之一。

**proc.go**

```go
func findrunnable() (gp *g, inheritTime bool) {
	_g_ := getg()

	// The conditions here and in handoffp must agree: if
	// findrunnable would return a G to run, handoffp must start
	// an M.

top:
	_p_ := _g_.m.p.ptr()
	if sched.gcwaiting != 0 {
		gcstopm()
		goto top
	}
	if _p_.runSafePointFn != 0 {
		runSafePointFn()
	}
  
  // 如果fing正在休眠，且被设置了唤醒标志
	if fingwait && fingwake {
    // 唤醒
		if gp := wakefing(); gp != nil {
			ready(gp, 0, true)
		}
	}
	if *cgo_yield != nil {
		asmcgocall(*cgo_yield, nil)
	}

	// local runq
	if gp, inheritTime := runqget(_p_); gp != nil {
		return gp, inheritTime
	}

	// global runq
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
	if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Load64(&sched.lastpoll) != 0 {
		if list := netpoll(false); !list.empty() { // non-blocking
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
	for i := 0; i < 4; i++ {
		for enum := stealOrder.start(fastrand()); !enum.done(); enum.next() {
			if sched.gcwaiting != 0 {
				goto top
			}
			stealRunNextG := i > 2 // first look for ready queues with more than 1 g
			if gp := runqsteal(_p_, allp[enum.position()], stealRunNextG); gp != nil {
				return gp, false
			}
		}
	}

stop:

	// We have nothing to do. If we're in the GC mark phase, can
	// safely scan and blacken objects, and have work to do, run
	// idle-time marking rather than give up the P.
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
	if sched.gcwaiting != 0 || _p_.runSafePointFn != 0 {
		unlock(&sched.lock)
		goto top
	}
	if sched.runqsize != 0 {
		gp := globrunqget(_p_, 0)
		unlock(&sched.lock)
		return gp, false
	}
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
	for _, _p_ := range allpSnapshot {
		if !runqempty(_p_) {
			lock(&sched.lock)
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
	stopm()
	goto top
}
```

其中，`wakefing`逻辑：

```go
func wakefing() *g {
	var res *g
	lock(&finlock)
  
  // 再次检查唤醒条件
	if fingwait && fingwake {
		fingwait = false
		fingwake = false
		res = fing
	}
	unlock(&finlock)
	return res
}
```



**总结：**

不管是`panic`，还是`finalizer`，都有特定的使用场景，因为它们有相应的设计制约。这种制约不应该被看作是缺陷，毕竟我们本就不该让它们去做无法保证的事情。保持有限度的谨慎和悲观不是坏事，但不能因此就无理由地去抵制和忽视。了解其原理，永远不要停留在文档的字里行间。

