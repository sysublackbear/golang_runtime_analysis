# 14.Map

## 14.1.内存模型

在源码中，表示 map 的结构体是 hmap，它是 hashmap 的缩写：

```go
// A header for a Go map.
type hmap struct {
	// Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
	// Make sure this stays in sync with the compiler's definition.
  // 元素个数，调用 len(map) 时，直接返回此值
	count     int // # live cells == size of map.  Must be first (used by len() builtin)
	flags     uint8
  // buckets 的对数 log_2
	B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
  // overflow 的 bucket近似数
	noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
  // 计算key的哈希的时候会传入哈希函数
	hash0     uint32 // hash seed

  // 指向buckets数组，大小为2^B
  // 如果元素个数为0，就为 nil
	buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
  // 扩容的时候，buckets长度会是oldbuckets的两倍
	oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
  // 指示扩容进度，小于此地址的buckets迁移完成
	nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

	extra *mapextra // optional fields
}
```

`B`是buckets数组的长度的对数，也就是说buckets数组的长度就是`2^B`。

`buckets`是一个指针，最终它指向的是一个结构体：

```go
// A bucket for a Go map.
type bmap struct {
	// tophash generally contains the top byte of the hash value
	// for each key in this bucket. If tophash[0] < minTopHash,
	// tophash[0] is a bucket evacuation state instead.
	tophash [bucketCnt]uint8
	// Followed by bucketCnt keys and then bucketCnt elems.
	// NOTE: packing all the keys together and then all the elems together makes the
	// code a bit more complicated than alternating key/elem/key/elem/... but it allows
	// us to eliminate padding which would be needed for, e.g., map[int64]int8.
	// Followed by an overflow pointer.
}
```

但这只是表面的结构，编译期间会给它加料，动态地创建一个新的结构：

```go
type bmap struct {
    topbits  [8]uint8
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr
}
```

`bmap` 就是我们常说的“桶”，桶里面会最多装 8 个 key，这些 key 之所以会落入同一个桶，是因为它们经过哈希计算后，哈希结果是“一类”的。在桶内，又会根据 key 计算出来的 hash 值的高 8 位来决定 key 到底落入桶内的哪个位置（一个桶内最多有8个位置）。



整体图如下：

![hashmap bmap](https://user-images.githubusercontent.com/7698088/57576986-acd87600-749f-11e9-8710-75e423c7efdb.png)

**还有一个特殊的点**：当 map 的 key 和 value 都不是指针，并且 size 都小于 128 字节的情况下，会把 bmap 标记为不含指针，这样可以避免 gc 时扫描整个 hmap。但是，我们看 bmap 其实有一个 overflow 的字段，是指针类型的，破坏了 bmap 不含指针的设想，这时会把 overflow 移动到 extra 字段来。



## 14.2.bmap的内部组成

![image-20200106110832576.png](https://github.com/sysublackbear/golang_runtime_analysis/blob/master/img/image-20200106110832576.png)

上图就是 bucket 的内存模型，`HOB Hash` 指的就是 top hash。 注意到 key 和 value 是各自放在一起的，并不是 `key/value/key/value/...` 这样的形式。源码里说明这样的好处是在某些情况下可以省略掉 padding 字段，节省内存空间。



如果按照 `key/value/key/value/...` 这样的模式存储，那在每一个 key/value 对之后都要额外 padding 7 个字节；而将所有的 key，value 分别绑定到一起，这种形式 `key/key/.../value/value/...`，则只需要在最后添加 padding。

每个 bucket 设计成最多只能放 8 个 key-value 对，如果有第 9 个 key-value 落入当前的 bucket，那就需要再构建一个 bucket ，通过 `overflow` 指针连接起来。





## 14.3.创建 map

通过汇编语言可以看到，实际上底层调用的是`makemap`函数，主要做的工作就是初始化`hmap`结构体的各种字段，例如计算B的大小，设置哈希种子`hash0`等等。

```go
// makemap implements Go map creation for make(map[k]v, hint).
// If the compiler has determined that the map or the first bucket
// can be created on the stack, h and/or bucket may be non-nil.
// If h != nil, the map can be created directly in h.
// If h.buckets != nil, bucket pointed to can be used as the first bucket.
func makemap(t *maptype, hint int, h *hmap) *hmap {
  // 各种条件要检查
	mem, overflow := math.MulUintptr(uintptr(hint), t.bucket.size)
	if overflow || mem > maxAlloc {
		hint = 0
	}

	// initialize Hmap
  // 初始化hmap
	if h == nil {
		h = new(hmap)
	}
	h.hash0 = fastrand()

	// Find the size parameter B which will hold the requested # of elements.
	// For hint < 0 overLoadFactor returns false since hint < bucketCnt.
  // 找到这样的一个B，使得map的装载因子在正常范围内。
	B := uint8(0)
	for overLoadFactor(hint, B) {
		B++
	}
	h.B = B

	// allocate initial hash table
	// if B == 0, the buckets field is allocated lazily later (in mapassign)
	// If hint is large zeroing this memory could take a while.
  // 初始化hash table
  // 如果B等于0，那么buckets就会在赋值的时候再分配
  // 如果长度比较大，分配内存会花费长一点
	if h.B != 0 {
		var nextOverflow *bmap
		h.buckets, nextOverflow = makeBucketArray(t, h.B, nil)
		if nextOverflow != nil {
			h.extra = new(mapextra)
			h.extra.nextOverflow = nextOverflow
		}
	}

	return h
}

// overLoadFactor reports whether count items placed in 1<<B buckets is over loadFactor.
func overLoadFactor(count int, B uint8) bool {
	return count > bucketCnt && uintptr(count) > loadFactorNum*(bucketShift(B)/loadFactorDen)
}
```



## 14.4.hash函数

map 的一个关键点在于，哈希函数的选择。在程序启动时，会检测 cpu 是否支持 aes，如果支持，则使用 aes hash，否则使用 memhash。这是在函数 `alginit()` 中完成，位于路径：`src/runtime/alg.go` 下。代码如下：

```go
func alginit() {
	// Install AES hash algorithms if the instructions needed are present.
	if (GOARCH == "386" || GOARCH == "amd64") &&
		GOOS != "nacl" &&
		cpu.X86.HasAES && // AESENC
		cpu.X86.HasSSSE3 && // PSHUFB
		cpu.X86.HasSSE41 { // PINSR{D,Q}
		initAlgAES()
		return
	}
	if GOARCH == "arm64" && cpu.ARM64.HasAES {
		initAlgAES()
		return
	}
	getRandomData((*[len(hashkey) * sys.PtrSize]byte)(unsafe.Pointer(&hashkey))[:])
	hashkey[0] |= 1 // make sure these numbers are odd
	hashkey[1] |= 1
	hashkey[2] |= 1
	hashkey[3] |= 1
}
```

之前讲过的表示类型的结构体：

```go
type _type struct {
	size       uintptr
	ptrdata    uintptr // size of memory prefix holding all pointers
	hash       uint32
	tflag      tflag
	align      uint8
	fieldalign uint8
	kind       uint8
	alg        *typeAlg
	// gcdata stores the GC type data for the garbage collector.
	// If the KindGCProg bit is set in kind, gcdata is a GC program.
	// Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
	gcdata    *byte
	str       nameOff
	ptrToThis typeOff
}
```

其中 `alg` 字段就和哈希相关，它是指向如下结构体的指针：

```go
// typeAlg is also copied/used in reflect/type.go.
// keep them in sync.
type typeAlg struct {
	// function for hashing objects of this type
	// (ptr to object, seed) -> hash
	hash func(unsafe.Pointer, uintptr) uintptr
	// function for comparing objects of this type
	// (ptr to object A, ptr to object B) -> ==?
	equal func(unsafe.Pointer, unsafe.Pointer) bool
}
```



## 14.5.key的定位过程

![mapacess](https://user-images.githubusercontent.com/7698088/57577721-faf57580-74af-11e9-8826-aacdb34a1d2b.png)



**查找过程**：

上图中，假定 B = 5，所以 bucket 总数就是 2^5 = 32。首先计算出待查找 key 的哈希，使用低 5 位 `00110`，找到对应的 6 号 bucket，使用高 8 位 `10010111`，对应十进制 151，在 6 号 bucket 中寻找 tophash 值（HOB hash）为 151 的 key（通过遍历bucket里面key的方式），找到了 2 号槽位，这样整个查找过程就结束了。

如果在 bucket 中没找到，并且 overflow 不为空，还要继续去 overflow bucket 中寻找，直到找到或是所有的 key 槽位都找遍了，包括所有的 overflow bucket。



查找某个key的底层函数是`mapaccess1`，代码如下：

```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		pc := funcPC(mapaccess1)
		racereadpc(unsafe.Pointer(h), callerpc, pc)
		raceReadObjectPC(t.key, key, callerpc, pc)
	}
	if msanenabled && h != nil {
		msanread(key, t.key.size)
	}
  // 如果h什么都没有，返回零值
	if h == nil || h.count == 0 {
		if t.hashMightPanic() {
			t.key.alg.hash(key, 0) // see issue 23734
		}
		return unsafe.Pointer(&zeroVal[0])
	}
  // 写和读冲突
	if h.flags&hashWriting != 0 {
		throw("concurrent map read and map write")
	}
  // 不同类型key使用的hash算法在编译期确定
	alg := t.key.alg
  // 计算哈希值，并且加入hash0引入随机性
	hash := alg.hash(key, uintptr(h.hash0))
  // 比如 B=5,那m就是31，二进制全是1
  // 求 bucket num时，将hash与m相与
  // 达到 bucket num 由 hash 的低8位决定的效果
	m := bucketMask(h.B)
  
  // b就是bucket的地址
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
  
  // oldbuckets不为nil，说明发生了扩容
	if c := h.oldbuckets; c != nil {
    // 如果不是同size扩容
    // 对应条件1的解决方案
		if !h.sameSizeGrow() {
			// There used to be half as many buckets; mask down one more power of two.
      // 新bucket数量是老的两倍
			m >>= 1
		}
    
    // 求出key在老的map中的bucket位置
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
    
    // 如果oldb没有搬迁到新的bucket
    // 那就在老的bucket中寻找
		if !evacuated(oldb) {
			b = oldb
		}
	}
  
  // 计算出高8位的hash
  // 相当于右移56位，只取高8位
	top := tophash(hash)
bucketloop:
  // 遍历bucket，进而遍历overflow逻辑
	for ; b != nil; b = b.overflow(t) {
		for i := uintptr(0); i < bucketCnt; i++ {
			if b.tophash[i] != top {
				if b.tophash[i] == emptyRest {
					break bucketloop
				}
				continue
			}
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			if t.indirectkey() {
				k = *((*unsafe.Pointer)(k))
			}
			if alg.equal(key, k) {
				e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
				if t.indirectelem() {
					e = *((*unsafe.Pointer)(e))
				}
				return e
			}
		}
	}
	return unsafe.Pointer(&zeroVal[0])
}

func evacuated(b *bmap) bool {
	h := b.tophash[0]
	return h > emptyOne && h < minTopHash
}
```

其中key和value的定位公式分别为：

```go
// key
k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))

// value
e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.elemsize))
```



## 14.6.map的两种get操作

Go 语言中读取 map 有两种语法：带 comma 和 不带 comma。当要查询的 key 不在 map 里，带 comma 的用法会返回一个 bool 型变量提示 key 是否在 map 中；而不带 comma 的语句则会返回一个 value 类型的零值。如果 value 是 int 型就会返回 0，如果 value 是 string 类型，就会返回空字符串。

分别对应到底层的两个不同的函数：`mapaccess1`和`mapaccess2`。



## 14.7.bucket的扩张

bucket的扩张由函数`hasGrow`来实现。代码如下：

```go
func hashGrow(t *maptype, h *hmap) {
	// If we've hit the load factor, get bigger.
	// Otherwise, there are too many overflow buckets,
	// so keep the same number of buckets and "grow" laterally.
  // B+1相当于是原来的2倍空间
	bigger := uint8(1)
  
  // 如果仅仅是由于大量写入又大量删除而导致overflow bucket增多而需要扩张
  // 其实就是bucket之间排列稀疏，不够紧密。
	if !overLoadFactor(h.count+1, h.B) {
    // 进行等量的内存扩容，所以B保持不变
		bigger = 0
		h.flags |= sameSizeGrow
	}
  // 将老buckets挂到buckets上
	oldbuckets := h.buckets
  // 申请新的buckets空间
	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger, nil)

	flags := h.flags &^ (iterator | oldIterator)
	if h.flags&iterator != 0 {
		flags |= oldIterator
	}
	// commit the grow (atomic wrt gc)
  // 提交grow的动作
	h.B += bigger
	h.flags = flags
	h.oldbuckets = oldbuckets
	h.buckets = newbuckets
  // 搬迁进度为0
	h.nevacuate = 0
  // overflow buckets数量为0
	h.noverflow = 0

	if h.extra != nil && h.extra.overflow != nil {
		// Promote current overflow buckets to the old generation.
		if h.extra.oldoverflow != nil {
			throw("oldoverflow is not nil")
		}
		h.extra.oldoverflow = h.extra.overflow
		h.extra.overflow = nil
	}
	if nextOverflow != nil {
		if h.extra == nil {
			h.extra = new(mapextra)
		}
		h.extra.nextOverflow = nextOverflow
	}

	// the actual copying of the hash table data is done incrementally
	// by growWork() and evacuate().
}
```

上面说的 `hashGrow()` 函数实际上并没有真正地“搬迁”，它只是分配好了新的 buckets，并将老的 buckets 挂到了 oldbuckets 字段上。真正搬迁 buckets 的动作在 `growWork()` 函数中，而调用 `growWork()` 函数的动作是在 mapassign 和 mapdelete 函数中。也就是插入或修改、删除 key 的时候，都会尝试进行搬迁 buckets 的工作。先检查 oldbuckets 是否搬迁完毕，具体来说就是检查 oldbuckets 是否为 nil。



然后看看真正执行迁移工作的`growWork()`函数。（由`mapassign`和`mapdelete`里面进行调用）

```go
func growWork(t *maptype, h *hmap, bucket uintptr) {
	// make sure we evacuate the oldbucket corresponding
	// to the bucket we're about to use
  // 确认搬迁老的bucket对应正在使用的bucket
	evacuate(t, h, bucket&h.oldbucketmask())

	// evacuate one more oldbucket to make progress on growing
  // 再搬迁一个bucket，以加快搬迁进程
	if h.growing() {
		evacuate(t, h, h.nevacuate)
	}
}

// growing reports whether h is growing. The growth may be to the same size or bigger.
func (h *hmap) growing() bool {
  // 如果 oldbuckets 不为空，说明还没有搬迁完毕，还得继续搬。
	return h.oldbuckets != nil
}

// evacDst is an evacuation destination.
type evacDst struct {
  // 表示bucket的移动的目标地址
	b *bmap          // current destination bucket
  // index下标，区分出x和y
	i int            // key/elem index into b
  // 指向key的指针
	k unsafe.Pointer // pointer to current key storage
  // 指向value的指针
	e unsafe.Pointer // pointer to current elem storage
}

func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
  // 定位老的bucket地址
	b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
  // 结果为2^B
	newbit := h.noldbuckets()
	if !evacuated(b) {
		// TODO: reuse overflow buckets instead of using new ones, if there
		// is no iterator using the old buckets.  (If !oldIterator.)

		// xy contains the x and y (low and high) evacuation destinations.
		var xy [2]evacDst
		x := &xy[0]
    // 默认是等size扩容，前后bucket序号不变
    // 使用x进行搬迁
		x.b = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
		x.k = add(unsafe.Pointer(x.b), dataOffset)
		x.e = add(x.k, bucketCnt*uintptr(t.keysize))

    // 如果不是等size扩容，前后bucket序号有变
    // 使用y来进行搬迁
		if !h.sameSizeGrow() {
			// Only calculate y pointers if we're growing bigger.
			// Otherwise GC can see bad pointers.
			y := &xy[1]
      // y代表的bucket序号增加了2^B
			y.b = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
			y.k = add(unsafe.Pointer(y.b), dataOffset)
			y.e = add(y.k, bucketCnt*uintptr(t.keysize))
		}

    // 遍历所有的bucket，包括overflow buckets
    // b是老的bucket地址
		for ; b != nil; b = b.overflow(t) {
			k := add(unsafe.Pointer(b), dataOffset)
			e := add(k, bucketCnt*uintptr(t.keysize))
      // 遍历bucket中的所有cell
			for i := 0; i < bucketCnt; i, k, e = i+1, add(k, uintptr(t.keysize)), add(e, uintptr(t.elemsize)) {
        // 当前cell的top hash值
				top := b.tophash[i]
        // 如果cell为空，即没有key
				if isEmpty(top) {
          // 那就标志它被”搬迁”过
					b.tophash[i] = evacuatedEmpty
          // 继续下个cell
					continue
				}
        
        // 正常不会出现这种情况
        // 未被搬迁的 cell 只可能是 empty 或是正常的 top hash（大于 minTopHash）
				if top < minTopHash {
					throw("bad map state")
				}
				k2 := k
        // 如果key是指针，则解引用
				if t.indirectkey() {
					k2 = *((*unsafe.Pointer)(k2))
				}
				var useY uint8
        // 如果不是等量扩容
				if !h.sameSizeGrow() {
					// Compute hash to make our evacuation decision (whether we need
					// to send this key/elem to bucket x or bucket y).
          // 计算hash值，和key第一次写入时一样
					hash := t.key.alg.hash(k2, uintptr(h.hash0))
          
          // 如果有协程正在遍历map && 如果出现相同的key值，算出来的hash值不同
					if h.flags&iterator != 0 && !t.reflexivekey() && !t.key.alg.equal(k2, k2) {
						// If key != key (NaNs), then the hash could be (and probably
						// will be) entirely different from the old hash. Moreover,
						// it isn't reproducible. Reproducibility is required in the
						// presence of iterators, as our evacuation decision must
						// match whatever decision the iterator made.
						// Fortunately, we have the freedom to send these keys either
						// way. Also, tophash is meaningless for these kinds of keys.
						// We let the low bit of tophash drive the evacuation decision.
						// We recompute a new random tophash for the next level so
						// these keys will get evenly distributed across all buckets
						// after multiple grows.
						useY = top & 1
						top = tophash(hash)
					} else {
						if hash&newbit != 0 {
							useY = 1
						}
					}
				}

				if evacuatedX+1 != evacuatedY || evacuatedX^1 != evacuatedY {
					throw("bad evacuatedN")
				}

				b.tophash[i] = evacuatedX + useY // evacuatedX + 1 == evacuatedY
				dst := &xy[useY]                 // evacuation destination

				if dst.i == bucketCnt {
					dst.b = h.newoverflow(t, dst.b)
					dst.i = 0
					dst.k = add(unsafe.Pointer(dst.b), dataOffset)
					dst.e = add(dst.k, bucketCnt*uintptr(t.keysize))
				}
				dst.b.tophash[dst.i&(bucketCnt-1)] = top // mask dst.i as an optimization, to avoid a bounds check
				if t.indirectkey() {
					*(*unsafe.Pointer)(dst.k) = k2 // copy pointer
				} else {
					typedmemmove(t.key, dst.k, k) // copy elem
				}
				if t.indirectelem() {
          // 将原 key（是指针）复制到新位置
					*(*unsafe.Pointer)(dst.e) = *(*unsafe.Pointer)(e)
				} else {
          // 将原 key（是值）复制到新位置
					typedmemmove(t.elem, dst.e, e)
				}
        // 定位到下一个cell
				dst.i++
				// These updates might push these pointers past the end of the
				// key or elem arrays.  That's ok, as we have the overflow pointer
				// at the end of the bucket to protect against pointing past the
				// end of the bucket.
				dst.k = add(dst.k, uintptr(t.keysize))
				dst.e = add(dst.e, uintptr(t.elemsize))
			}
		}
		// Unlink the overflow buckets & clear key/elem to help GC.
    // 如果没有协程在使用老的buckets,就把老buckets清除掉，帮助gc。
		if h.flags&oldIterator == 0 && t.bucket.ptrdata != 0 {
			b := add(h.oldbuckets, oldbucket*uintptr(t.bucketsize))
			// Preserve b.tophash because the evacuation
			// state is maintained there.
      // 只清除bucket 的 key,value 部分，保留 top hash 部分，指示搬迁状态
			ptr := add(b, dataOffset)
			n := uintptr(t.bucketsize) - dataOffset
			memclrHasPointers(ptr, n)
		}
	}

  // 更新搬迁进度
  // 如果此次搬迁的 bucket 等于当前进度
	if oldbucket == h.nevacuate {
		advanceEvacuationMark(h, t, newbit)
	}
}


func advanceEvacuationMark(h *hmap, t *maptype, newbit uintptr) {
  // 进度加1
	h.nevacuate++
	// Experiments suggest that 1024 is overkill by at least an order of magnitude.
	// Put it in there as a safeguard anyway, to ensure O(1) behavior.
  // 尝试往后看1024个bucket
	stop := h.nevacuate + 1024
	if stop > newbit {
		stop = newbit
	}
  // 寻找没有搬迁的bucket
	for h.nevacuate != stop && bucketEvacuated(t, h, h.nevacuate) {
		h.nevacuate++
	}
  
  // 现在h.nevacuate之前的bucket都被搬迁完毕
  
  // 所有的buckets搬迁完毕
	if h.nevacuate == newbit { // newbit == # of oldbuckets
		// Growing is all done. Free old main bucket array.
    // 清除老的buckets
		h.oldbuckets = nil
		// Can discard old overflow buckets as well.
		// If they are still referenced by an iterator,
		// then the iterator holds a pointers to the slice.
    // 清除老的overflow bucket
		if h.extra != nil {
			h.extra.oldoverflow = nil
		}
    // 清除正在扩容的标志位
		h.flags &^= sameSizeGrow
	}
}
```

源码里提到 X, Y part，其实就是我们说的如果是扩容到原来的 2 倍，桶的数量是原来的 2 倍，前一半桶被称为 X part，后一半桶被称为 Y part。一个 bucket 中的 key 可能会分裂落到 2 个桶，一个位于 X part，一个位于 Y part。所以在搬迁一个 cell 之前，需要知道这个 cell 中的 key 是落到哪个 Part。很简单，重新计算 cell 中 key 的 hash，并向前“多看”一位，决定落入哪个 Part，这个前面也说得很详细了。



**问题**：为什么遍历map是无序的？

map 在扩容后，会发生 key 的搬迁，原来落在同一个 bucket 中的 key，搬迁后，有些 key 就要远走高飞了（bucket 序号加上了 2^B）。而遍历的过程，就是按顺序遍历 bucket，同时按顺序遍历 bucket 中的 key。搬迁后，key 的位置发生了重大的变化，有些 key 飞上高枝，有些 key 则原地不动。这样，遍历 map 的结果就不可能按原来的顺序了。

**问题**：当然，如果我就一个 hard code 的 map，我也不会向 map 进行插入删除的操作，按理说每次遍历这样的 map 都会返回一个固定顺序的 key/value 序列吧。的确是这样，但是 Go 杜绝了这种做法，因为这样会给新手程序员带来误解，以为这是一定会发生的事情，在某些情况下，可能会酿成大错。

当然，Go 做得更绝，当我们在遍历 map 时，并不是固定地从 0 号 bucket 开始遍历，每次都是从一个**随机值序号的 bucket** 开始遍历，并且是从这个 bucket 的一个随机序号的 cell 开始遍历。这样，即使你是一个写死的 map，仅仅只是遍历它，也不太可能会返回一个固定序列的 key/value 对了。





## 14.8.map的遍历

这样，关于 map 迭代，底层的函数调用关系一目了然。先是调用 `mapiterinit` 函数初始化迭代器，然后循环调用 `mapiternext` 函数进行 map 迭代。

![map iter loop](https://user-images.githubusercontent.com/7698088/57976471-ad2ebf00-7a13-11e9-8dd8-d7be54f96440.png)

迭代器的数据结构如下：

```go
// A hash iteration structure.
// If you modify hiter, also change cmd/compile/internal/gc/reflect.go to indicate
// the layout of this structure.
type hiter struct {
  // key指针
	key         unsafe.Pointer // Must be in first position.  Write nil to indicate iteration end (see cmd/internal/gc/range.go).
  // value指针
	elem        unsafe.Pointer // Must be in second position (see cmd/internal/gc/range.go).
	// map类型，包含如key size 大小等
  t           *maptype
  // map header
	h           *hmap
  // 初始化时指向的bucket
	buckets     unsafe.Pointer // bucket ptr at hash_iter initialization time
  // 当前遍历到的bmap
	bptr        *bmap          // current bucket
	overflow    *[]*bmap       // keeps overflow buckets of hmap.buckets alive
	oldoverflow *[]*bmap       // keeps overflow buckets of hmap.oldbuckets alive
  // 起始遍历的bucket编号
	startBucket uintptr        // bucket iteration started at
  // 遍历开始时cell的编号（每个bucket中有8个cell）
	offset      uint8          // intra-bucket offset to start from during iteration (should be big enough to hold bucketCnt-1)
  // 是否从头遍历了
	wrapped     bool           // already wrapped around from end of bucket array to beginning
	// B的大小
  B           uint8
	// 指示当前cell序号
  i           uint8
	// 指示当前的bucket
  bucket      uintptr
	// 因为扩容，需要检查的bucket
  checkBucket uintptr
}
```

`mapiterinit` 就是对 hiter 结构体里的字段进行初始化赋值操作。

```go
// mapiterinit initializes the hiter struct used for ranging over maps.
// The hiter struct pointed to by 'it' is allocated on the stack
// by the compilers order pass or on the heap by reflect_mapiterinit.
// Both need to have zeroed hiter since the struct contains pointers.
func mapiterinit(t *maptype, h *hmap, it *hiter) {
	if raceenabled && h != nil {
		callerpc := getcallerpc()
		racereadpc(unsafe.Pointer(h), callerpc, funcPC(mapiterinit))
	}

	if h == nil || h.count == 0 {
		return
	}

	if unsafe.Sizeof(hiter{})/sys.PtrSize != 12 {
		throw("hash_iter size incorrect") // see cmd/compile/internal/gc/reflect.go
	}
	it.t = t
	it.h = h

	// grab snapshot of bucket state
	it.B = h.B
	it.buckets = h.buckets
	if t.bucket.ptrdata == 0 {
		// Allocate the current slice and remember pointers to both current and old.
		// This preserves all relevant overflow buckets alive even if
		// the table grows and/or overflow buckets are added to the table
		// while we are iterating.
		h.createOverflow()
		it.overflow = h.extra.overflow
		it.oldoverflow = h.extra.oldoverflow
	}

	// decide where to start
	r := uintptr(fastrand())
	if h.B > 31-bucketCntBits {
		r += uintptr(fastrand()) << 31
	}
	it.startBucket = r & bucketMask(h.B)
	it.offset = uint8(r >> h.B & (bucketCnt - 1))

	// iterator state
	it.bucket = it.startBucket

	// Remember we have an iterator.
	// Can run concurrently with another mapiterinit().
	if old := h.flags; old&(iterator|oldIterator) != iterator|oldIterator {
		atomic.Or8(&h.flags, iterator|oldIterator)
	}

	mapiternext(it)
}

func mapiternext(it *hiter) {
	h := it.h
	if raceenabled {
		callerpc := getcallerpc()
		racereadpc(unsafe.Pointer(h), callerpc, funcPC(mapiternext))
	}
	if h.flags&hashWriting != 0 {
		throw("concurrent map iteration and map write")
	}
	t := it.t
	bucket := it.bucket
	b := it.bptr
	i := it.i
	checkBucket := it.checkBucket
	alg := t.key.alg

next:
	if b == nil {
		if bucket == it.startBucket && it.wrapped {
			// end of iteration
			it.key = nil
			it.elem = nil
			return
		}
		if h.growing() && it.B == h.B {
			// Iterator was started in the middle of a grow, and the grow isn't done yet.
			// If the bucket we're looking at hasn't been filled in yet (i.e. the old
			// bucket hasn't been evacuated) then we need to iterate through the old
			// bucket and only return the ones that will be migrated to this bucket.
			oldbucket := bucket & it.h.oldbucketmask()
			b = (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
			if !evacuated(b) {
				checkBucket = bucket
			} else {
				b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
				checkBucket = noCheck
			}
		} else {
			b = (*bmap)(add(it.buckets, bucket*uintptr(t.bucketsize)))
			checkBucket = noCheck
		}
		bucket++
		if bucket == bucketShift(it.B) {
			bucket = 0
			it.wrapped = true
		}
		i = 0
	}
	for ; i < bucketCnt; i++ {
		offi := (i + it.offset) & (bucketCnt - 1)
		if isEmpty(b.tophash[offi]) || b.tophash[offi] == evacuatedEmpty {
			// TODO: emptyRest is hard to use here, as we start iterating
			// in the middle of a bucket. It's feasible, just tricky.
			continue
		}
		k := add(unsafe.Pointer(b), dataOffset+uintptr(offi)*uintptr(t.keysize))
		if t.indirectkey() {
			k = *((*unsafe.Pointer)(k))
		}
		e := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+uintptr(offi)*uintptr(t.elemsize))
		if checkBucket != noCheck && !h.sameSizeGrow() {
			// Special case: iterator was started during a grow to a larger size
			// and the grow is not done yet. We're working on a bucket whose
			// oldbucket has not been evacuated yet. Or at least, it wasn't
			// evacuated when we started the bucket. So we're iterating
			// through the oldbucket, skipping any keys that will go
			// to the other new bucket (each oldbucket expands to two
			// buckets during a grow).
			if t.reflexivekey() || alg.equal(k, k) {
				// If the item in the oldbucket is not destined for
				// the current new bucket in the iteration, skip it.
				hash := alg.hash(k, uintptr(h.hash0))
				if hash&bucketMask(it.B) != checkBucket {
					continue
				}
			} else {
				// Hash isn't repeatable if k != k (NaNs).  We need a
				// repeatable and randomish choice of which direction
				// to send NaNs during evacuation. We'll use the low
				// bit of tophash to decide which way NaNs go.
				// NOTE: this case is why we need two evacuate tophash
				// values, evacuatedX and evacuatedY, that differ in
				// their low bit.
				if checkBucket>>(it.B-1) != uintptr(b.tophash[offi]&1) {
					continue
				}
			}
		}
		if (b.tophash[offi] != evacuatedX && b.tophash[offi] != evacuatedY) ||
			!(t.reflexivekey() || alg.equal(k, k)) {
			// This is the golden data, we can return it.
			// OR
			// key!=key, so the entry can't be deleted or updated, so we can just return it.
			// That's lucky for us because when key!=key we can't look it up successfully.
			it.key = k
			if t.indirectelem() {
				e = *((*unsafe.Pointer)(e))
			}
			it.elem = e
		} else {
			// The hash table has grown since the iterator was started.
			// The golden data for this key is now somewhere else.
			// Check the current hash table for the data.
			// This code handles the case where the key
			// has been deleted, updated, or deleted and reinserted.
			// NOTE: we need to regrab the key as it has potentially been
			// updated to an equal() but not identical key (e.g. +0.0 vs -0.0).
			rk, re := mapaccessK(t, h, k)
			if rk == nil {
				continue // key has been deleted
			}
			it.key = rk
			it.elem = re
		}
		it.bucket = bucket
		if it.bptr != b { // avoid unnecessary write barrier; see issue 14921
			it.bptr = b
		}
		it.i = i + 1
		it.checkBucket = checkBucket
		return
	}
	b = b.overflow(t)
	i = 0
	goto next
}
```

![map init](https://user-images.githubusercontent.com/7698088/57980268-a4fa7200-7a5b-11e9-9ad1-fb2b64fe3159.png)

标红的表示起始位置，bucket 遍历顺序为：3 -> 0 -> 1 -> 2（每次随机起点）。



## 14.9.map的赋值

详见`mapassign`。



## 14.10.map的删除

详见`mapdelete`。



## 14.11.map的一些疑难杂症

### 1.可以边遍历边删除吗

map 并不是一个线程安全的数据结构。同时读写一个 map 是未定义的行为，如果被检测到，会直接 panic。

一般而言，这可以通过读写锁来解决：`sync.RWMutex`。

读之前调用 `RLock()` 函数，读完之后调用 `RUnlock()` 函数解锁；写之前调用 `Lock()` 函数，写完之后，调用 `Unlock()` 解锁。

另外，`sync.Map` 是线程安全的 map，也可以使用。



### 2.key 可以是 float 型吗？

从语法上看，是可以的。Go 语言中只要是可比较的类型都可以作为 key。除开 slice，map，functions 这几种类型，其他类型都是 OK 的。具体包括：布尔值、数字、字符串、指针、通道、接口类型、结构体、只包含上述类型的数组。

**当用 float64 作为 key 的时候，先要将其转成 unit64 类型，再插入 key 中**。

**具体是通过 `Float64frombits` 函数完成**。

这些类型的共同特征是支持 `==` 和 `!=` 操作符，`k1 == k2` 时，可认为 k1 和 k2 是同一个 key。如果是结构体，则需要它们的字段值都相等，才被认为是相同的 key。

顺便说一句，任何类型都可以作为 value，包括 map 类型。



**最后说结论：float 型可以作为 key，但是由于精度的问题，会导致一些诡异的问题，慎用之（比如多个NaN作为key的问题）**。
