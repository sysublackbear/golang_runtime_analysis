# 13.Slice

slice的组成如下：

```go
type slice struct {
	array unsafe.Pointer  // 元素指针
	len   int  // 长度
	cap   int  // 容量
}
```



## 13.1.创建方式

1. 直接声明：`var slice []int`；
2. new：`slice := *new([]int)`；
3. 字面量：`slice := []int{1, 2, 3, 4, 5}`；
4. make：`slice := make([]int, 5, 10)`；
5. 从别的切片或数组中截取：`slice := array[1:5]`或`slice := souceSlice[1:5]`。



## 13.2.nil slice和empty slice的差异

+ 通过`var slice []int`创建出来的slice是一个`nil slice`。它的长度和容量都为0。和`nil`比较的结果为`true`；
+ 这里容易混淆的是`empty slice`，它的长度和容量都为0，但是所有空切片的数据指针都指向同一个地址`0xc42003bda0`。空切片和`nil`比较的结果为`false`。



## 13.3. make slice底层

先看一个demo：

```go
package main

import "fmt"

func main() {
    slice := make([]int, 5, 10)
    slice[2] = 2
    fmt.Println(slice)
}
```

然后通过命令：

```bash
go tool compile -S main.go
```

得到对应的汇编代码：

```bash
[sysublackbear@centos-linux zhuowen.deng]$ go tool compile -S test.go
"".main STEXT size=224 args=0x0 locals=0x60
	0x0000 00000 (test.go:5)	TEXT	"".main(SB), $96-0
	0x0000 00000 (test.go:5)	MOVQ	(TLS), CX
	0x0009 00009 (test.go:5)	CMPQ	SP, 16(CX)
	0x000d 00013 (test.go:5)	JLS	214
	0x0013 00019 (test.go:5)	SUBQ	$96, SP
	0x0017 00023 (test.go:5)	MOVQ	BP, 88(SP)
	0x001c 00028 (test.go:5)	LEAQ	88(SP), BP
	0x0021 00033 (test.go:5)	FUNCDATA	$0, gclocals·69c1753bd5f81501d95132d08af04464(SB)
	0x0021 00033 (test.go:5)	FUNCDATA	$1, gclocals·57cc5e9a024203768cbab1c731570886(SB)
	0x0021 00033 (test.go:6)	LEAQ	type.int(SB), AX
	0x0028 00040 (test.go:6)	MOVQ	AX, (SP)
	0x002c 00044 (test.go:6)	MOVQ	$5, 8(SP)
	0x0035 00053 (test.go:6)	MOVQ	$10, 16(SP)
	0x003e 00062 (test.go:6)	PCDATA	$0, $0
	0x003e 00062 (test.go:6)	CALL	runtime.makeslice(SB)
	0x0043 00067 (test.go:6)	MOVQ	32(SP), AX
	0x0048 00072 (test.go:6)	MOVQ	24(SP), CX
	0x004d 00077 (test.go:6)	MOVQ	40(SP), DX
	0x0052 00082 (test.go:7)	CMPQ	AX, $2
	0x0056 00086 (test.go:7)	JLS	207
	0x0058 00088 (test.go:7)	MOVQ	$2, 16(CX)
	0x0060 00096 (test.go:8)	MOVQ	CX, ""..autotmp_2+64(SP)
	0x0065 00101 (test.go:8)	MOVQ	AX, ""..autotmp_2+72(SP)
	0x006a 00106 (test.go:8)	MOVQ	DX, ""..autotmp_2+80(SP)
	0x006f 00111 (test.go:8)	XORPS	X0, X0
	0x0072 00114 (test.go:8)	MOVUPS	X0, ""..autotmp_1+48(SP)
	0x0077 00119 (test.go:8)	LEAQ	type.[]int(SB), AX
	0x007e 00126 (test.go:8)	MOVQ	AX, (SP)
	0x0082 00130 (test.go:8)	LEAQ	""..autotmp_2+64(SP), AX
	0x0087 00135 (test.go:8)	MOVQ	AX, 8(SP)
	0x008c 00140 (test.go:8)	PCDATA	$0, $1
	0x008c 00140 (test.go:8)	CALL	runtime.convT2Eslice(SB)
	0x0091 00145 (test.go:8)	MOVQ	16(SP), AX
	0x0096 00150 (test.go:8)	MOVQ	24(SP), CX
	0x009b 00155 (test.go:8)	MOVQ	AX, ""..autotmp_1+48(SP)
	0x00a0 00160 (test.go:8)	MOVQ	CX, ""..autotmp_1+56(SP)
	0x00a5 00165 (test.go:8)	LEAQ	""..autotmp_1+48(SP), AX
	0x00aa 00170 (test.go:8)	MOVQ	AX, (SP)
	0x00ae 00174 (test.go:8)	MOVQ	$1, 8(SP)
	0x00b7 00183 (test.go:8)	MOVQ	$1, 16(SP)
	0x00c0 00192 (test.go:8)	PCDATA	$0, $1
	0x00c0 00192 (test.go:8)	CALL	fmt.Println(SB)
	0x00c5 00197 (test.go:9)	MOVQ	88(SP), BP
	0x00ca 00202 (test.go:9)	ADDQ	$96, SP
	0x00ce 00206 (test.go:9)	RET
	0x00cf 00207 (test.go:7)	PCDATA	$0, $0
	0x00cf 00207 (test.go:7)	CALL	runtime.panicindex(SB)
	0x00d4 00212 (test.go:7)	UNDEF
	0x00d6 00214 (test.go:7)	NOP
	0x00d6 00214 (test.go:5)	PCDATA	$0, $-1
	0x00d6 00214 (test.go:5)	CALL	runtime.morestack_noctxt(SB)
	0x00db 00219 (test.go:5)	JMP	0
```

其中的关键行有：

```bash
0x003e 00062 (test.go:6)	CALL	runtime.makeslice(SB)    # 创建slice
0x008c 00140 (test.go:8)	CALL	runtime.convT2Eslice(SB)  # 类型转换
0x00c0 00192 (test.go:8)	CALL	fmt.Println(SB)  # 打印函数
0x00d6 00214 (test.go:5)	CALL	runtime.morestack_noctxt(SB)  # 栈空间扩容
```



然后看`runtime.makeslice`源码，其实非常简单，就是调用了`mallocgc`来分配了空间。

```go
func makeslice(et *_type, len, cap int) unsafe.Pointer {
	mem, overflow := math.MulUintptr(et.size, uintptr(cap))
	if overflow || mem > maxAlloc || len < 0 || len > cap {
		// NOTE: Produce a 'len out of range' error instead of a
		// 'cap out of range' error when someone does make([]T, bignumber).
		// 'cap out of range' is true too, but since the cap is only being
		// supplied implicitly, saying len is clearer.
		// See golang.org/issue/4085.
		mem, overflow := math.MulUintptr(et.size, uintptr(len))
		if overflow || mem > maxAlloc || len < 0 {
			panicmakeslicelen()
		}
		panicmakeslicecap()
	}

	return mallocgc(mem, et, true)
}
```



## 13.4.扩张

当比较短的切片扩容时，系统会多分配 100% 的空间，也就是说分配的数组容量是切片长度的2倍。但切片长度超过1024时，扩容策略调整为多分配 25% 的空间，这是为了避免空间的过多浪费。直至最小于`cap`的最大值。

slice的扩张通过`growslice`来实现，如下：

```go
func growslice(et *_type, old slice, cap int) slice {
	if raceenabled {
		callerpc := getcallerpc()
		racereadrangepc(old.array, uintptr(old.len*int(et.size)), callerpc, funcPC(growslice))
	}
	if msanenabled {
		msanread(old.array, uintptr(old.len*int(et.size)))
	}

	if cap < old.cap {
		panic(errorString("growslice: cap out of range"))
	}

	if et.size == 0 {
		// append should not create a slice with nil pointer but non-zero len.
		// We assume that append doesn't need to preserve old.array in this case.
		return slice{unsafe.Pointer(&zerobase), old.len, cap}
	}

	newcap := old.cap
	doublecap := newcap + newcap
	if cap > doublecap {
		newcap = cap
	} else {
		if old.len < 1024 {
			newcap = doublecap  // 少于1024，直接double扩张
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4  // 当newcap < cap时，newcap = 1.25 * newcap
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}

	var overflow bool
	var lenmem, newlenmem, capmem uintptr
	// Specialize for common values of et.size.
	// For 1 we don't need any division/multiplication.
	// For sys.PtrSize, compiler will optimize division/multiplication into a shift by a constant.
	// For powers of 2, use a variable shift.
	switch {
	case et.size == 1:
		lenmem = uintptr(old.len)
		newlenmem = uintptr(cap)
		capmem = roundupsize(uintptr(newcap))
		overflow = uintptr(newcap) > maxAlloc
		newcap = int(capmem)
	case et.size == sys.PtrSize:
		lenmem = uintptr(old.len) * sys.PtrSize
		newlenmem = uintptr(cap) * sys.PtrSize
		capmem = roundupsize(uintptr(newcap) * sys.PtrSize)
		overflow = uintptr(newcap) > maxAlloc/sys.PtrSize
		newcap = int(capmem / sys.PtrSize)
	case isPowerOfTwo(et.size):
		var shift uintptr
		if sys.PtrSize == 8 {
			// Mask shift for better code generation.
			shift = uintptr(sys.Ctz64(uint64(et.size))) & 63
		} else {
			shift = uintptr(sys.Ctz32(uint32(et.size))) & 31
		}
		lenmem = uintptr(old.len) << shift
		newlenmem = uintptr(cap) << shift
		capmem = roundupsize(uintptr(newcap) << shift)
		overflow = uintptr(newcap) > (maxAlloc >> shift)
		newcap = int(capmem >> shift)
	default:
		lenmem = uintptr(old.len) * et.size
		newlenmem = uintptr(cap) * et.size
		capmem, overflow = math.MulUintptr(et.size, uintptr(newcap))
		capmem = roundupsize(capmem)
		newcap = int(capmem / et.size)
	}

	// The check of overflow in addition to capmem > maxAlloc is needed
	// to prevent an overflow which can be used to trigger a segfault
	// on 32bit architectures with this example program:
	//
	// type T [1<<27 + 1]int64
	//
	// var d T
	// var s []T
	//
	// func main() {
	//   s = append(s, d, d, d, d)
	//   print(len(s), "\n")
	// }
	if overflow || capmem > maxAlloc {
		panic(errorString("growslice: cap out of range"))
	}

	var p unsafe.Pointer
	if et.ptrdata == 0 {
		p = mallocgc(capmem, nil, false)
		// The append() that calls growslice is going to overwrite from old.len to cap (which will be the new length).
		// Only clear the part that will not be overwritten.
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	} else {
		// Note: can't use rawmem (which avoids zeroing of memory), because then GC can scan uninitialized memory.
		p = mallocgc(capmem, et, true)
		if lenmem > 0 && writeBarrier.enabled {
			// Only shade the pointers in old.array since we know the destination slice p
			// only contains nil pointers because it has been cleared during alloc.
			bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(old.array), lenmem)
		}
	}
	memmove(p, old.array, lenmem)

	return slice{p, old.len, newcap}
}
```



## 13.5.切片切割

![WechatIMG341](/Users/dengzhuowen/Desktop/golang源码剖析/img/WechatIMG341.png)

`s1[:]`：同样的共享底层数组，同样是浅拷贝。

如果要使用深拷贝，请使用`copy`函数。

**值得注意的是，**新老 slice 或者新 slice 老数组互相影响的前提是两者共用底层数组，如果因为执行 `append` 操作使得新 slice 底层数组扩容，移动到了新的位置，两者就不会相互影响了。

**所以**，要看新老slice是否相互影响，要看**问题的关键在于两者是否会共用底层数组**。



# 14.interface

## 14.1.值接收者和指针接收者

实现了接收者是值类型的方法，相当于自动实现了接收者是指针类型的方法；

而实现了接收者是指针类型的方法，不会自动生成对应接收者是值类型的方法。



## 14.2.iface和eface

`iface` 和 `eface` 都是 Go 中描述接口的底层结构体，区别在于 `iface` 描述的接口包含方法，而 `eface` 则是不包含任何方法的空接口：`interface{}`。

```go
type iface struct {
    tab  *itab
    data unsafe.Pointer
}

type itab struct {
    inter  *interfacetype  // 接口的类型
    _type  *_type          // 实体的类型，包括内存对齐方式，大小等
    link   *itab
    hash   uint32 // copy of _type.hash. Used for type switches.
    bad    bool   // type does not implement interface
    inhash bool   // has this itab been added to hash?
    unused [2]byte
    fun    [1]uintptr // 放置和接口方法对应的具体数据类型的方法地址，这里存储的是第一个方法的函数指针，如果有更多的方法，在它之后的内存空间里继续存储。
}

type interfacetype struct {
    typ     _type       // 描述 Go 语言中各种数据类型的结构体
    pkgpath name        // 记录定义了接口的包名
    mhdr    []imethod   // 接口所定义的函数列表 
}

type _type struct {
	size       uintptr // 类型大小
	ptrdata    uintptr // size of memory prefix holding all pointers
	hash       uint32  // 类型的hash值
	tflag      tflag   // 类型的flag，和反射相关
	align      uint8   // 内存对齐相关
	fieldalign uint8
	kind       uint8   // 类型的编号，有bool,slice,struct等等
	alg        *typeAlg
	// gcdata stores the GC type data for the garbage collector.
	// If the KindGCProg bit is set in kind, gcdata is a GC program.
	// Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
	gcdata    *byte    // gc相关
	str       nameOff
	ptrToThis typeOff
}
```

`iface` 内部维护两个指针，`tab` 指向一个 `itab` 实体， 它表示接口的类型以及赋给这个接口的实体类型。`data` 则指向接口具体的值，一般而言是一个指向堆内存的指针。

当仅且当这两部分的值都为 `nil` 的情况下，这个接口值就才会被认为 `接口值 == nil`。

![iface 结构体全景](https://user-images.githubusercontent.com/7698088/56564826-82527600-65e1-11e9-956d-d98a212bc863.png)



再看`eface`的源码：

```go
type eface struct {
    _type *_type
    data  unsafe.Pointer
}

type _type struct {
	size       uintptr // 类型大小
	ptrdata    uintptr // size of memory prefix holding all pointers
	hash       uint32  // 类型的hash值
	tflag      tflag   // 类型的flag，和反射相关
	align      uint8   // 内存对齐相关
	fieldalign uint8
	kind       uint8   // 类型的编号，有bool,slice,struct等等
	alg        *typeAlg
	// gcdata stores the GC type data for the garbage collector.
	// If the KindGCProg bit is set in kind, gcdata is a GC program.
	// Otherwise it is a ptrmask bitmap. See mbitmap.go for details.
	gcdata    *byte    // gc相关
	str       nameOff
	ptrToThis typeOff
}
```

相比 `iface`，`eface` 就比较简单了。只维护了一个 `_type` 字段，表示空接口所承载的具体的实体类型。`data` 描述了具体的值。

![eface 结构体全景](https://user-images.githubusercontent.com/7698088/56565105-318f4d00-65e2-11e9-96bd-4b2e192791dc.png)

Go 语言各种数据类型都是在 `_type` 字段的基础上，增加一些额外的字段来进行管理的：

```go
type arraytype struct {
    typ   _type
    elem  *_type
    slice *_type
    len   uintptr
}

type chantype struct {
    typ  _type
    elem *_type
    dir  uintptr
}

type slicetype struct {
    typ  _type
    elem *_type
}

type structtype struct {
    typ     _type
    pkgPath name
    fields  []structfield
}
```



**如何判断某个对象是否实现了某个接口？**先组装出对应的`interface`类型，然后通过检查`_type`结构体中的`hash`来进行校验。

当判定一种类型是否满足某个接口时，Go 使用类型的方法集和接口所需要的方法集进行匹配，如果类型的方法集完全包含接口的方法集，则可认为该类型实现了该接口。



## 14.3.接口转换的原理

例子如下：

```go
package main

import "fmt"

type coder interface {
    code()
    run()
}

type runner interface {
    run()
}

type Gopher struct {
    language string
}

func (g Gopher) code() {
    return
}

func (g Gopher) run() {
    return
}

func main() {
    var c coder = Gopher{}

    var r runner
    r = c
    fmt.Println(c, r)
}
```

执行命令：

```bash
go tool compile -S ./src/main.go
```

可以看到： `r = c` 这一行语句实际上是调用了 `runtime.convI2I(SB)`，也就是 `convI2I` 函数，从函数名来看，就是将一个 `interface` 转换成另外一个 `interface`，看下它的源代码：

```go
func convI2I(inter *interfacetype, i iface) (r iface) {
    tab := i.tab
    if tab == nil {
        return
    }
    if tab.inter == inter {
        r.tab = tab
        r.data = i.data
        return
    }
    r.tab = getitab(inter, tab._type, false)
    r.data = i.data
    return
}
```

实际上 `convI2I` 函数真正要做的事，找到新 `interface` 的 `tab` 和 `data`，就大功告成了。

然后，我们再看下`getitab`的源码：

```go
func getitab(inter *interfacetype, typ *_type, canfail bool) *itab {
	if len(inter.mhdr) == 0 {
		throw("internal error - misuse of itab")
	}

	// easy case
	if typ.tflag&tflagUncommon == 0 {
		if canfail {
			return nil
		}
		name := inter.typ.nameOff(inter.mhdr[0].name)
		panic(&TypeAssertionError{nil, typ, &inter.typ, name.name()})
	}

	var m *itab

	// First, look in the existing table to see if we can find the itab we need.
	// This is by far the most common case, so do it without locks.
	// Use atomic to ensure we see any previous writes done by the thread
	// that updates the itabTable field (with atomic.Storep in itabAdd).
	t := (*itabTableType)(atomic.Loadp(unsafe.Pointer(&itabTable)))
	if m = t.find(inter, typ); m != nil {
		goto finish
	}

	// Not found.  Grab the lock and try again.
	lock(&itabLock)
  // 遍历哈希表的一个slot
	if m = itabTable.find(inter, typ); m != nil {
    // 如果在 hash 表中已经找到了 itab（inter 和 typ 指针都相同）
		unlock(&itabLock)
		goto finish
	}

	// Entry doesn't exist yet. Make a new entry & add it.
  // 在 hash 表中没有找到 itab，那么新生成一个 itab
	m = (*itab)(persistentalloc(unsafe.Sizeof(itab{})+uintptr(len(inter.mhdr)-1)*sys.PtrSize, 0, &memstats.other_sys))
	m.inter = inter
	m._type = typ
	m.init()
  // 添加到全局的 hash 表中
	itabAdd(m)
	unlock(&itabLock)
finish:
	if m.fun[0] != 0 {
		return m
	}
	if canfail {
		return nil
	}
	// this can only happen if the conversion
	// was already done once using the , ok form
	// and we have a cached negative result.
	// The cached result doesn't record which
	// interface function was missing, so initialize
	// the itab again to get the missing function name.
	panic(&TypeAssertionError{concrete: typ, asserted: &inter.typ, missingMethod: m.init()})
}

// itabAdd adds the given itab to the itab hash table.
// itabLock must be held.
func itabAdd(m *itab) {
	// Bugs can lead to calling this while mallocing is set,
	// typically because this is called while panicing.
	// Crash reliably, rather than only when we need to grow
	// the hash table.
	if getg().m.mallocing != 0 {
		throw("malloc deadlock")
	}

	t := itabTable
	if t.count >= 3*(t.size/4) { // 75% load factor
		// Grow hash table.
		// t2 = new(itabTableType) + some additional entries
		// We lie and tell malloc we want pointer-free memory because
		// all the pointed-to values are not in the heap.
		t2 := (*itabTableType)(mallocgc((2+2*t.size)*sys.PtrSize, nil, true))
		t2.size = t.size * 2

		// Copy over entries.
		// Note: while copying, other threads may look for an itab and
		// fail to find it. That's ok, they will then try to get the itab lock
		// and as a consequence wait until this copying is complete.
		iterate_itabs(t2.add)
		if t2.count != t.count {
			throw("mismatched count during itab table copy")
		}
		// Publish new hash table. Use an atomic write: see comment in getitab.
		atomicstorep(unsafe.Pointer(&itabTable), unsafe.Pointer(t2))
		// Adopt the new table as our own.
		t = itabTable
		// Note: the old table can be GC'ed here.
	}
	t.add(m)
}
```

`getitab` 函数会根据 `interfacetype` 和 `_type` 去全局的 itab 哈希表中查找，如果能找到，则直接返回；否则，会根据给定的 `interfacetype` 和 `_type` 新生成一个 `itab`，并插入到 itab 哈希表，这样下一次就可以直接拿到 `itab`。



## 14.4.Go的接口与C++接口的差别

C++ 的接口是使用抽象类来实现的，如果类中至少有一个函数被声明为纯虚函数，则这个类就是抽象类。纯虚函数是通过在声明中使用 "= 0" 来指定的。例如：

```c++
class Shape
{
   public:
      // 纯虚函数
      virtual double getArea() = 0;
   private:
      string name;      // 名称
};
```

差异如下：

+ C++ 定义接口的方式称为“侵入式”，而 Go 采用的是 “非侵入式”，不需要显式声明，只需要实现接口定义的函数，编译器自动会识别。
+ C++ 和 Go 在定义接口方式上的不同，也导致了底层实现上的不同。C++ 通过虚函数表来实现基类调用派生类的函数；而 Go 通过 `itab` 中的 `fun` 字段来实现接口变量调用实体类型的函数。
+ C++ 中的虚函数表是在编译期生成的；而 Go 的 `itab` 中的 `fun` 字段是在运行期间动态生成的（运行时才进行绑定，视不同接口对象而定，跟C++里面的虚表指针很类似）。原因在于，Go 中实体类型可能会无意中实现 N 多接口，很多接口并不是本来需要的，所以不能为类型实现的所有接口都生成一个 `itab`， 这也是“非侵入式”带来的影响；这在 C++ 中是不存在的，因为派生需要显示声明它继承自哪个基类。(**这一点比较重要，值得注意**。)





