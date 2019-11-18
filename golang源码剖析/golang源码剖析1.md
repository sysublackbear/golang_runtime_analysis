# 1.程序引导

我们先写一个最简单的程序，来看下代码是怎么引导到程序的入口的。代码如下：

```go
package main

import (
	"fmt"
)

func main() {
  fmt.Println("Hello world!")
}
```

然后进行编译，建议使用 `-gcflags "-N -l"`参数来关闭编译器代码优化和函数内联，避免断点和单步执行无法准确对应源码行，避免小函数和局部变量被优化掉。

```bash
$ go build -gcflags "-N -l" -o test test.go
```

然后用GDB查看。

```bash
$ gdb test

(gdb) info files
Symbols from "/home/sysublackbear/go/src/test".
Local exec file:
	`/home/sysublackbear/go/src/test', file type elf64-x86-64.
	Entry point: 0x44f4d0
	0x0000000000401000 - 0x0000000000482178 is .text
	0x0000000000483000 - 0x00000000004c4a5a is .rodata
	0x00000000004c4b80 - 0x00000000004c56c8 is .typelink
	0x00000000004c56c8 - 0x00000000004c5708 is .itablink
	0x00000000004c5708 - 0x00000000004c5708 is .gosymtab
	0x00000000004c5720 - 0x0000000000513119 is .gopclntab
	0x0000000000514000 - 0x0000000000520bdc is .noptrdata
	0x0000000000520be0 - 0x00000000005276f0 is .data
	0x0000000000527700 - 0x0000000000543d88 is .bss
	0x0000000000543da0 - 0x0000000000546438 is .noptrbss
	0x0000000000400f9c - 0x0000000000401000 is .note.go.buildid
(gdb) b *0x44f4d0
Breakpoint 1 at 0x44f4d0: file /opt/go/src/runtime/rt0_linux_amd64.s, line 8.
```

我们找到程序的入口：`Entry point: 0x44f4d0`。然后我们通过断点找出目标源文件的信息。

```bash
(gdb) b *0x44f4d0
Breakpoint 1 at 0x44f4d0: file /opt/go/src/runtime/rt0_linux_amd64.s, line 8.
```

然后再看下`/opt/go/src/runtime/rt0_linux_amd64.s`的代码：

```bash
// Copyright 2009 The Go Authors. All rights reserved.
// Use of this source code is governed by a BSD-style
// license that can be found in the LICENSE file.

#include "textflag.h"

TEXT _rt0_amd64_linux(SB),NOSPLIT,$-8
	JMP	_rt0_amd64(SB)

TEXT _rt0_amd64_linux_lib(SB),NOSPLIT,$0
	JMP	_rt0_amd64_lib(SB)
```

我们再看下`_rt0_amd64`在哪里。

```bash
(gdb) b _rt0_amd64
Breakpoint 3 at 0x44be00: file /opt/go/src/runtime/asm_amd64.s, line 15.
```

然后看下对应的汇编代码：

```bash
TEXT _rt0_amd64(SB),NOSPLIT,$-8
	MOVQ 0(SP), DI  // argc
	LEAQ 8(SP), SI  // argv
	JMP runtime.rt0_go(SB)
```

再看下`runtime.rt0_go`在哪里。

```bash
(gdb) b runtime.rt0_go
Breakpoint 2 at 0x44be10: file /opt/go/src/runtime/asm_amd64.s, line 89.
```

打开asm_amd64.s的89行。

```bash
TEXT runtime·rt0_go(SB),NOSPLIT,$0
     // copy arguments forward on an even stack
     MOVQ    DI, AX      // argc
     MOVQ    SI, BX      // argv
     SUBQ    $(4*8+7), SP        // 2args 2auto
     ANDQ    $~15, SP
     MOVQ    AX, 16(SP)
     MOVQ    BX, 24(SP)

     // create istack out of the given (operating system) stack.
     // _cgo_init may update stackguard.
     MOVQ    $runtime·g0(SB), DI
     LEAQ    (-64*1024+104)(SP), BX
     MOVQ    BX, g_stackguard0(DI)
     MOVQ    BX, g_stackguard1(DI)
     MOVQ    BX, (g_stack+stack_lo)(DI)
     MOVQ    SP, (g_stack+stack_hi)(DI)

     // find out information about the processor we're on
     MOVL    $0, AX
     CPUID
     MOVL    AX, SI
     CMPL    AX, $0
     JE  nocpuinfo
```

再看`nocpuinfo`：

```bash
 nocpuinfo:
     // if there is an _cgo_init, call it.
     MOVQ    _cgo_init(SB), AX
     TESTQ   AX, AX
     JZ  needtls
     // g0 already in DI
     MOVQ    DI, CX  // Win64 uses CX for first parameter
     MOVQ    $setg_gcc<>(SB), SI
     CALL    AX

     // update stackguard after _cgo_init
     MOVQ    $runtime·g0(SB), CX
     MOVQ    (g_stack+stack_lo)(CX), AX
     ADDQ    $const__StackGuard, AX
     MOVQ    AX, g_stackguard0(CX)
     MOVQ    AX, g_stackguard1(CX)

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
```

再看`ok`：

```bash
ok:
     // set the per-goroutine and per-mach "registers"
     get_tls(BX)
     LEAQ    runtime·g0(SB), CX
     MOVQ    CX, g(BX)
     LEAQ    runtime·m0(SB), AX

     // save m->g0 = g0
     MOVQ    CX, m_g0(AX)
     // save m0 to g0->m
     MOVQ    AX, g_m(CX)

     CLD             // convention is D is always left cleared
     CALL    runtime·check(SB)

     MOVL    16(SP), AX      // copy argc
     MOVL    AX, 0(SP)
     MOVQ    24(SP), AX      // copy argv
     MOVQ    AX, 8(SP)
     // 调用初始化函数
     CALL    runtime·args(SB)
     CALL    runtime·osinit(SB)
     CALL    runtime·schedinit(SB)

     // create a new goroutine to start program
     // 创建 main goroutine 用于执行 runtime.main
     MOVQ    $runtime·mainPC(SB), AX     // entry，执行runtime.main
     PUSHQ   AX
     PUSHQ   $0          // arg size
     CALL    runtime·newproc(SB)
     POPQ    AX
     POPQ    AX

     // start this M
     CALL    runtime·mstart(SB)

     MOVL    $0xf1, 0xf1  // crash
     RET

DATA    runtime·mainPC+0(SB)/8,$runtime·main(SB)
GLOBL   runtime·mainPC(SB),RODATA,$8
```

至此，由汇编语言针对特定平台实现的引导过程就全部完成。后续内容基本上都是有golang代码实现的。

```bash
(gdb) b runtime.main
Breakpoint 1 at 0x427700: file /opt/go/src/runtime/proc.go, line 109.
```

由上面可看，初始化过程分为下面几个步骤：

+ runtime.args
+ runtime.osinit
+ runtime.schedinit



## 1.1.golang编译过程

### 1.1.1.常用的go build选项

+ `-a`：将命令源码文件与库源码文件全部重新构建，即使是最新的；
+ `-n`：把编译期间涉及的命令全部打印出来，但不会真的执行，非常方便我们学习；
+ `-race`：开启竞态条件的检测，支持的平台有限制；
+ `-x`：打印编译期间用到的命名，它与`-n`的区别是，它不仅打印还会执行。



### 1.1.2.编译器原理

![img](https://mmbiz.qpic.cn/mmbiz_png/5WXEuGYZIibBfKWA8wUZoYugGibCfLB58rBLRHPxxA2YA06ObwoFltslzGoo0koXCZvfCUYRIQdY6ibIjt3KjuRXg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### 1.1.3.编译全过程

1. **词法分析**：从源代码翻译成`token`的过程；
2. **语法分析**：把生成的 `Token` 序列，作为语法分析器的输入。然后经过处理后生成 `AST` 结构作为输出。（构造树有自上而下和自下而上两种方法，golang采用自下而上的方式构造`AST`；
3. **语义分析**：主要要做类型检查，还有类型推断，查看类型是否匹配，是否进行隐式转化（go 没有隐式转化）。以及函数调用内联，逃逸分析等等；
4. **中间码生成**：在进行各个平台的兼容之前，进行一些底层函数的替换；
5. **代码优化**：比如进行以下步骤：
   1. 并行性，充分利用现在多核计算机的特性；
   2. 流水线，cpu 有时候在处理 a 指令的时候，还能同时处理 b 指令；
   3. 指令的选择，为了让 cpu 完成某些操作，需要使用指令，但是不同的指令效率有非常大的差别，这里会进行指令优化；
   4. 利用寄存器与高速缓存，我们都知道 cpu 从寄存器取是最快的，从高速缓存取次之。这里会进行充分的利用。
6. **机器码生成**：经过优化后的中间代码，首先会在这个阶段被转化为汇编代码（Plan9），而汇编语言仅仅是机器码的文本表示，机器还不能真的去执行它。所以这个阶段会调用汇编器，汇编器会根据我们在执行编译时设置的架构，调用对应代码来生成目标机器码。



# 2.初始化

根据上面的`ok`的汇编代码，可以分别看下初始化过程涉及哪些方面：

```bash
(gdb) b runtime.args
Breakpoint 7 at 0x42ebf0: file /opt/go/src/runtime/runtime1.go, line 48.

(gdb) b runtime.osinit
Breakpoint 8 at 0x41e9d0: file /opt/go/src/runtime/os_linux.go, line 172.

(gdb) b runtime.schedinit
Breakpoint 9 at 0x424590: file /opt/go/src/runtime/proc.go, line 40.
```

逐个文件的源码进行分析：

`runtime1.go`：

```go
func args(c int32, v **byte) {
     argc = c
     argv = v
     sysargs(c, v)
}
```

`os_linux.go`，函数`osinit`确定CPU Core数量。

```go
func osinit() {
    ncpu = getproccount()   // 获取cpu核数量
}

func getproccount() int32 {
    // This buffer is huge (8 kB) but we are on the system stack
    // and there should be plenty of space (64 kB).
    // Also this is a leaf, so we're not holding up the memory for long.
    // See golang.org/issue/11823.
    // The suggested behavior here is to keep trying with ever-larger
    // buffers, but we don't have a dynamic memory allocator at the
    // moment, so that's a bit tricky and seems like overkill.
    const maxCPUs = 64 * 1024
    var buf [maxCPUs / 8]byte
  	// 获取CPU亲和性掩码数组
  	/*
  	CPU的亲和性， 就是进程要在指定的 CPU 上尽量长时间地运行而不被迁移到其他处理器，亲和性是从affinity翻译过来的，应该有点不准确，给人的感觉是亲和性就是有倾向的意思，而实际上是倒向的意思，称为CPU关联性更好，程序员的土话就是绑定CPU，绑核。
  	在多核运行的机器上，每个CPU本身自己会有缓存，缓存着进程使用的信息，而进程可能会被OS调度到其他CPU上，如此，CPU cache命中率就低了，当绑定CPU后，程序就会一直在指定的cpu跑，不会由操作系统调度到其他CPU上，性能有一定的提高。
  	另外一种使用绑核考虑就是将重要的业务进程隔离开，对于部分实时进程调度优先级高，可以将其绑定到一个指定核上，既可以保证实时进程的调度，也可以避免其他CPU上进程被该实时进程干扰。
  	*/
    r := sched_getaffinity(0, unsafe.Sizeof(buf), &buf[0])
    if r < 0 {
        return 1
    }
    n := int32(0)
  	// 查看进程绑定在哪些核上面
    for _, v := range buf[:r] {
        for v != 0 {
            n += int32(v & 1)
            v >>= 1
        }
    }
    if n == 0 {
        n = 1
    }
    return n
}
```

`proc.go`，

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

  // 最大系统线程数量限制，参考标准库 runtime/debug.SetMaxThreads
	sched.maxmcount = 10000

	tracebackinit()
	moduledataverify()
  // 栈，内存分配器，调度器相关初始化。
	stackinit()
	mallocinit()
	mcommoninit(_g_.m)
	alginit()       // maps must not be used before this call
	modulesinit()   // provides activeModules
	typelinksinit() // uses maps, activeModules
	itabsinit()     // uses activeModules

	msigsave(_g_.m)
	initSigmask = _g_.m.sigmask

  // 处理命令行参数和环境变量
	goargs()
	goenvs()
  
  // 处理 GODEBUG，GOTRACEBACK 调试相关的环境变量设置。
	parsedebugvars()
  
  // gc(垃圾回收器)初始化
	gcinit()

	sched.lastpoll = uint64(nanotime())
	procs := ncpu
	if n, ok := atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
		procs = n
	}
  
  // 调整进程数量
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
}
```

**事实上**，初始化操作到此并未结束，因为接下来要执行的是`runtime.main`，而不是用户逻辑入口函数`main.main`，详见上面汇编代码。



`runrime.main`位于`proc.go`中的`main`函数，逻辑如下：

proc.go

```go
// The main goroutine.
func main() {
	g := getg()

	// Racectx of m0->g0 is used only as the parent of the main goroutine.
	// It must not be used for anything else.
	g.m.g0.racectx = 0

	// Max stack size is 1 GB on 64-bit, 250 MB on 32-bit.
	// Using decimal instead of binary GB and MB because
	// they look nicer in the stack overflow failure message.
  // 执行栈的最大限制: 1GB on 64-bit， 250MB on 32-bit.
	if sys.PtrSize == 8 {
		maxstacksize = 1000000000
	} else {
		maxstacksize = 250000000
	}

	// Allow newproc to start new Ms.
	mainStarted = true

  // 启动系统后台监控（定期垃圾回收，以及并发任务调度相关的信息）
	systemstack(func() {
		newm(sysmon, nil)
	})

	// Lock the main goroutine onto this, the main OS thread,
	// during initialization. Most programs won't care, but a few
	// do require certain calls to be made by the main thread.
	// Those can arrange for main.main to run in the main thread
	// by calling runtime.LockOSThread during initialization
	// to preserve the lock.
	lockOSThread()

	if g.m != &m0 {
		throw("runtime.main not on m0")
	}

  // 执行runtime包内所有初始化函数 init
	runtime_init() // must be before defer
	if nanotime() == 0 {
		throw("nanotime returning zero")
	}

	// Defer unlock so that runtime.Goexit during init does the unlock too.
	needUnlock := true
	defer func() {
		if needUnlock {
			unlockOSThread()
		}
	}()

	// Record when the world started. Must be after runtime_init
	// because nanotime on some platforms depends on startNano.
	runtimeInitTime = nanotime()

  // 启动垃圾回收器后台操作
	gcenable()

	main_init_done = make(chan bool)
	if iscgo {
		if _cgo_thread_start == nil {
			throw("_cgo_thread_start missing")
		}
		if GOOS != "windows" {
			if _cgo_setenv == nil {
				throw("_cgo_setenv missing")
			}
			if _cgo_unsetenv == nil {
				throw("_cgo_unsetenv missing")
			}
		}
		if _cgo_notify_runtime_init_done == nil {
			throw("_cgo_notify_runtime_init_done missing")
		}
		// Start the template thread in case we enter Go from
		// a C-created thread and need to create a new thread.
		startTemplateThread()
		cgocall(_cgo_notify_runtime_init_done, nil)
	}

  // 执行所有用户包（包括标准库）初始化函数init
	fn := main_init // make an indirect call, as the linker doesn't know the address of the main package when laying down the runtime
	fn()
	close(main_init_done)

	needUnlock = false
	unlockOSThread()

	if isarchive || islibrary {
		// A program compiled with -buildmode=c-archive or c-shared
		// has a main, but it is not executed.
		return
	}
  
  // 执行用户逻辑入口 main.main 函数
	fn = main_main // make an indirect call, as the linker doesn't know the address of the main package when laying down the runtime
	fn()
	if raceenabled {
		racefini()
	}

	// Make racy client program work: if panicking on
	// another goroutine at the same time as main returns,
	// let the other goroutine finish printing the panic trace.
	// Once it does, it will exit. See issues 3934 and 20018.
	if atomic.Load(&runningPanicDefers) != 0 {
		// Running deferred functions should not take long.
		for c := 0; c < 1000; c++ {
			if atomic.Load(&runningPanicDefers) == 0 {
				break
			}
			Gosched()
		}
	}
	if atomic.Load(&panicking) != 0 {
		gopark(nil, nil, "panicwait", traceEvGoStop, 1)
	}

  // 执行结束，返回退出状态码
	exit(0)
	for {
		var x *int32
		*x = 0
	}
}
```

与之相关的就是`runtime_init`和`main_init`这两个函数，它们都由编译器动态生成。

+ `runtime`内相关的多个`init`函数被赋予唯一符号名，然后再由`runtime.init`进行统一调用。
+ 至于`main.init`，情况基本一致，区别在于它负责调用非`runtime`包的初始化函数。



**runtime内相关的多个init函数被赋予唯一符号名，然后再由runtime.init进行统一调用。**

最后，必须记住：

+ 所有init函数都在同一个goroutine内执行；
+ 所有init函数结束后才会执行main.main函数。

![image-20191111161411307](https://github.com/sysublackbear/golang_runtime_analysis/blob/master/img/image-20191111161411307.png)



# 3.内存分配

内存分配的整个过程：

1. 每次从操作系统申请一大块内存（比如1MB），以减少系统调用；
2. 将申请到的大块内存按照特定大小预先切分成小块，构成链表；
3. 为对象分配内存时，只须从大小合适的链表提取一个小块即可；
4. 回收对象内存时，将该小块内存重新归还到原链表，以便复用；
5. 如闲置内存过多，则尝试归还部分内存给操作系统，降低整体开销。

其中，内存分配器只管理内存块，并不关心对象状态。且它不会主动回收内存，垃圾回收器（GC）在完成清理操作后，触发内存分配器的回收操作。



golang里面，分配器将其管理的内存块分为两种。

+ span：由多个地址连续的页（page）组成的大块内存；
+ object：将span按特定大小切分成多个小块，每个小块可存储一个对象。

分配器按页数来区分不同大小的span。比如，以页数为单位将span存放到管理数组中，需要时就以页数为索引进行查找。当然，span大小并非固定不变。在获取闲置span时，如果没找到大小合适的，那就返回页数更多的，此时会引发裁剪操作，多余部分将构成新的span被放回管理数组。分配器还会尝试将地址相邻的空闲span合并，以构建更大的内存块，减少碎片，提供更灵活的分配策略。



内存分配的管理组件：

优秀的内存分配器必须要在性能和内存利用率之间做到平衡。Golang直接使用了tcmalloc的成熟架构。

分配器由三种组件组成：

+ ThreadCache：每个运行期工作线程都会绑定一个cache，用于无锁object分配。
+ CentralCache：为所有cache提供切分好的后备span资源。
+ PageHeap：管理闲置span，需要时向操作系统申请内存。

![img](https://pic4.zhimg.com/80/v2-05a8740554bedf4dc0a6912c6e8551db_hd.png)

![skiplist](https://cyningsun.github.io/public/blog-img/allocator/tcmalloc.png)

由上图可知，所有的cache都采用了**freeList**的数据结构。

分配过程：

1. 计算待分配对象对应的规格（size class）；
2. 每个线程都有一个线程局部的ThreadCache，按照不同的规格，维护了对象的链表；
3. 如果ThreadCache的对象不够了，就从CentralCache进行批量分配；
4. 如果CentralCache依然没有，就从PageHeap申请Span；
5. 如果PageHeap没有合适的Page，就只能从操作系统申请了。

释放过程：

1. 将标记为可回收的object交还给所属的ThreadCache；
2. ThreadCache依然遵循批量释放的策略，对象积累到一定程度就释放给CentralCache；
3. CentralCache发现一个Span的内存完全释放了，就可以把这个Span归还给PageHeap；
4. PageHeap发现一批连续的Page都释放了，就可以归还给操作系统。

值得一提：

+ page和span之间的关系通过radix tree（前缀树）去记录，保证效率不低的位运算。
+ 操作系统->PageHeap(内含SpanList)->Span（Span之间通过前缀树查找）->ThreadCache（内含FreeList）->object。



## 3.1.ptmalloc, tcmalloc和jemalloc之间的区别

### 3.1.1.ptmalloc：

+ 每个进程只有1个主分区，但可能会有多个动态分区，ptmalloc会根据系统对分区的争用情况动态增加动态分区的数量；
+ 主分配区在二进制启动的时候调用`sbrk`从堆中分配内存，动态分区则是每次使用`mmap`向操作系统批发固定大小的虚拟内存，释放的时候调用`munmap`
+ 主分配去和动态分配去采用环形数组去管理，每个分配区都使用互斥锁使得线程对于该分区的访问保证互斥。
+ 用户向ptmalloc请求分配内存时，内存分配器会将缓存的内存切成小块，减少系统调用的次数。
+ 数据结构有：
  + `malloc_state`：所有分区共享的公共头部；
  + `heap_info`：每个分区独有的头部；
  + `malloc_chunk`：根据用户请求，每个堆被分为若干chunk。每个chunk都有自己的chunk header。内存管理使用malloc_chunk，把heap当做链表从一个内存块游走到下一个块。

![skiplist](https://cyningsun.github.io/public/blog-img/allocator/ptmalloc-logical.png)

**注意：**Main arena 无需维护多个堆，因此也无需 heap_info。当空间耗尽时，与 thread arena 不同，main arena 可以通过 sbrk 拓展堆段，直至堆段「碰」到内存映射段。

+ 用户调用`malloc()`分配内存空间时，该线程先看线程私有变量中是否已经存在一个分配区，如果存在，则尝试对该分配区加锁，如果加锁成功，使用该分配区分配内存，如果失败，该线程搜索循环链表试图获得一个没有加锁的分配区。如果所有的分配区都已经加锁，那么`malloc()`会开辟一个新的分配区，把该分配区加入到全局分配区循环链表并加锁，然后使用该分配区进行分配内存操作。
+ 在释放操作中，线程同样试图获得待释放内存块所在分配区的锁，如果该分配区正在被别的线程使用，则需要等待直到其他线程释放该分配区的互斥锁之后才进行释放操作（自旋操作）；
+ 对于空闲的`chunk`，malloc都维护着[bins]的列表数据结构，根据chunk的大小和状态，放到不同的bin中。
  + Fast bins：是小内存块的高速缓存，当一些大小小于64字节的chunk被回收时，首先会放入fast bins中，在分配小内存时，首先会查看fast bins中是否有合适的内存块，如果存在，则直接返回fast bins中的内存块，以加快分配速度。**Fast bins 可以看着是small bins的一小部分cache，主要是用于提高小内存的分配效率，虽然这可能会加剧内存碎片化，但也大大加速了内存释放的速度！**
  + Usorted bin：只有一个，回收的chunk块必须先放到unsorted bin中，分配内存时会查看unsorted bin中是否有合适的chunk，如果找到满足条件的chunk，则直接返回给用户，否则将unsorted bin的所有chunk放入small bins或是large bins中。**Unsorted bin 可以重新使用最近 free 掉的 chunk，从而消除了寻找合适 bin 的时间开销，会增加很多的内存碎片，进而加速了内存分配及释放的效率。**
  + Small bins：用于存放固定大小的chunk，共64个bin，最小的chunk大小为16字节或32字节，每个bin的大小相差8字节或是16字节，当分配小内存块时，采用精确匹配的方式从small bins中查找合适的chunk。**Small bins 相邻的 free chunk 将被合并，这减缓了内存碎片化，但是减慢了 free 的速度；**
  + Large bins：用于存储大于等于512B或1024B的空闲chunk，这些chunk使用双向链表的形式按大小顺序排序，分配内存时按**最近匹配方式**从large bins中分配chunk。**Large bin 中所有 chunk 大小不一定相同，各 chunk 大小递减保存。最大的 chunk 保存顶端，而最小的 chunk 保存在尾端；查找较慢，且释放时两个相邻的空闲 chunk 会被合并。**



![skiplist](https://cyningsun.github.io/public/blog-img/allocator/ptmalloc-free-bins.png)



+ 一个 arena 中最顶部的 chunk 被称为「top chunk」。它不属于任何 bin 。当所有 bin 中都没有合适空闲内存时，就会使用 top chunk 来响应用户请求。当 top chunk 的大小比用户请求的大小小的时候，top chunk 就通过 sbrk（main arena）或 mmap（ thread arena）系统调用扩容。

+ ptmalloc的弊端：
  + 如果后分配的内存先释放，无法及时归还系统。因为ptmalloc收缩内存是从top chunk开始，如果与top chunk相邻的chunk不能释放，top chunk以下的chunk都无法释放。
  + 内存不能在线程间移动，多线程使用内存不均衡将导致内存浪费；
  + 每个chunk至少占8个字节，开销大；
  + 不定期分配长生命周期的内存容易造成内存碎片，不利于回收；
  + 加锁耗时，无论当前分区有无耗时，在内存分配和释放时，会首先加锁。



### 3.1.2.tcmalloc

+ tcmalloc是专门对多线程并发的内存管理而设计的，tcmalloc在线程级别实现了缓存，使得用户在申请内存时大多情况下是无锁内存分配。
+ 整个tcmalloc实现了三级缓存，分别是：ThreadCache（线程级缓存），CentralCache（中央级缓存，使用FreeList的数据结构），PageHeap（页缓存），其中，CentralCache和PageHeap需要加锁访问。

![skiplist](https://cyningsun.github.io/public/blog-img/allocator/tcmalloc-simple.png)

+ tcmalloc把8kb的连续内存称为一个页，一个或多个连续的页组成一个Span。tcmalloc中所有页级别的操作，都是对Span的操作。PageHeap是一个全局的管理Span的类，PageHeap把小的Span保存到双向循环链表中，而大的Span则保存在了Set中，保证内存分配的速度，减少内存查找。
+ **分配过程（大致看看即可）：**每个线程都一个线程局部的 ThreadCache，ThreadCache中包含一个链表数组FreeList list_[kNumClasses]，维护了不同规格的空闲内存的链表；当申请内存的时候可以直接根据大小寻找恰当的规则的内存。如果ThreadCache的对象不够了，就从 CentralCache 进行批量分配；如果 CentralCache 依然没有，就从PageHeap申请Span；PageHeap首先在free[n,128]中查找、然后到large set中查找，目标就是找到一个最小的满足要求的空闲Span，优先使用normal类链表中的Span。如果找到了一个Span，则尝试分裂(Carve)这个Span并分配出去；如果所有的链表中都没找到length>=n的Span，则只能从操作系统申请了。Tcmalloc一次最少向系统申请1MB的内存，默认情况下，使用sbrk申请，在sbrk失败的时候，使用mmap申请。当我们申请的内存大于kMaxSize(256k)的时候，内存大小超过了ThreadCache和CenterCache的最大规格，所以会直接从全局的PageHeap中申请最小的Span分配出去(return span->start « kPageShift))；
+ tcmalloc的优势：
  + 小内存可以在ThreadCache中不加锁分配（加锁的代价大约100ms）；
  + 大内存可以直接按照大小分配而不需要再像ptmalloc一样进行查找；
  + 大内存加锁使用更高效的自旋锁；
  + 减少了内存碎片，原因详见：https://zhuanlan.zhihu.com/p/29216091 （概括就是使用合理的chunk size，来减少内存碎片的占有率）；
+ tcmalloc的弊端：使用自旋锁虽然减少了加锁效率，但是如果使用大内存较多的情况下，内存在Central Cache或者Page Heap加锁分配。而tcmalloc对大小内存的分配过于保守，在一些内存需求较大的服务（如推荐系统），小内存上限过低，当请求量上来，锁冲突严重，CPU使用率将指数暴增。



### 3.1.3.jemalloc

+ 为了解决多线程在申请分配内存块时候的加锁问题，jemalloc将一把global lock分散成很多与线程相关的lock。而针对多核心, 则要尽量把不同线程下分配的内存隔离开, 避免不同线程使用同一个cache-line的情况.按照上面的思路, 一个较好的实现方式就是引入arena.将内存划分成若干数量的arenas, 线程最终会与某一个arena绑定.由于两个arena在地址空间上几乎不存在任何联系, 就可以在无锁的状态下完成分配. 同样由于空间不连续, 落到同一个cache-line中的几率也很小, 保证了各自独立。由于arena的数量有限, 因此不能保证所有线程都能独占arena, 分享同一个arena的所有线程, 由该arena内部的lock保持同步。（其实有点儿类似tcmalloc的CentralCache）；
+ chunk是仅次于arena的次级内存结构，arena都有专属的chunks, 每个chunk的头部都记录了chunk的分配信息。chunk是具体进行内存分配的区域，目前的默认大小是4M。chunk以page（默认为4K)为单位进行管理，每个chunk的前几个page（默认是6个）用于存储chunk的元数据，后面跟着一个或多个page的runs。后面的runs可以是未分配区域， 多个小对象组合在一起组成run, 其元数据放在run的头部。 大对象构成的run, 其元数据放在chunk的头部。在使用某一个chunk的时候，会把它分割成很多个run，并记录到bin中。不同size的class对应着不同的bin，在bin里，都会有一个红黑树来维护空闲的run，并且在run里，使用了bitmap来记录了分配状态。此外，每个arena里面维护一组按地址排列的可获得的run的红黑树。（利用红黑树来管理各个run，层级结构：arena->chunk->run）

+ 用户态向jemalloc申请内存：jemalloc 按照内存分配请求的尺寸，分了 small object (例如 1 – 57344B)、 large object (例如 57345 – 4MB )、 huge object (例如 4MB以上)。jemalloc同样有一层线程缓存的内存名字叫tcache，当分配的内存大小小于tcache_maxclass时，jemalloc会首先在tcache的small object以及large object中查找分配，tcache不中则从arena中申请run，并将剩余的区域缓存到tcache。若arena找不到合适大小的内存块， 则向系统申请内存。当申请大小大于tcache_maxclass且大小小于huge大小的内存块时，则直接从arena开始分配。而huge object的内存不归arena管理， 直接采用mmap从system memory中申请，并由一棵与arena独立的红黑树进行管理。

![skiplist](https://cyningsun.github.io/public/blog-img/allocator/jemalloc-simple.png)



# 4.对象分配

我们看一个例子：

```go
package main

import "fmt"

func test() *int {
    x := new(int)  // 注意：new说明在堆上分配了一个空间
    *x = 0xAABB  
    return x
}

func main(){
    fmt.Println(*test())
}
```



当我们使用编译器禁用内联优化时，所生成代码和我们的源码表面上预期一致。

```bash
[sysublackbear@centos-linux zhuowen.deng]$ go build -gcflags "-l" -o test test.go  // 关闭内联优化
[sysublackbear@centos-linux zhuowen.deng]$ go tool objdump -s "main\.test" test  // 这句话的意思是：解析可执行文件test，将其中的main包的test方法转成汇编代码。
TEXT main.test(SB) /home/sysublackbear/go/src/zhuowen.deng/test.go
  test.go:5		0x4820f0		64488b0c25f8ffffff	MOVQ FS:0xfffffff8, CX
  test.go:5		0x4820f9		483b6110		CMPQ 0x10(CX), SP
  test.go:5		0x4820fd		7639			JBE 0x482138
  test.go:5		0x4820ff		4883ec18		SUBQ $0x18, SP
  test.go:5		0x482103		48896c2410		MOVQ BP, 0x10(SP)
  test.go:5		0x482108		488d6c2410		LEAQ 0x10(SP), BP
  test.go:6		0x48210d		488d056c010100		LEAQ 0x1016c(IP), AX
  test.go:6		0x482114		48890424		MOVQ AX, 0(SP)
  test.go:6		0x482118		e803bdf8ff		CALL runtime.newobject(SB)  // 在堆上优化
  test.go:6		0x48211d		488b442408		MOVQ 0x8(SP), AX
  test.go:7		0x482122		48c700bbaa0000		MOVQ $0xaabb, 0(AX)
  test.go:8		0x482129		4889442420		MOVQ AX, 0x20(SP)
  test.go:8		0x48212e		488b6c2410		MOVQ 0x10(SP), BP
  test.go:8		0x482133		4883c418		ADDQ $0x18, SP
  test.go:8		0x482137		c3			RET
  test.go:5		0x482138		e8b3a1fcff		CALL runtime.morestack_noctxt(SB)
  test.go:5		0x48213d		ebb1			JMP main.test(SB)
```



但是当我们使用默认参数的时候，函数test会被main内联，此时结果就变得不同了。

```bash
[sysublackbear@centos-linux zhuowen.deng]$ go build -o test test.go
[sysublackbear@centos-linux zhuowen.deng]$ go tool objdump -s "main\.main" test
TEXT main.main(SB) /home/sysublackbear/go/src/zhuowen.deng/test.go
  test.go:11		0x4820f0		64488b0c25f8ffffff	MOVQ FS:0xfffffff8, CX
  test.go:11		0x4820f9		483b6110		CMPQ 0x10(CX), SP
  test.go:11		0x4820fd		7677			JBE 0x482176
  test.go:11		0x4820ff		4883ec50		SUBQ $0x50, SP
  test.go:11		0x482103		48896c2448		MOVQ BP, 0x48(SP)
  test.go:11		0x482108		488d6c2448		LEAQ 0x48(SP), BP
  test.go:7		0x48210d		48c7442430bbaa0000	MOVQ $0xaabb, 0x30(SP)
  test.go:12		0x482116		0f57c0			XORPS X0, X0
  test.go:12		0x482119		0f11442438		MOVUPS X0, 0x38(SP)
  test.go:12		0x48211e		488d442430		LEAQ 0x30(SP), AX
  test.go:12		0x482123		4889442408		MOVQ AX, 0x8(SP)
  test.go:12		0x482128		488d0551010100		LEAQ 0x10151(IP), AX
  test.go:12		0x48212f		48890424		MOVQ AX, 0(SP)
  test.go:12		0x482133		e8f899f8ff		CALL runtime.convT2E64(SB)
  test.go:12		0x482138		488b442418		MOVQ 0x18(SP), AX
  test.go:12		0x48213d		488b4c2410		MOVQ 0x10(SP), CX
  test.go:12		0x482142		48894c2438		MOVQ CX, 0x38(SP)
  test.go:12		0x482147		4889442440		MOVQ AX, 0x40(SP)
  test.go:12		0x48214c		488d442438		LEAQ 0x38(SP), AX
  test.go:12		0x482151		48890424		MOVQ AX, 0(SP)
  test.go:12		0x482155		48c744240801000000	MOVQ $0x1, 0x8(SP)
  test.go:12		0x48215e		48c744241001000000	MOVQ $0x1, 0x10(SP)
  test.go:12		0x482167		e8c49dffff		CALL fmt.Println(SB)
  test.go:13		0x48216c		488b6c2448		MOVQ 0x48(SP), BP
  test.go:13		0x482171		4883c450		ADDQ $0x50, SP
  test.go:13		0x482175		c3			RET
  test.go:11		0x482176		e875a1fcff		CALL runtime.morestack_noctxt(SB)
  test.go:11		0x48217b		e970ffffff		JMP main.main(SB)
```

显然内联优化的代码没有调用`newobject`在堆上分配内存。

**原因：**没有内联的时候，需要在两个栈帧之间（main函数栈帧，test函数栈帧）传递对象，因此会在堆上分配而不是返回一个失败栈帧的数据。而当内联后，它实际上就成了main栈帧内的局部变量，无须去堆上操作。



**注意：**

+ golang编译器支持“逃逸分析”，即它会在编译器通过构建调用图来分析局部变量是否会被外部引用，从而决定是否可直接分配在栈上。
+ 编译参数 `-gcflags "-m"`可输出编译优化信息，其中包括内联和逃逸分析。



然后，我们看下golang是怎么实现构造对象的。位于`runtime/malloc.go`的`newobject`函数，如下：

```go
// implementation of new builtin
// compiler (both frontend and SSA backend) knows the signature
// of this function
func newobject(typ *_type) unsafe.Pointer {
	return mallocgc(typ.size, typ, true)
}

// Allocate an object of size bytes.
// Small objects are allocated from the per-P cache's free lists.
// Large objects (> 32 kB) are allocated straight from the heap.
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	if gcphase == _GCmarktermination {
		throw("mallocgc called with gcphase == _GCmarktermination")
	}

	if size == 0 {
		return unsafe.Pointer(&zerobase)
	}

	if debug.sbrk != 0 {
		align := uintptr(16)
		if typ != nil {
			align = uintptr(typ.align)
		}
		return persistentalloc(size, align, &memstats.other_sys)
	}

	// assistG is the G to charge for this allocation, or nil if
	// GC is not currently active.
	var assistG *g
	if gcBlackenEnabled != 0 {
		// Charge the current user G for this allocation.
		assistG = getg()
		if assistG.m.curg != nil {
			assistG = assistG.m.curg
		}
		// Charge the allocation against the G. We'll account
		// for internal fragmentation at the end of mallocgc.
		assistG.gcAssistBytes -= int64(size)

		if assistG.gcAssistBytes < 0 {
			// This G is in debt. Assist the GC to correct
			// this before allocating. This must happen
			// before disabling preemption.
			gcAssistAlloc(assistG)
		}
	}

	// Set mp.mallocing to keep from being preempted by GC.
	mp := acquirem()
	if mp.mallocing != 0 {
		throw("malloc deadlock")
	}
	if mp.gsignal == getg() {
		throw("malloc during signal")
	}
	mp.mallocing = 1

	shouldhelpgc := false
	dataSize := size
  
  // 当前线程所绑定的cache
	c := gomcache()
	var x unsafe.Pointer
	noscan := typ == nil || typ.kind&kindNoPointers != 0
  
  // 小对象分配(32KB)
	if size <= maxSmallSize {
    // 没被扫描过 && 微小的内存
		if noscan && size < maxTinySize {
			// Tiny allocator.
			//
			// Tiny allocator combines several tiny allocation requests
			// into a single memory block. The resulting memory block
			// is freed when all subobjects are unreachable. The subobjects
			// must be noscan (don't have pointers), this ensures that
			// the amount of potentially wasted memory is bounded.
			//
			// Size of the memory block used for combining (maxTinySize) is tunable.
			// Current setting is 16 bytes, which relates to 2x worst case memory
			// wastage (when all but one subobjects are unreachable).
			// 8 bytes would result in no wastage at all, but provides less
			// opportunities for combining.
			// 32 bytes provides more opportunities for combining,
			// but can lead to 4x worst case wastage.
			// The best case winning is 8x regardless of block size.
			//
			// Objects obtained from tiny allocator must not be freed explicitly.
			// So when an object will be freed explicitly, we ensure that
			// its size >= maxTinySize.
			//
			// SetFinalizer has a special case for objects potentially coming
			// from tiny allocator, it such case it allows to set finalizers
			// for an inner byte of a memory block.
			//
			// The main targets of tiny allocator are small strings and
			// standalone escaping variables. On a json benchmark
			// the allocator reduces number of allocations by ~12% and
			// reduces heap size by ~20%.
			off := c.tinyoffset
			// Align tiny pointer for required (conservative) alignment.
			if size&7 == 0 {
        // 字节对齐，调整偏移量
				off = round(off, 8)
			} else if size&3 == 0 {
				off = round(off, 4)
			} else if size&1 == 0 {
				off = round(off, 2)
			}
      
      // 如果剩余空间足够
			if off+size <= maxTinySize && c.tiny != 0 {
				// The object fits into existing tiny block.
        // 返回指针，调整偏移量为下次分配做好准备
				x = unsafe.Pointer(c.tiny + off)
				c.tinyoffset = off + size
				c.local_tinyallocs++
				mp.mallocing = 0
				releasem(mp)
				return x
			}
			// Allocate a new maxTinySize block.
      // 否则，就开始获取新的tiny块
      // 就是从 sizeclass = 2的span.freelist 获取一个16字节的object
			span := c.alloc[tinySpanClass]
      
      // 如果没有可用的object，那么需要从central获取新的span
			v := nextFreeFast(span)
			if v == 0 {
        // 
				v, _, shouldhelpgc = c.nextFree(tinySpanClass)
			}
      
      // 初始化零值tiny块
			x = unsafe.Pointer(v)
			(*[2]uint64)(x)[0] = 0
			(*[2]uint64)(x)[1] = 0
			// See if we need to replace the existing tiny block with the new one
			// based on amount of remaining free space.
      // 对比新旧两个tiny块的剩余空间
      // 新块分配后，其tinyoffset = size,因此对比偏移量即可。
			if size < c.tinyoffset || c.tiny == 0 {
        // 用新块替换
				c.tiny = uintptr(x)
				c.tinyoffset = size
			}
      // 消费一个新的完整tiny块
			size = maxTinySize
		} else {
      // 普通小对象
			var sizeclass uint8
      
      // 查表，以确定sizeclass
			if size <= smallSizeMax-8 {
				sizeclass = size_to_class8[(size+smallSizeDiv-1)/smallSizeDiv]
			} else {
				sizeclass = size_to_class128[(size-smallSizeMax+largeSizeDiv-1)/largeSizeDiv]
			}
			size = uintptr(class_to_size[sizeclass])
			spc := makeSpanClass(sizeclass, noscan)
      // 从对应规格的span.freelist提取object
			span := c.alloc[spc]
			v := nextFreeFast(span)
			if v == 0 {
				v, span, shouldhelpgc = c.nextFree(spc)
			}
      // 清零（变量默认总是初始化为零值）
			x = unsafe.Pointer(v)
			if needzero && span.needzero != 0 {
				memclrNoHeapPointers(unsafe.Pointer(v), size)
			}
		}
	} else {
    // 大对象直接从heap分配span
		var s *mspan
		shouldhelpgc = true
		systemstack(func() {
			s = largeAlloc(size, needzero, noscan)
		})
		s.freeindex = 1
		s.allocCount = 1
    // span.start实际由 address >> pageShift 生成
		x = unsafe.Pointer(s.base())
		size = s.elemsize
	}

  // 在bitmap做标记...
  // 检查处罚条件，启动垃圾回收...
	var scanSize uintptr
	if !noscan {
		// If allocating a defer+arg block, now that we've picked a malloc size
		// large enough to hold everything, cut the "asked for" size down to
		// just the defer header, so that the GC bitmap will record the arg block
		// as containing nothing at all (as if it were unused space at the end of
		// a malloc block caused by size rounding).
		// The defer arg areas are scanned as part of scanstack.
		if typ == deferType {
			dataSize = unsafe.Sizeof(_defer{})
		}
		heapBitsSetType(uintptr(x), size, dataSize, typ)
		if dataSize > typ.size {
			// Array allocation. If there are any
			// pointers, GC has to scan to the last
			// element.
			if typ.ptrdata != 0 {
				scanSize = dataSize - typ.size + typ.ptrdata
			}
		} else {
			scanSize = typ.ptrdata
		}
		c.local_scan += scanSize
	}

	// Ensure that the stores above that initialize x to
	// type-safe memory and set the heap bits occur before
	// the caller can make x observable to the garbage
	// collector. Otherwise, on weakly ordered machines,
	// the garbage collector could follow a pointer to x,
	// but see uninitialized memory or stale heap bits.
	publicationBarrier()

	// Allocate black during GC.
	// All slots hold nil so no scanning is needed.
	// This may be racing with GC so do it atomically if there can be
	// a race marking the bit.
	if gcphase != _GCoff {
		gcmarknewobject(uintptr(x), size, scanSize)
	}

	if raceenabled {
		racemalloc(x, size)
	}

	if msanenabled {
		msanmalloc(x, size)
	}

	mp.mallocing = 0
	releasem(mp)

	if debug.allocfreetrace != 0 {
		tracealloc(x, size, typ)
	}

	if rate := MemProfileRate; rate > 0 {
		if rate != 1 && int32(size) < c.next_sample {
			c.next_sample -= int32(size)
		} else {
			mp := acquirem()
			profilealloc(mp, x, size)
			releasem(mp)
		}
	}

	if assistG != nil {
		// Account for internal fragmentation in the assist
		// debt now that we know it.
		assistG.gcAssistBytes -= int64(size - dataSize)
	}

	if shouldhelpgc {
		if t := (gcTrigger{kind: gcTriggerHeap}); t.test() {
			gcStart(t)
		}
	}

	return x
}
```

基本思路：

+ 大对象直接从heap获取span；
+ 小对象从`cache.alloc[sizeclass].freelist`获取object；
+ 微小对象组合使用`cache.tiny`。
