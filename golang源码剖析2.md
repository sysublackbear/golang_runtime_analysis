## 4.2.大内存分配细节

主要实现在于：`largeAlloc`这个函数。

```go
func largeAlloc(size uintptr, needzero bool, noscan bool) *mspan {
	// print("largeAlloc size=", size, "\n")

	if size+_PageSize < size {
		throw("out of memory")
	}
  // 计算所需页数
	npages := size >> _PageShift
	if size&_PageMask != 0 {
		npages++
	}

	// Deduct credit for this span allocation and sweep if
	// necessary. mHeap_Alloc will also sweep npages, so this only
	// pays the debt down to npage pages.
  // 清理垃圾
	deductSweepCredit(npages*_PageSize, npages)

	s := mheap_.alloc(npages, makeSpanClass(0, noscan), true, needzero)
	if s == nil {
		throw("out of memory")
	}
	s.limit = s.base() + size
	heapBitsForAddr(s.base()).initSpan(s)
	return s
}
```



## 4.3.小对象扩张

再看看小对象的扩张函数：`mcache.go`的`nextFree`->`refill`。

```go
func (c *mcache) refill(spc spanClass) {
	// Return the current cached span to the central lists.
	s := c.alloc[spc]

  // 放弃当前正在使用的span（尚在 central.empty)
	if uintptr(s.allocCount) != s.nelems {
		throw("refill of span with free space remaining")
	}
	if s != &emptymspan {
		// Mark this span as no longer cached.
    // if sweepgen == h->sweepgen + 3, the span was swept and then cached and is still cached
		if s.sweepgen != mheap_.sweepgen+3 {
			throw("bad sweepgen in refill")
		}
		atomic.Store(&s.sweepgen, mheap_.sweepgen)  // 取消正在使用的标记
	}

	// Get a new cached span from the central lists.
  // 从central获取span进行替换。
  // 从central获取内存空间
	s = mheap_.central[spc].mcentral.cacheSpan()
	if s == nil {
		throw("out of memory")
	}

	if uintptr(s.allocCount) == s.nelems {
		throw("span has no free space")
	}

	// Indicate that this span is cached and prevent asynchronous
	// sweeping in the next sweep phase.
	s.sweepgen = mheap_.sweepgen + 3

	c.alloc[spc] = s
}


```

再看看sweepgen各世代的具体含义：

+ if sweepgen == h->sweepgen - 2, the span needs sweeping；
+ if sweepgen == h->sweepgen - 1, the span is currently being swept；
+ if sweepgen == h->sweepgen, the span is swept and ready to use；
+ if sweepgen == h->sweepgen + 1, the span was cached before sweep began and is still cached, and needs sweeping；
+ if sweepgen == h->sweepgen + 3, the span was swept and then cached and is still cached；
+ h->sweepgen is incremented by 2 after every GC。



**再看看从central中获取内存空间：**

```go
type mcentral struct {
	lock      mutex
	spanclass spanClass
  // 链表：span尚有空闲object可用；
	nonempty  mSpanList // list of spans with a free object, ie a nonempty free list
	// 链表：span没有空闲object可用，或已被cache取走
  empty     mSpanList // list of spans with no free objects (or cached in an mcache)

	// nmalloc is the cumulative count of objects allocated from
	// this mcentral, assuming all spans in mcaches are
	// fully-allocated. Written atomically, read under STW.
	nmalloc uint64
}

// Allocate a span to use in an mcache.
func (c *mcentral) cacheSpan() *mspan {
	// Deduct credit for this span allocation and sweep if necessary.
	spanBytes := uintptr(class_to_allocnpages[c.spanclass.sizeclass()]) * _PageSize
	deductSweepCredit(spanBytes, 0)

	lock(&c.lock)
	traceDone := false
	if trace.enabled {
		traceGCSweepStart()
	}
  // 上面是清理垃圾的过程...
	sg := mheap_.sweepgen
retry:
	var s *mspan
  // 遍历 nonempty 链表
	for s = c.nonempty.first; s != nil; s = s.next {
    // 需要清理的span
		if s.sweepgen == sg-2 && atomic.Cas(&s.sweepgen, sg-2, sg-1) {
      // 因为要交给cache使用，所以转移到empty链表
			c.nonempty.remove(s)
			c.empty.insertBack(s)
			unlock(&c.lock)
      // 垃圾清理
			s.sweep(true)
			goto havespan
		}
    // 忽略正在清理的span
		if s.sweepgen == sg-1 {
			// the span is being swept by background sweeper, skip
			continue
		}
		// we have a nonempty span that does not require sweeping, allocate from it
    // 已经清理过的span
		c.nonempty.remove(s)
		c.empty.insertBack(s)
		unlock(&c.lock)
		goto havespan
	}

  // 遍历empty链表
	for s = c.empty.first; s != nil; s = s.next {
    // 需要清理的span
		if s.sweepgen == sg-2 && atomic.Cas(&s.sweepgen, sg-2, sg-1) {
			// we have an empty span that requires sweeping,
			// sweep it and see if we can free some space in it
			c.empty.remove(s)
			// swept spans are at the end of the list
			c.empty.insertBack(s)
			unlock(&c.lock)
			s.sweep(true)
			freeIndex := s.nextFreeIndex()
      // 清理后有可用的object
			if freeIndex != s.nelems {
				s.freeindex = freeIndex
				goto havespan
			}
			lock(&c.lock)
			// the span is still empty after sweep
			// it is already in the empty list, so just retry
      // 清理后依然没可用的object,重试
			goto retry
		}
    // 忽略正在清理的span
		if s.sweepgen == sg-1 {
			// the span is being swept by background sweeper, skip
			continue
		}
		// already swept empty span,
		// all subsequent ones must also be either swept or in process of sweeping
    // 已清理过，且不为空的span都被转移到noempty链表
    // 这里剩下的自然都是全空或正在被cache使用的，继续循环已没有意义
		break
	}
	if trace.enabled {
		traceGCSweepDone()
		traceDone = true
	}
	unlock(&c.lock)

	// Replenish central list if empty.
  // 如果两个链表里都没span可用，扩张
	s = c.grow()
	if s == nil {
		return nil
	}
	lock(&c.lock)
  // 新span将被cache使用，所以放到empty链表尾部
	c.empty.insertBack(s)
	unlock(&c.lock)

	// At this point s is a non-empty span, queued at the end of the empty list,
	// c is unlocked.
havespan:
	if trace.enabled && !traceDone {
		traceGCSweepDone()
	}
	n := int(s.nelems) - int(s.allocCount)
	if n == 0 || s.freeindex == s.nelems || uintptr(s.allocCount) == s.nelems {
		throw("span has no free objects")
	}
	// Assume all objects from this span will be allocated in the
	// mcache. If it gets uncached, we'll adjust this.
	atomic.Xadd64(&c.nmalloc, int64(n))
	usedBytes := uintptr(s.allocCount) * s.elemsize
	atomic.Xadd64(&memstats.heap_live, int64(spanBytes)-int64(usedBytes))
	if trace.enabled {
		// heap_live changed.
		traceHeapAlloc()
	}
	if gcBlackenEnabled != 0 {
		// heap_live changed.
		gcController.revise()
	}
	freeByteBase := s.freeindex &^ (64 - 1)
	whichByte := freeByteBase / 8
	// Init alloc bits cache.
	s.refillAllocCache(whichByte)

	// Adjust the allocCache so that s.freeindex corresponds to the low bit in
	// s.allocCache.
	s.allocCache >>= s.freeindex % 64

	return s
}
```

可以看出，从central里获取span时，优先取用已有资源，哪怕是要执行清理操作。只有当现有资源都无法满足时，采取heap获取span，并重新切分成object链表。

```go
// grow allocates a new empty span from the heap and initializes it for c's size class.
func (c *mcentral) grow() *mspan {
	npages := uintptr(class_to_allocnpages[c.spanclass.sizeclass()])
	size := uintptr(class_to_size[c.spanclass.sizeclass()])
	// 计算切分object数量
  n := (npages << _PageShift) / size
	// 从heap获取span
	s := mheap_.alloc(npages, c.spanclass, false, true)
	if s == nil {
		return nil
	}

	p := s.base()
	s.limit = p + size*n

  // 重置在bitmap里的标记
	heapBitsForAddr(s.base()).initSpan(s)
	return s
}
```



## 4.4.堆内存的分配

```go
func (h *mheap) alloc(npage uintptr, spanclass spanClass, large bool, needzero bool) *mspan {
	// Don't do any operations that lock the heap on the G stack.
	// It might trigger stack growth, and the stack growth code needs
	// to be able to allocate heap.
	var s *mspan
	systemstack(func() {
		s = h.alloc_m(npage, spanclass, large)
	})

  // 对span进行清0操作
	if s != nil {
		if needzero && s.needzero != 0 {
			memclrNoHeapPointers(unsafe.Pointer(s.base()), s.npages<<_PageShift)
		}
		s.needzero = 0
	}
	return s
}

func (h *mheap) alloc_m(npage uintptr, spanclass spanClass, large bool) *mspan {
	_g_ := getg()

	// To prevent excessive heap growth, before allocating n pages
	// we need to sweep and reclaim at least n pages.
	if h.sweepdone == 0 {
		h.reclaim(npage)
	}

	lock(&h.lock)
	// transfer stats from cache to global
	memstats.heap_scan += uint64(_g_.m.mcache.local_scan)
	_g_.m.mcache.local_scan = 0
	memstats.tinyallocs += uint64(_g_.m.mcache.local_tinyallocs)
	_g_.m.mcache.local_tinyallocs = 0

  // 从heap获取指定页数的span
	s := h.allocSpanLocked(npage, &memstats.heap_inuse)
	if s != nil {
		// Record span info, because gc needs to be
		// able to map interior pointer to containing span.
    // 重置span状态
		atomic.Store(&s.sweepgen, h.sweepgen)
		h.sweepSpans[h.sweepgen/2%2].push(s) // Add to swept in-use list.
		s.state = mSpanInUse
		s.allocCount = 0
		s.spanclass = spanclass
		if sizeclass := spanclass.sizeclass(); sizeclass == 0 {
			s.elemsize = s.npages << _PageShift
			s.divShift = 0
			s.divMul = 0
			s.divShift2 = 0
			s.baseMask = 0
		} else {
			s.elemsize = uintptr(class_to_size[sizeclass])
			m := &class_to_divmagic[sizeclass]
			s.divShift = m.shift
			s.divMul = m.mul
			s.divShift2 = m.shift2
			s.baseMask = m.baseMask
		}

		// Mark in-use span in arena page bitmap.
		arena, pageIdx, pageMask := pageIndexOf(s.base())
		arena.pageInUse[pageIdx] |= pageMask

		// update stats, sweep lists
		h.pagesInUse += uint64(npage)
		if large {
			memstats.heap_objects++
			mheap_.largealloc += uint64(s.elemsize)
			mheap_.nlargealloc++
			atomic.Xadd64(&memstats.heap_live, int64(npage<<_PageShift))
		}
	}
	// heap_scan and heap_live were updated.
	if gcBlackenEnabled != 0 {
		gcController.revise()
	}

	if trace.enabled {
		traceHeapAlloc()
	}

	// h.spans is accessed concurrently without synchronization
	// from other threads. Hence, there must be a store/store
	// barrier here to ensure the writes to h.spans above happen
	// before the caller can publish a pointer p to an object
	// allocated from s. As soon as this happens, the garbage
	// collector running on another processor could read p and
	// look up s in h.spans. The unlock acts as the barrier to
	// order these writes. On the read side, the data dependency
	// between p and the index in h.spans orders the reads.
	unlock(&h.lock)
	return s
}
```

从heap获取span的算法核心是找到大小最合适的块。首先从页数相同的链表查找，如没有结果，再从页数更多的链表提取，直至超大块或申请新块。

如返回更大的span，为避免浪费，会将多余部分切出来重新放回heap链表。同时还会尝试合并相邻的闲置span空间，减少碎片。

```go
func (h *mheap) allocSpanLocked(npage uintptr, stat *uint64) *mspan {
	var s *mspan

  // 先尝试找出空闲的，指定页数的page
	s = h.pickFreeSpan(npage)
	if s != nil {
		goto HaveSpan
	}
	// On failure, grow the heap and try again.
  // 空间不够，就得从操作系统申请新的空间
	if !h.grow(npage) {
		return nil
	}
  
  // 再重试找出可用空间
	s = h.pickFreeSpan(npage)
	if s != nil {
		goto HaveSpan
	}
	throw("grew heap, but no adequate free span found")

HaveSpan:
	// Mark span in use.
	if s.state != mSpanFree {
		throw("candidate mspan for allocation is not free")
	}
	if s.npages < npage {
		throw("candidate mspan for allocation is too small")
	}

	// First, subtract any memory that was released back to
	// the OS from s. We will re-scavenge the trimmed section
	// if necessary.
	memstats.heap_released -= uint64(s.released())

  // 如果拿到的span页数多于预期 ...
	if s.npages > npage {
		// Trim extra and put it back in the heap.
    // 创建新span用来管理多余的内存
		t := (*mspan)(h.spanalloc.alloc())
		t.init(s.base()+npage<<_PageShift, s.npages-npage)
    
    // 调整切割后的页数
		s.npages = npage
		h.setSpan(t.base()-1, s)
		h.setSpan(t.base(), t)
		h.setSpan(t.base()+t.npages*pageSize-1, t)
		t.needzero = s.needzero
		// If s was scavenged, then t may be scavenged.
		start, end := t.physPageBounds()
		if s.scavenged && start < end {
			memstats.heap_released += uint64(end - start)
			t.scavenged = true
		}
		s.state = mSpanManual // prevent coalescing with s
		t.state = mSpanManual
    // 将新建span放回heap
		h.freeSpanLocked(t, false, false, s.unusedsince)
		s.state = mSpanFree
	}
	// "Unscavenge" s only AFTER splitting so that
	// we only sysUsed whatever we actually need.
	if s.scavenged {
		// sysUsed all the pages that are actually available
		// in the span. Note that we don't need to decrement
		// heap_released since we already did so earlier.
		sysUsed(unsafe.Pointer(s.base()), s.npages<<_PageShift)
		s.scavenged = false
	}
	s.unusedsince = 0

	h.setSpans(s.base(), npage, s)

  // 在span填充全部指针。
	*stat += uint64(npage << _PageShift)
	memstats.heap_idle -= uint64(npage << _PageShift)

	//println("spanalloc", hex(s.start<<_PageShift))
	if s.inList() {
		throw("still in list")
	}
	return s
}
```

```go
func (h *mheap) sysAlloc(n uintptr) (v unsafe.Pointer, size uintptr) {
	n = round(n, heapArenaBytes)

	// First, try the arena pre-reservation.
	v = h.arena.alloc(n, heapArenaBytes, &memstats.heap_sys)
	if v != nil {
		size = n
		goto mapped
	}

	// Try to grow the heap at a hint address.
	for h.arenaHints != nil {
		hint := h.arenaHints
		p := hint.addr
		if hint.down {
			p -= n
		}
		if p+n < p {
			// We can't use this, so don't ask.
			v = nil
		} else if arenaIndex(p+n-1) >= 1<<arenaBits {
			// Outside addressable heap. Can't use.
			v = nil
		} else {
			v = sysReserve(unsafe.Pointer(p), n)
		}
		if p == uintptr(v) {
			// Success. Update the hint.
			if !hint.down {
				p += n
			}
			hint.addr = p
			size = n
			break
		}
		// Failed. Discard this hint and try the next.
		//
		// TODO: This would be cleaner if sysReserve could be
		// told to only return the requested address. In
		// particular, this is already how Windows behaves, so
		// it would simply things there.
		if v != nil {
			sysFree(v, n, nil)
		}
		h.arenaHints = hint.next
		h.arenaHintAlloc.free(unsafe.Pointer(hint))
	}

	if size == 0 {
		if raceenabled {
			// The race detector assumes the heap lives in
			// [0x00c000000000, 0x00e000000000), but we
			// just ran out of hints in this region. Give
			// a nice failure.
			throw("too many address space collisions for -race mode")
		}

		// All of the hints failed, so we'll take any
		// (sufficiently aligned) address the kernel will give
		// us.
		v, size = sysReserveAligned(nil, n, heapArenaBytes)
		if v == nil {
			return nil, 0
		}

		// Create new hints for extending this region.
		hint := (*arenaHint)(h.arenaHintAlloc.alloc())
		hint.addr, hint.down = uintptr(v), true
		hint.next, mheap_.arenaHints = mheap_.arenaHints, hint
		hint = (*arenaHint)(h.arenaHintAlloc.alloc())
		hint.addr = uintptr(v) + size
		hint.next, mheap_.arenaHints = mheap_.arenaHints, hint
	}

	// Check for bad pointers or pointers we can't use.
	{
		var bad string
		p := uintptr(v)
		if p+size < p {
			bad = "region exceeds uintptr range"
		} else if arenaIndex(p) >= 1<<arenaBits {
			bad = "base outside usable address space"
		} else if arenaIndex(p+size-1) >= 1<<arenaBits {
			bad = "end outside usable address space"
		}
		if bad != "" {
			// This should be impossible on most architectures,
			// but it would be really confusing to debug.
			print("runtime: memory allocated by OS [", hex(p), ", ", hex(p+size), ") not in usable address space: ", bad, "\n")
			throw("memory reservation exceeds address space limit")
		}
	}

	if uintptr(v)&(heapArenaBytes-1) != 0 {
		throw("misrounded allocation in sysAlloc")
	}

	// Back the reservation.
	sysMap(v, size, &memstats.heap_sys)

mapped:
	// Create arena metadata.
	for ri := arenaIndex(uintptr(v)); ri <= arenaIndex(uintptr(v)+size-1); ri++ {
		l2 := h.arenas[ri.l1()]
		if l2 == nil {
			// Allocate an L2 arena map.
			l2 = (*[1 << arenaL2Bits]*heapArena)(persistentalloc(unsafe.Sizeof(*l2), sys.PtrSize, nil))
			if l2 == nil {
				throw("out of memory allocating heap arena map")
			}
			atomic.StorepNoWB(unsafe.Pointer(&h.arenas[ri.l1()]), unsafe.Pointer(l2))
		}

		if l2[ri.l2()] != nil {
			throw("arena already initialized")
		}
		var r *heapArena
		r = (*heapArena)(h.heapArenaAlloc.alloc(unsafe.Sizeof(*r), sys.PtrSize, &memstats.gc_sys))
		if r == nil {
			r = (*heapArena)(persistentalloc(unsafe.Sizeof(*r), sys.PtrSize, &memstats.gc_sys))
			if r == nil {
				throw("out of memory allocating heap arena metadata")
			}
		}

		// Add the arena to the arenas list.
		if len(h.allArenas) == cap(h.allArenas) {
			size := 2 * uintptr(cap(h.allArenas)) * sys.PtrSize
			if size == 0 {
				size = physPageSize
			}
			newArray := (*notInHeap)(persistentalloc(size, sys.PtrSize, &memstats.gc_sys))
			if newArray == nil {
				throw("out of memory allocating allArenas")
			}
			oldSlice := h.allArenas
			*(*notInHeapSlice)(unsafe.Pointer(&h.allArenas)) = notInHeapSlice{newArray, len(h.allArenas), int(size / sys.PtrSize)}
			copy(h.allArenas, oldSlice)
			// Do not free the old backing array because
			// there may be concurrent readers. Since we
			// double the array each time, this can lead
			// to at most 2x waste.
		}
		h.allArenas = h.allArenas[:len(h.allArenas)+1]
		h.allArenas[len(h.allArenas)-1] = ri

		// Store atomically just in case an object from the
		// new heap arena becomes visible before the heap lock
		// is released (which shouldn't happen, but there's
		// little downside to this).
		atomic.StorepNoWB(unsafe.Pointer(&l2[ri.l2()]), unsafe.Pointer(r))
	}

	// Tell the race detector about the new heap memory.
	if raceenabled {
		racemapshadow(v, size)
	}

	return
}

func sysMap(v unsafe.Pointer, n uintptr, sysStat *uint64) {
	mSysStatInc(sysStat, n)

	p, err := mmap(v, n, _PROT_READ|_PROT_WRITE, _MAP_ANON|_MAP_FIXED|_MAP_PRIVATE, -1, 0)
	if err == _ENOMEM || (GOOS == "solaris" && err == _sunosEAGAIN) {
		throw("runtime: out of memory")
	}
	if p != v || err != 0 {
		throw("runtime: cannot map pages in arena address space")
	}
}
```

对象分配的整个过程如下：

![image-20191111173141574](/Users/dengzhuowen/Library/Application Support/typora-user-images/image-20191111173141574.png)





# 5.对象回收

整个内存分配器的核心是内存复用，不再使用的内存会被放回合适位置，等下次分配时再次使用。只有当空闲内存资源过多时，才会考虑释放。

**基于效率考虑，回收操作自然不会直接顶着单个对象，而是以span为基本单位。通过比对bitmap里的扫描标记，逐步将object收归原span，最终上交central或heap复用。**

回收的主函数在：mgcsweep.go中。

```go
// sweepone sweeps some unswept heap span and returns the number of pages returned
// to the heap, or ^uintptr(0) if there was nothing to sweep.
func sweepone() uintptr {
	_g_ := getg()
	sweepRatio := mheap_.sweepPagesPerByte // For debugging

	// increment locks to ensure that the goroutine is not preempted
	// in the middle of sweep thus leaving the span in an inconsistent state for next GC
	_g_.m.locks++
	if atomic.Load(&mheap_.sweepdone) != 0 {
		_g_.m.locks--
		return ^uintptr(0)
	}
	atomic.Xadd(&mheap_.sweepers, +1)

	// Find a span to sweep.
	var s *mspan
	sg := mheap_.sweepgen
	for {
		s = mheap_.sweepSpans[1-sg/2%2].pop()
		if s == nil {
			atomic.Store(&mheap_.sweepdone, 1)
			break
		}
		if s.state != mSpanInUse {
			// This can happen if direct sweeping already
			// swept this span, but in that case the sweep
			// generation should always be up-to-date.
			if !(s.sweepgen == sg || s.sweepgen == sg+3) {
				print("runtime: bad span s.state=", s.state, " s.sweepgen=", s.sweepgen, " sweepgen=", sg, "\n")
				throw("non in-use span in unswept list")
			}
			continue
		}
		if s.sweepgen == sg-2 && atomic.Cas(&s.sweepgen, sg-2, sg-1) {
			break
		}
	}

	// Sweep the span we found.
	npages := ^uintptr(0)
	if s != nil {
		npages = s.npages
    // 这里调用下一级。
		if s.sweep(false) {
			// Whole span was freed. Count it toward the
			// page reclaimer credit since these pages can
			// now be used for span allocation.
			atomic.Xadduintptr(&mheap_.reclaimCredit, npages)
		} else {
			// Span is still in-use, so this returned no
			// pages to the heap and the span needs to
			// move to the swept in-use list.
			npages = 0
		}
	}

	// Decrement the number of active sweepers and if this is the
	// last one print trace information.
	if atomic.Xadd(&mheap_.sweepers, -1) == 0 && atomic.Load(&mheap_.sweepdone) != 0 {
		if debug.gcpacertrace > 0 {
			print("pacer: sweep done at heap size ", memstats.heap_live>>20, "MB; allocated ", (memstats.heap_live-mheap_.sweepHeapLiveBasis)>>20, "MB during sweep; swept ", mheap_.pagesSwept, " pages at ", sweepRatio, " pages/byte\n")
		}
	}
	_g_.m.locks--
	return npages
}
```

下一级的函数是`mspan.sweep`：

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
					mbits.setMarkedNonAtomic()
					hasFin = true
					break
				}
			}
			// Pass 2: queue all finalizers _or_ handle profile record.
			for special != nil && uintptr(special.offset) < endOffset {
				// Find the exact byte for which the special was setup
				// (as opposed to object beginning).
				p := s.base() + uintptr(special.offset)
				if special.kind == _KindSpecialFinalizer || !hasFin {
					// Splice out special record.
					y := special
					special = special.next
					*specialp = special
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

  // 回收内存
  // 小对象：如果没有可回收object，那么维持原状态，根本无须处理
  // 大对象：整个span就是一个object,直接交还heap
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

central的`freeSpan`操作，如下：

```go
func (c *mcentral) freeSpan(s *mspan, preserve bool, wasempty bool) bool {
	if sg := mheap_.sweepgen; s.sweepgen == sg+1 || s.sweepgen == sg+3 {
		throw("freeSpan given cached span")
	}
	s.needzero = 1

  // 阻止进一步回收
	if preserve {
		// preserve is set only when called from (un)cacheSpan above,
		// the span must be in the empty list.
		if !s.inList() {
			throw("can't preserve unlinked span")
		}
		atomic.Store(&s.sweepgen, mheap_.sweepgen)
		return false
	}

	lock(&c.lock)

	// Move to nonempty if necessary.
  // 将原本为空的span转移到central.nonempty链表
	if wasempty {
		c.empty.remove(s)
		c.nonempty.insert(s)
	}

	// delay updating sweepgen until here. This is the signal that
	// the span may be used in an mcache, so it must come after the
	// linked list operations above (actually, just after the
	// lock of c above.)
	atomic.Store(&s.sweepgen, mheap_.sweepgen)

	if s.allocCount != 0 {
		unlock(&c.lock)
		return false
	}

  // 如果收回全部object，就从central交还给heap
	c.nonempty.remove(s)
	unlock(&c.lock)
	mheap_.freeSpan(s, false)
	return true
}
```

再看看对的回收操作：

```go
func (h *mheap) freeSpan(s *mspan, large bool) {
	systemstack(func() {
		mp := getg().m
		lock(&h.lock)
		memstats.heap_scan += uint64(mp.mcache.local_scan)
		mp.mcache.local_scan = 0
		memstats.tinyallocs += uint64(mp.mcache.local_tinyallocs)
		mp.mcache.local_tinyallocs = 0
		if msanenabled {
			// Tell msan that this entire span is no longer in use.
			base := unsafe.Pointer(s.base())
			bytes := s.npages << _PageShift
			msanfree(base, bytes)
		}
		if large {
			// Match accounting done in mheap.alloc.
			memstats.heap_objects--
		}
		if gcBlackenEnabled != 0 {
			// heap_scan changed.
			gcController.revise()
		}
		h.freeSpanLocked(s, true, true, 0)
		unlock(&h.lock)
	})
}


func (h *mheap) freeSpanLocked(s *mspan, acctinuse, acctidle bool, unusedsince int64) {
	switch s.state {
	case mSpanManual:
		if s.allocCount != 0 {
			throw("mheap.freeSpanLocked - invalid stack free")
		}
	case mSpanInUse:
		if s.allocCount != 0 || s.sweepgen != h.sweepgen {
			print("mheap.freeSpanLocked - span ", s, " ptr ", hex(s.base()), " allocCount ", s.allocCount, " sweepgen ", s.sweepgen, "/", h.sweepgen, "\n")
			throw("mheap.freeSpanLocked - invalid free")
		}
		h.pagesInUse -= uint64(s.npages)

		// Clear in-use bit in arena page bitmap.
		arena, pageIdx, pageMask := pageIndexOf(s.base())
		arena.pageInUse[pageIdx] &^= pageMask
	default:
		throw("mheap.freeSpanLocked - invalid span state")
	}

	if acctinuse {
		memstats.heap_inuse -= uint64(s.npages << _PageShift)
	}
	if acctidle {
		memstats.heap_idle += uint64(s.npages << _PageShift)
	}
	s.state = mSpanFree

	// Stamp newly unused spans. The scavenger will use that
	// info to potentially give back some pages to the OS.
	s.unusedsince = unusedsince
	if unusedsince == 0 {
		s.unusedsince = nanotime()
	}

	// Coalesce span with neighbors.
	h.coalesce(s)

	// Insert s into the appropriate treap.
	if s.scavenged {
		h.scav.insert(s)
	} else {
		h.free.insert(s)
	}
}
```

+ 无论是向操作系统申请内存，还是清理回收内存，只要往heap里放span，都会尝试合并左右相邻的闲置span，以构成更大的自由块。
+ 这些被回收的span并不会被释放，而是等待复用。





# 6.对象释放

在运行时，入口函数`main.main`里，会专门启动一个监控任务`sysmon`，它每隔一段时间就会检查heap里的闲置内存块。

位于`runtime/proc.go`中，

```go
func sysmon() {
	lock(&sched.lock)
	sched.nmsys++
	checkdead()
	unlock(&sched.lock)

	// If a heap span goes unused for 5 minutes after a garbage collection,
	// we hand it back to the operating system.
	scavengelimit := int64(5 * 60 * 1e9)

	if debug.scavenge > 0 {
		// Scavenge-a-lot for testing.
		forcegcperiod = 10 * 1e6
		scavengelimit = 20 * 1e6
	}

	lastscavenge := nanotime()
	nscavenge := 0

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
		if debug.schedtrace <= 0 && (sched.gcwaiting != 0 || atomic.Load(&sched.npidle) == uint32(gomaxprocs)) {
			lock(&sched.lock)
			if atomic.Load(&sched.gcwaiting) != 0 || atomic.Load(&sched.npidle) == uint32(gomaxprocs) {
				atomic.Store(&sched.sysmonwait, 1)
				unlock(&sched.lock)
				// Make wake-up period small enough
				// for the sampling to be correct.
				maxsleep := forcegcperiod / 2
				if scavengelimit < forcegcperiod {
					maxsleep = scavengelimit / 2
				}
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
		// scavenge heap once in a while
    //检查heap里的闲置内存块
		if lastscavenge+scavengelimit/2 < now {  // 到时间了
			mheap_.scavenge(int32(nscavenge), uint64(now), uint64(scavengelimit))
			lastscavenge = now
			nscavenge++
		}
		if debug.schedtrace > 0 && lasttrace+int64(debug.schedtrace)*1000000 <= now {
			lasttrace = now
			schedtrace(debug.scheddetail > 0)
		}
	}
}
```

再看堆的操作：

```go
func (h *mheap) scavenge(k int32, now, limit uint64) {
	// Disallow malloc or panic while holding the heap lock. We do
	// this here because this is an non-mallocgc entry-point to
	// the mheap API.
	gp := getg()
	gp.m.mallocing++
	lock(&h.lock)
	released := h.scavengeAll(now, limit)
	unlock(&h.lock)
	gp.m.mallocing--

	if debug.gctrace > 0 {
		if released > 0 {
			print("scvg", k, ": ", released>>20, " MB released\n")
		}
		print("scvg", k, ": inuse: ", memstats.heap_inuse>>20, ", idle: ", memstats.heap_idle>>20, ", sys: ", memstats.heap_sys>>20, ", released: ", memstats.heap_released>>20, ", consumed: ", (memstats.heap_sys-memstats.heap_released)>>20, " (MB)\n")
	}
}

func (h *mheap) scavengeAll(now, limit uint64) uintptr {
	// Iterate over the treap scavenging spans if unused for at least limit time.
	released := uintptr(0)
	for t := h.free.start(); t.valid(); {
		s := t.span()
		n := t.next()
    // 检查闲置时间是否超出限制，而且内存没有全部被释放过
    // 因为存在span合并的情况，所以有局部释放很正常。
		if (now - uint64(s.unusedsince)) > limit {
			r := s.scavenge()
			if r != 0 {
				h.free.erase(t)
				// Now that s is scavenged, we must eagerly coalesce it
				// with its neighbors to prevent having two spans with
				// the same scavenged state adjacent to each other.
				h.coalesce(s)
				h.scav.insert(s)
				released += r
			}
		}
		t = n
	}
	return released
}


func (s *mspan) scavenge() uintptr {
	// start and end must be rounded in, otherwise madvise
	// will round them *out* and release more memory
	// than we want.
	start, end := s.physPageBounds()
	if end <= start {
		// start and end don't span a whole physical page.
		return 0
	}
	released := end - start
	memstats.heap_released += uint64(released)
	s.scavenged = true
	sysUnused(unsafe.Pointer(start), released)
	return released
}

func sysUnused(v unsafe.Pointer, n uintptr) {
	madvise(v, n, _MADV_FREE)
}
```

所谓的物理内存释放，本质上是调用`madvise`系统api，告知操作系统某段内存暂不使用，建议内核收回对应的物理内存。当然，这只是一个建议，是否回收由内核决定。如物理内存资源充足，该建议可能会被忽略，以避免无效的损耗。而当再次使用该内存块时，会引发缺页异常，内核会自动重新关联物理内存页。

