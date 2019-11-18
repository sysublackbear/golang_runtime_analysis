# 10.延迟

延迟调用(`defer`)的最大优势是，即便函数执行出错，依然能保证回收资源等操作得以执行。但如果对性能有要求，且错误能被控制，那么还是直接执行比较好。



## 10.1.实现细节

先看下`defer`是怎么实现的，通过一个示例：

**test.go**

```go
package main

import "fmt"

func main() {
    defer fmt.Println(0x11)
}
```

反汇编：

```bash
[sysublackbear@centos-linux zhuowen.deng]$ go build -o test test.go
[sysublackbear@centos-linux zhuowen.deng]$ go tool objdump -s "main\.main" test
TEXT main.main(SB) /home/sysublackbear/go/src/zhuowen.deng/test.go
  test.go:5		0x482070		64488b0c25f8ffffff	MOVQ FS:0xfffffff8, CX
  test.go:5		0x482079		483b6110		CMPQ 0x10(CX), SP
  test.go:5		0x48207d		0f8686000000		JBE 0x482109
  test.go:5		0x482083		4883ec58		SUBQ $0x58, SP
  test.go:5		0x482087		48896c2450		MOVQ BP, 0x50(SP)
  test.go:5		0x48208c		488d6c2450		LEAQ 0x50(SP), BP
  test.go:6		0x482091		0f57c0			XORPS X0, X0
  test.go:6		0x482094		0f11442440		MOVUPS X0, 0x40(SP)
  test.go:6		0x482099		488d05e0010100		LEAQ 0x101e0(IP), AX
  test.go:6		0x4820a0		4889442440		MOVQ AX, 0x40(SP)
  test.go:6		0x4820a5		488d05ec0e0400		LEAQ main.statictmp_0(SB), AX
  test.go:6		0x4820ac		4889442448		MOVQ AX, 0x48(SP)
  test.go:6		0x4820b1		488d442440		LEAQ 0x40(SP), AX
  test.go:6		0x4820b6		4889442410		MOVQ AX, 0x10(SP)
  test.go:6		0x4820bb		48c744241801000000	MOVQ $0x1, 0x18(SP)
  test.go:6		0x4820c4		48c744242001000000	MOVQ $0x1, 0x20(SP)
  test.go:6		0x4820cd		c7042430000000		MOVL $0x30, 0(SP)
  test.go:6		0x4820d4		488d054d6d0300		LEAQ 0x36d4d(IP), AX
  test.go:6		0x4820db		4889442408		MOVQ AX, 0x8(SP)
  test.go:6		0x4820e0		e84b2afaff		CALL runtime.deferproc(SB)
  test.go:6		0x4820e5		85c0			TESTL AX, AX
  test.go:6		0x4820e7		7510			JNE 0x4820f9
  test.go:7		0x4820e9		90			NOPL
  test.go:7		0x4820ea		e82133faff		CALL runtime.deferreturn(SB)
  test.go:7		0x4820ef		488b6c2450		MOVQ 0x50(SP), BP
  test.go:7		0x4820f4		4883c458		ADDQ $0x58, SP
  test.go:7		0x4820f8		c3			RET
  test.go:6		0x4820f9		90			NOPL
  test.go:6		0x4820fa		e81133faff		CALL runtime.deferreturn(SB)
  test.go:6		0x4820ff		488b6c2450		MOVQ 0x50(SP), BP
  test.go:6		0x482104		4883c458		ADDQ $0x58, SP
  test.go:6		0x482108		c3			RET
  test.go:5		0x482109		e862a1fcff		CALL runtime.morestack_noctxt(SB)
  test.go:5		0x48210e		e95dffffff		JMP main.main(SB)
```

可以看到：编译器将`defer`处理成两个函数调用，`deferproc`定义一个延迟调用对象，然后在函数结束前通过`deferreturn`完成最终调用。

和前面一样，对于这类参数不确定的都是用`funcval`处理，`siz`是目标函数参数长度。

**runtime2.go**

```go
type _defer struct {
	siz     int32 // includes both arguments and results
	started bool
	heap    bool
	sp      uintptr // 调用deferproc时的SP
	pc      uintptr // 调用deferproc时的IP
	fn      *funcval
	_panic  *_panic // panic that is running defer
	link    *_defer
}
```

**panic.go**

```go
func deferproc(siz int32, fn *funcval) { // arguments of fn follow fn
	if getg().m.curg != getg() {
		// go code on the system stack can't defer
		throw("defer on system stack")
	}

	// the arguments of fn are in a perilous state. The stack map
	// for deferproc does not describe them. So we can't let garbage
	// collection or stack copying trigger until we've copied them out
	// to somewhere safe. The memmove below does that.
	// Until the copy completes, we can only call nosplit routines.
	sp := getcallersp()
	argp := uintptr(unsafe.Pointer(&fn)) + unsafe.Sizeof(fn)
	callerpc := getcallerpc()

  // 创建d
	d := newdefer(siz)
	if d._panic != nil {
		throw("deferproc: d.panic != nil after newdefer")
	}
	d.fn = fn
	d.pc = callerpc
	d.sp = sp
	switch siz {
	case 0:
		// Do nothing.
	case sys.PtrSize:
		*(*uintptr)(deferArgs(d)) = *(*uintptr)(unsafe.Pointer(argp))
	default:
		memmove(deferArgs(d), unsafe.Pointer(argp), uintptr(siz))
	}

	// deferproc returns 0 normally.
	// a deferred func that stops a panic
	// makes the deferproc return 1.
	// the code the compiler generates always
	// checks the return value and jumps to the
	// end of the function if deferproc returns != 0.
	return0()
	// No code can go here - the C return register has
	// been set and must not be clobbered.
}
```

再看看`newdefer`的实现细节：

```go
func newdefer(siz int32) *_defer {
	var d *_defer
  
  // 参数长度对齐后，获取缓存等级
	sc := deferclass(uintptr(siz))
	gp := getg()
  
  // 未超出缓存大小
	if sc < uintptr(len(p{}.deferpool)) {
		pp := gp.m.p.ptr()
		if len(pp.deferpool[sc]) == 0 && sched.deferpool[sc] != nil {
			// Take the slow path on the system stack so
			// we don't grow newdefer's stack.
			systemstack(func() {
				lock(&sched.deferlock)
        // 如果P本地缓存已空，从全局提取一批到本地
				for len(pp.deferpool[sc]) < cap(pp.deferpool[sc])/2 && sched.deferpool[sc] != nil {
					d := sched.deferpool[sc]
					sched.deferpool[sc] = d.link
					d.link = nil
					pp.deferpool[sc] = append(pp.deferpool[sc], d)
				}
				unlock(&sched.deferlock)
			})
		}
    // 从本地缓存尾部提取
		if n := len(pp.deferpool[sc]); n > 0 {
			d = pp.deferpool[sc][n-1]
			pp.deferpool[sc][n-1] = nil
			pp.deferpool[sc] = pp.deferpool[sc][:n-1]
		}
	}
  // 新建。很显然分配的空间大小除_defer外，还有参数
	if d == nil {
		// Allocate new defer+args.
		systemstack(func() {
			total := roundupsize(totaldefersize(uintptr(siz)))
			d = (*_defer)(mallocgc(total, deferType, true))
		})
		if debugCachedWork {
			// Duplicate the tail below so if there's a
			// crash in checkPut we can tell if d was just
			// allocated or came from the pool.
			d.siz = siz
			d.link = gp._defer
			gp._defer = d
			return d
		}
	}
  // 将d保存到G._defer链表
	d.siz = siz
	d.heap = true
	d.link = gp._defer
	gp._defer = d
	return d
}
```

`defer`同样使用了二级缓存，`newdefer`先一次性为`defer`和参数分配空间，创建了d后，在挂到`G._defer`链表上。

然后再看`deferreturn`，自然是从`G._defer`获取并执行延迟函数了。

```go
func deferreturn(arg0 uintptr) {
	gp := getg()
  
  // 提取defer延迟对象
	d := gp._defer
	if d == nil {
		return
	}
  // 对比SP，避免调用其他栈帧的延迟函数。（arg0也就是 deferproc siz参数）
	sp := getcallersp()
	if d.sp != sp {
		return
	}

	// Moving arguments around.
	//
	// Everything called after this point must be recursively
	// nosplit because the garbage collector won't know the form
	// of the arguments until the jmpdefer can flip the PC over to
	// fn.
	switch d.siz {
	case 0:
		// Do nothing.
	case sys.PtrSize:
		*(*uintptr)(unsafe.Pointer(&arg0)) = *(*uintptr)(deferArgs(d))
	default:
    // 将延迟函数的参数复制到堆栈（这会覆盖掉size, fn,不过没有影响）
		memmove(unsafe.Pointer(&arg0), deferArgs(d), uintptr(d.siz))
	}
	fn := d.fn
	d.fn = nil
  // 调整G._defer链表
	gp._defer = d.link
  // 释放_defer对象，放回缓存
	freedefer(d)
  
  // 执行延迟函数
	jmpdefer(fn, uintptr(unsafe.Pointer(&arg0)))
}
```

再看汇编实现的`jmpdefer`。

**asm_amd64.s**

```bash
// func jmpdefer(fv *funcval, argp uintptr)
// argp is a caller SP.
// called from deferreturn.
// 1. pop the caller
// 2. sub 5 bytes from the callers return
// 3. jmp to the argument
TEXT runtime·jmpdefer(SB), NOSPLIT, $0-16
	MOVQ	fv+0(FP), DX	  // 延迟函数fn地址 
	MOVQ	argp+8(FP), BX	// argp+8是arg0地址，也就是main的SP
	LEAQ	-8(BX), SP	    // 将SP-8获取的其实是 call deferreturn 是压入的 main IP
	MOVQ	-8(SP), BP	    //  
	SUBQ	$5, (SP)	      // CALL指令长度减5，-5返回的就是call deferreturn的指令地址
	MOVQ	0(DX), BX       // 执行fn函数
	JMP	BX	// but first run the deferred function
```

首先通过`arg0`参数，也就是调用`deferproc`时压入的第一参数`siz`获取`main.main SP`。当`main`调用`deferreturn`时，用`SP-8`就可以获取当时保存的`main IP`值。因为`IP`保存了下一条指令地址，那么用该地址减去`CALL`指令长度，自然又回到了`main`调用`deferreturn`函数的位置。将这个计算得来的地址入栈，加上`jmpdefer`没有保存现场，那么延迟函数`fn RET`自然回到`CALL deferreturn`，如此就实现了多个`defer`延迟调用循环。



虽然整个调用堆栈的`defer`都挂在`G._defer`链表，但在`deferreturn`里面通过`sp`值的比对，可避免调用其他栈帧的延迟函数。

如中途用`Goexit`终止，它会负责处理整个调用堆栈的延迟函数。

**panic.go**

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



## 10.2.性能

延迟调用远不是一个`CALL`指令那么简单，会涉及很多内容。诸如对象分配，缓存，以及多次函数调用。在某些性能要求比较高的场合，应该避免使用`defer`。



## 10.3.错误

`panic`/`recover`的实现和`defer`息息相关，且过程算不上复杂。

**runtime2.go**

```go
type _panic struct {
	argp      unsafe.Pointer // pointer to arguments of deferred call run during panic; cannot move - known to liblink
	arg       interface{}    // argument to panic
	link      *_panic        // link to earlier panic
	recovered bool           // whether this panic is over
	aborted   bool           // the panic was aborted
}

type _defer struct {
	siz     int32 // includes both arguments and results
	started bool
	heap    bool
	sp      uintptr // sp at time of defer
	pc      uintptr
	fn      *funcval
	_panic  *_panic // panic that is running defer
	link    *_defer
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



**流程**：编译器将`panic`翻译成`gopanic`函数调用。它会将错误信息打包成`_panic`对象，并挂到`G._panic`链表的头部，然后遍历执行`G._defer`链表，检查是否`recover`。如被`recover`，则终止遍历执行，跳转到正常的`deferreturn`环节。否则执行整个调用堆栈的延迟函数后，显示异常信息，终止进程。

**panic.go**

```go
// The implementation of the predeclared function panic.
func gopanic(e interface{}) {
	gp := getg()
	if gp.m.curg != gp {
		print("panic: ")
		printany(e)
		print("\n")
		throw("panic on system stack")
	}

	if gp.m.mallocing != 0 {
		print("panic: ")
		printany(e)
		print("\n")
		throw("panic during malloc")
	}
	if gp.m.preemptoff != "" {
		print("panic: ")
		printany(e)
		print("\n")
		print("preempt off reason: ")
		print(gp.m.preemptoff)
		print("\n")
		throw("panic during preemptoff")
	}
	if gp.m.locks != 0 {
		print("panic: ")
		printany(e)
		print("\n")
		throw("panic holding locks")
	}

  // 新建_panic,挂到G._panic链表头部
	var p _panic
	p.arg = e
	p.link = gp._panic
	gp._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

	atomic.Xadd(&runningPanicDefers, 1)

  // 遍历执行G._defer（整个调用堆栈），直到某个recover
	for {
		d := gp._defer
		if d == nil {
			break
		}

		// If defer was started by earlier panic or Goexit (and, since we're back here, that triggered a new panic),
		// take defer off list. The earlier panic or Goexit will not continue running.
    // 如果defer已经执行，继续下一个
		if d.started {
			if d._panic != nil {
				d._panic.aborted = true
			}
			d._panic = nil
			d.fn = nil
			gp._defer = d.link  // 下一个
			freedefer(d)
			continue
		}

		// Mark defer as started, but keep on list, so that traceback
		// can find and update the defer's argument frame if stack growth
		// or a garbage collection happens before reflectcall starts executing d.fn.
    // 不移除defer，便于traceback输出所有调用堆栈信息
		d.started = true

		// Record the panic that is running the defer.
		// If there is a new panic during the deferred call, that panic
		// will find d in the list and will mark d._panic (this panic) aborted.
    // 将_panic保存到defer._panic
		d._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

    // 执行defer函数
    // p.argp地址很重要，defer里的recover以此来判断是否直接在defer内执行
    // reflectcall会修改p.argp
		p.argp = unsafe.Pointer(getargp(0))
		reflectcall(nil, unsafe.Pointer(d.fn), deferArgs(d), uint32(d.siz), uint32(d.siz))
		p.argp = nil

		// reflectcall did not panic. Remove d.
		if gp._defer != d {
			throw("bad defer entry in panic")
		}
    
    // 将已经执行的defer从G._defer链表移除
		d._panic = nil
		d.fn = nil
		gp._defer = d.link

		// trigger shrinkage to test stack copy. See stack_test.go:TestStackPanic
		//GC()

		pc := d.pc
		sp := unsafe.Pointer(d.sp) // must be pointer so it gets adjusted during stack copy
		freedefer(d)
    
    // 如果该defer执行了recover，那么recovered = true
		if p.recovered {
			atomic.Xadd(&runningPanicDefers, -1)

      // 移除当前recovered panic
			gp._panic = p.link
			// Aborted panics are marked but remain on the g.panic list.
			// Remove them from the list.
      
      // 移除aborted panic
			for gp._panic != nil && gp._panic.aborted {
				gp._panic = gp._panic.link
			}
			if gp._panic == nil { // must be done with signal
				gp.sig = 0
			}
			// Pass information about recovering frame to recovery.
      // recovery会跳转回defer.pc，也就是调用deferproc后，
      // 编译器会调用deferproc后插入比较指令，通过标志判断，跳转
      // 到deferreturn执行剩余的defer函数
			gp.sigcode0 = uintptr(sp)
			gp.sigcode1 = pc
			mcall(recovery)
			throw("recovery failed") // mcall should not return
		}
	}
  
  // 如果没有recovered，那么循环执行整个调用堆栈的延迟函数
  // 要么被后续 recover，要么崩溃

	// ran out of deferred calls - old-school panic now
	// Because it is unsafe to call arbitrary user code after freezing
	// the world, we call preprintpanics to invoke all necessary Error
	// and String methods to prepare the panic strings before startpanic.
	preprintpanics(gp._panic)

  // 如果没有捕获，显示错误信息后终止(exit)进程
	fatalpanic(gp._panic) // should not return
	*(*int)(nil) = 0      // not reached
}
```



和`panic`相比，`recover`函数除返回最后一个错误信息外，主要是设置`recovered`标志。注意，它会通过参数堆栈地址确认是否在延迟函数内被直接调用。



**总结：**

- 遇到处理不了的错误，找`panic`；
- `panic`有操守，退出前会执行本goroutine的`defer`，方式是原路返回(`reverse order`)；
- `panic`不多管，不是本goroutine的`defer`，不执行。



**panic.go**

```go
func gorecover(argp uintptr) interface{} {
	// Must be in a function running as part of a deferred call during the panic.
	// Must be called from the topmost function of the call
	// (the function used in the defer statement).
	// p.argp is the argument pointer of that topmost deferred function call.
	// Compare against argp reported by caller.
	// If they match, the caller is the one who can recover.
	gp := getg()
	p := gp._panic
	if p != nil && !p.recovered && argp == uintptr(p.argp) {
		p.recovered = true  // 如果该defer执行了recover，那么recovered = true
		return p.arg
	}
	return nil
}
```



**附录**：panic/recover的用法

**例子**：

```go
package main
 
import (
    "fmt"
)
 
func main() {
    defer func() {
        fmt.Println("1")
    }()
    defer func() {
        if err := recover(); err != nil {
            fmt.Println(err)
        }
    }()
    panic("fault")
    fmt.Println("2")
}
```

运行结果：

fault

1



**程序首先运行panic，出现故障，此时跳转到包含recover()的defer函数执行，recover捕获panic，此时panic就不继续传递。但是recover之后，程序并不会返回到panic那个点继续执行以后的动作，而是在recover这个点继续执行以后的动作，即执行上面的defer函数，输出1。**

**注意：**利用`recover`处理`panic`指令，必须利用`defer`在`panic`之前声明，否则当`panic`时，`recover`无法捕获到`panic`，无法防止`panic`扩散．
