# 18.sync.WaitGroup 实现原理



## 18.1.前言

WaitGroup 是golang开发过程中经常使用的并发控制技术。

WaitGroup，可以理解为Wait-Goroutine-Group，即等待一组goroutine 结束。比如某个goroutine 需要等待其他几个goroutine 全部完成，那么使用WaitGroup 可以轻松实现。

下面程序展示了一个goroutine 等待另外两个goroutine 结束的例子。

```go
package main

import (
    "fmt"
    "time"
    "sync"
)

func main() {
    var wg sync.WaitGroup

    wg.Add(2) //设置计数器，数值即为goroutine的个数
    go func() {
        //Do some work
        time.Sleep(1*time.Second)

        fmt.Println("Goroutine 1 finished!")
        wg.Done() //goroutine执行结束后将计数器减1
    }()

    go func() {
        //Do some work
        time.Sleep(2*time.Second)

        fmt.Println("Goroutine 2 finished!")
        wg.Done() //goroutine执行结束后将计数器减1
    }()

    wg.Wait() //主goroutine阻塞等待计数器变为0
    fmt.Printf("All Goroutine finished!")
}
```

简单地说，上面的程序，`wg`内部维护了一个计数器：

1. 启动goroutine前将计数器通过`wg.Add(2)`将计数器设置为待启动的goroutine个数。
2. 启动goroutine后，使用`wg.Wait()`方法阻塞自己，等待计数器变为0。
3. 每个goroutine执行结束通过`wg.Done()`方法将计数器值减1。
4. 计数器变为0后，阻塞的goroutine被唤醒。

下面介绍它的实现原理。



## 18.2.基础知识

### 18.2.1.信号量

信号量是Unix系统提供的一种保护共享资源的机制，用于防止多个线程同时访问某个资源。

可简单理解为信号量为一个数值：

+ 当信号量 > 0 时，表示资源可用，获取信号量时系统自动将信号量减1(`runtime_Semacquire`）；
+ 当信号量 == 0 时，表示资源暂不可用，获取信号量时，当前线程会进入睡眠，当信号量为正被唤醒(`runtime_Semrelease`)。

WaitGroup的实现正是使用了信号量。



## 18.3.数据结构

源码包中`src/sync/waitgroup.go:WaitGroup`定义了其数据结构：

```go
type WaitGroup struct {
	noCopy noCopy

	// 64-bit value: high 32 bits are counter, low 32 bits are waiter count.
	// 64-bit atomic operations require 64-bit alignment, but 32-bit
	// compilers do not ensure it. So we allocate 12 bytes and then use
	// the aligned 8 bytes in them as state, and the other 4 as storage
	// for the sema.
	state1 [3]uint32
}
```

state1是个长度为3的数组，其中包含了state和一个信号量，而state实际上是两个计数器：

+ counter：当前还未执行结束的goroutine计数器；
+ waiter count：等待goroutine-group结束的goroutine数量，即有多少个等候者；
+ semaphore：信号量。

考虑到字节是否对齐，三者出现的位置不同，为简单起见，依照字节已对齐情况下，三者在内存中的位置如下所示：

![3d44fe27957013aa82eca2923cafe50d909](https://github.com/sysublackbear/golang_runtime_analysis/blob/master/img/image-20200106110530876.png)

WaitGroup对外提供三个接口：

- Add(delta int): 将delta值加到counter中
- Wait()： waiter递增1，并阻塞等待信号量semaphore
- Done()： counter递减1，按照waiter数值释放相应次数信号量

下面分别介绍这三个函数的实现细节。



## 18.4.Add(delta int)

代码如下：

```go
func (wg *WaitGroup) Add(delta int) {
	statep, semap := wg.state()  // 获取state和semaphore地址指针
    // race.Enabled -- golang自带竞态检测器逻辑
    // go命令加--race开启静态检测器检测开关
	if race.Enabled {
		_ = *statep // trigger nil deref early
		if delta < 0 {
			// Synchronize decrements with Wait.
			race.ReleaseMerge(unsafe.Pointer(wg))
		}
		race.Disable()
		defer race.Enable()
	}
    // 把delta左移32位累加到state,即累加到counter中
	state := atomic.AddUint64(statep, uint64(delta)<<32)
	v := int32(state >> 32)  // 获取counter的值
	w := uint32(state)       // 获取waiter的值
	if race.Enabled && delta > 0 && v == int32(delta) {
		// The first increment must be synchronized with Wait.
		// Need to model this as a read, because there can be
		// several concurrent wg.counter transitions from 0.
		race.Read(unsafe.Pointer(semap))
	}
	if v < 0 {  // 经过累加后counter值变为负值,panic
		panic("sync: negative WaitGroup counter")
	}
    // w不够用，也要panic
	if w != 0 && delta > 0 && v == int32(delta) {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}
    // 经过累加后，此时，counter >= 0
    // 如果counter为正，说明不需要释放信号量，直接退出
    // 如果waiter为零，说明没有等待者，也不需要释放信号量了，直接退出
	if v > 0 || w == 0 {
		return
	}
	// This goroutine has set counter to 0 when waiters > 0.
	// Now there can't be concurrent mutations of state:
	// - Adds must not happen concurrently with Wait,
	// - Wait does not increment waiters if it sees counter == 0.
	// Still do a cheap sanity check to detect WaitGroup misuse.
	if *statep != state {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}
	// Reset waiters count to 0.
    // 此时, counter一定等于0,而waiter一定大于0(内部维护waiter，不会出现小于0的情况)
    // 先把counter置为0,再释放waiter个数的信号量
    
    // 来到这里，意味着所有的子goroutine已经Done
	*statep = 0
	for ; w != 0; w-- {
		runtime_Semrelease(semap, false, 0)  // 释放信号量,执行一次释放一个,唤醒一个等待者
	}
}

func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
	if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
		return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
	} else {
		return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
	}
}
```

`Add`函数做了两件事：

1. 把`delta`值累加到counter中，因为`delta`可以为负值，也就是说counter有可能变为0或者负值；
2. 当counter值变为0的时候，相当于所有的子任务Done了，根据waiter数值释放等量的信号量，把等待的goroutine全部唤醒，如果counter变为负值，则panic。



## 18.5.Wait()

代码如下：

```go
// Wait blocks until the WaitGroup counter is zero.
func (wg *WaitGroup) Wait() {
	statep, semap := wg.state()  // 获取state和semaphore地址指针
	if race.Enabled {
		_ = *statep // trigger nil deref early
		race.Disable()
	}
	for {
		state := atomic.LoadUint64(statep)  // 获取state值
		v := int32(state >> 32)             // 获取counter值
		w := uint32(state)                  // 获取waiter值
		if v == 0 {
            // 如果counter值为0,说明所有的goroutine都退出了,不需要等待,直接返回
			// Counter is 0, no need to wait.
			if race.Enabled {
				race.Enable()
				race.Acquire(unsafe.Pointer(wg))
			}
			return
		}
		// Increment waiters count.
        // 使用CAS（比较交换算法）累加waiter，累加可能会失败，失败后for loop下次重试
		if atomic.CompareAndSwapUint64(statep, state, state+1) {
			if race.Enabled && w == 0 {
				// Wait must be synchronized with the first Add.
				// Need to model this is as a write to race with the read in Add.
				// As a consequence, can do the write only for the first waiter,
				// otherwise concurrent Waits will race with each other.
				race.Write(unsafe.Pointer(semap))
			}
			runtime_Semacquire(semap)  // 累加成功后，等待信号量唤醒自己
			if *statep != 0 {
				panic("sync: WaitGroup is reused before previous Wait has returned")
			}
			if race.Enabled {
				race.Enable()
				race.Acquire(unsafe.Pointer(wg))
			}
			return
		}
	}
}
```

`Wait`函数也做了两件事：

1. 累加waiter；
2. 阻塞等待信号量。



## 18.6.Done()

Done()只做一件事，即把counter减1，我们知道Add()可以接受负值，所以Done实际上只是调用了Add(-1)。

```go
func (wg *WaitGroup) Done() {
	wg.Add(-1)
}
```

Done()的执行逻辑就转到了Add()，实际上也正是最后一个完成的goroutine把等待者唤醒的。



## 18.7.注意

+ Add()操作必须早于Wait()，否则会panic；
+ Add()设置的值必须与实际等待的goroutine个数一致，否则也会panic。



## 18.8.协程屏障用到的技术

+ CAS原子操作，同步原语；
+ 计数值（可以是信号量，可以是共享变量，等等）。
