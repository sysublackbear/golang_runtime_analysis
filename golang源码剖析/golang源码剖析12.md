# 15.unsafe

unsafe 包用于 Go 编译器，在编译阶段使用。从名字就可以看出来，它是不安全的，官方并不建议使用。我在用 unsafe 包的时候会有一种不舒服的感觉，可能这也是语言设计者的意图吧。

但是高阶的 Gopher，怎么能不会使用 unsafe 包呢？它可以绕过 Go 语言的类型系统，直接操作内存。例如，一般我们不能操作一个结构体的未导出成员，但是通过 unsafe 包就能做到。unsafe 包让我可以直接读写内存，还管你什么导出还是未导出。



## 15.1.源码

位于`unsafe/unsafe.go`，源码如下：

```go
package unsafe

type ArbitraryType int

type Pointer *ArbitraryType

func Sizeof(x ArbitraryType) uintptr

func Offsetof(x ArbitraryType) uintptr

func Alignof(x ArbitraryType) uintptr
```

+ `Arbitrary`：是任意的意思，也就是说 Pointer 可以指向任意类型，实际上它类似于C语言里的`void*`。
+ `Sizeof`：返回类型x所占据的字节数，但不包含x所指向的内容的大小。
+ `Offsetof`：返回结构体成员在内存中的位置离结构体起始处的字节数，所传参数必须是结构体的成员。
+ `Alignof`：返回m, n是指当类型进行内存对齐时，它分配到的内存地址能整除m。

注意到以上三个函数返回的结果都是 uintptr 类型，这和 unsafe.Pointer 可以相互转换。三个函数都是在编译期间执行，它们的结果可以直接赋给 `const 型变量`。另外，因为三个函数执行的结果和操作系统、编译器相关，所以是不可移植的。



## 15.2.unsafe.Pointer

1. 任何类型的指针和 unsafe.Pointer 可以相互转换。
2. uintptr 类型和 unsafe.Pointer 可以相互转换。

![type pointer uintptr](https://user-images.githubusercontent.com/7698088/58747453-1dbaee80-849e-11e9-8c75-2459f76792d2.png)

pointer 不能直接进行数学运算，但可以把它转换成 uintptr，对 uintptr 类型进行数学运算，再转换成 pointer 类型。

还有一点要注意的是，uintptr 并没有指针的语义，意思就是 uintptr 所指向的对象会被 gc 无情地回收。而 unsafe.Pointer 有指针语义，可以保护它所指向的对象在“有用”的时候不会被垃圾回收。



我们看一个例子，来看下`uintptr`和`unsafe.Pointer`是如何相互转换的。

```go
func main() {
    s := make([]int, 9, 20)
    var Len = *(*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + uintptr(8)))
    fmt.Println(Len, len(s)) // 9 9

    var Cap = *(*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + uintptr(16)))
    fmt.Println(Cap, cap(s)) // 20 20
}
```



## 15.3.Offsetof获取成员偏移量

对于一个结构体，通过 offset 函数可以获取结构体成员的偏移量，进而获取成员的地址，读写该地址的内存，就可以达到改变成员值的目的。

这里有一个内存分配相关的事实：结构体会被分配一块连续的内存，结构体的地址也代表了第一个成员的地址。

我们可以看一个例子：

```go
package main

import (
    "fmt"
    "unsafe"
)

type Programmer struct {
    name string
    language string
}

func main() {
    p := Programmer{"stefno", "go"}
    fmt.Println(p)
    
    name := (*string)(unsafe.Pointer(&p))
    *name = "qcrao"

    lang := (*string)(unsafe.Pointer(uintptr(unsafe.Pointer(&p)) + unsafe.Offsetof(p.language)))
    *lang = "Golang"

    fmt.Println(p)
}
```



## 15.4.string和slice的相互切换（共享底层结构）

```go
func string2bytes(s string) []byte {
    stringHeader := (*reflect.StringHeader)(unsafe.Pointer(&s))

    bh := reflect.SliceHeader{
        Data: stringHeader.Data,
        Len:  stringHeader.Len,
        Cap:  stringHeader.Len,
    }

    return *(*[]byte)(unsafe.Pointer(&bh))
}

func bytes2string(b []byte) string{
    sliceHeader := (*reflect.SliceHeader)(unsafe.Pointer(&b))

    sh := reflect.StringHeader{
        Data: sliceHeader.Data,
        Len:  sliceHeader.Len,
    }

    return *(*string)(unsafe.Pointer(&sh))
}
```



# 16.pprof

![pprof 采集的信息类型](https://user-images.githubusercontent.com/7698088/68523507-3ce36500-02f5-11ea-8e8f-438c9ef2b9f8.png)

Go 语言自带的 pprof 库就可以分析程序的运行情况，并且提供可视化的功能。它包含两个相关的库：

- runtime/pprof
  对于只跑一次的程序，例如每天只跑一次的离线预处理程序，调用 pprof 包提供的函数，手动开启性能数据采集。
- net/http/pprof
  对于在线服务，对于一个 HTTP Server，访问 pprof 提供的 HTTP 接口，获得性能数据。当然，实际上这里底层也是调用的 runtime/pprof 提供的函数，封装成接口对外提供网络访问。



## 16.1.pprof的原理

1. 当 CPU 性能分析启用后（profile），Go runtime 会每 10ms 就暂停一下，记录当前运行的 goroutine 的调用堆栈及相关数据。当性能分析数据保存到硬盘后，我们就可以分析代码中的热点了。
2. 内存性能分析（allocs）则是在堆（Heap）分配的时候，记录一下调用堆栈。默认情况下，是每 1000 次分配，取样一次，这个数值可以改变。栈(Stack)分配 由于会随时释放，因此不会被内存分析所记录。由于内存分析是取样方式，并且也**因为其记录的是分配内存，而不是使用内存**。因此使用内存性能分析工具来准确判断程序具体的内存使用是比较困难的。
3. 阻塞分析（blocks）是一个很独特的分析，它有点儿类似于 CPU 性能分析，但是它所记录的是 goroutine 等待资源所花的时间。阻塞分析对分析程序并发瓶颈非常有帮助，阻塞性能分析可以显示出什么时候出现了大批的 goroutine 被阻塞了。阻塞性能分析是特殊的分析工具，在排除 CPU 和内存瓶颈前，不应该用它来分析。



pprof进入交互式模式之后，比较常用的有 `top`、`list`、`web` 等命令。



# 17.反射

## 17.1.为什么要用反射

需要反射的 2 个常见场景：

1. 有时你需要编写一个函数，但是并不知道传给你的参数类型是什么，可能是没约定好；也可能是传入的类型很多，这些类型并不能统一表示。这时反射就会用的上了。
2. 有时候需要根据某些条件决定调用哪个函数，比如根据用户的输入来决定。这时就需要对函数和函数的参数进行反射，在运行期间动态地执行函数。

在讲反射的原理以及如何用之前，还是说几点不使用反射的理由：

1. 与反射相关的代码，经常是难以阅读的。在软件工程中，代码可读性也是一个非常重要的指标。
2. Go 语言作为一门静态语言，编码过程中，编译器能提前发现一些类型错误，但是对于反射代码是无能为力的。所以包含反射相关的代码，很可能会运行很久，才会出错，这时候经常是直接 panic，可能会造成严重的后果。
3. 反射对性能影响还是比较大的，比正常代码运行速度慢一到两个数量级。所以，对于一个项目中处于运行效率关键位置的代码，尽量避免使用反射特性。



## 17.2.反射的基本函数

reflect 包里定义了一个接口和一个结构体，即 `reflect.Type` 和 `reflect.Value`，它们提供很多函数来获取存储在接口里的类型信息。

`reflect.Type` 主要提供关于类型相关的信息，所以它和 `_type` 关联比较紧密；`reflect.Value` 则结合 `_type` 和 `data` 两者，因此程序员可以获取甚至改变类型的值。

reflect 包中提供了两个基础的关于反射的函数来获取上述的接口和结构体：

```go
func TypeOf(i interface{}) Type 
func ValueOf(i interface{}) Value
```

`TypeOf` 函数用来提取一个接口中值的类型信息。由于它的输入参数是一个空的 `interface{}`，调用此函数时，实参会先被转化为 `interface{}`类型。这样，实参的类型信息、方法集、值信息都存储到 `interface{}` 变量里了。

```go
func TypeOf(i interface{}) Type {
    eface := *(*emptyInterface)(unsafe.Pointer(&i))
    return toType(eface.typ)
}

type emptyInterface struct {
    typ  *rtype
    word unsafe.Pointer
}
```

这里的 `emptyInterface` 和上面提到的 `eface` 是一回事（字段名略有差异，字段是相同的），且在不同的源码包：前者在 `reflect` 包，后者在 `runtime` 包。 `eface.typ` 就是动态类型。



其中，返回值`Type`实际上是一个接口，定义了很多方法，用来获取类型相关的各种信息，而 `*rtype` 实现了 `Type` 接口。

```go
type Type interface {
    // 所有的类型都可以调用下面这些函数

    // 此类型的变量对齐后所占用的字节数
    Align() int
    
    // 如果是 struct 的字段，对齐后占用的字节数
    FieldAlign() int

    // 返回类型方法集里的第 `i` (传入的参数)个方法
    Method(int) Method

    // 通过名称获取方法
    MethodByName(string) (Method, bool)

    // 获取类型方法集里导出的方法个数
    NumMethod() int

    // 类型名称
    Name() string

    // 返回类型所在的路径，如：encoding/base64
    PkgPath() string

    // 返回类型的大小，和 unsafe.Sizeof 功能类似
    Size() uintptr

    // 返回类型的字符串表示形式
    String() string

    // 返回类型的类型值
    Kind() Kind

    // 类型是否实现了接口 u
    Implements(u Type) bool

    // 是否可以赋值给 u
    AssignableTo(u Type) bool

    // 是否可以类型转换成 u
    ConvertibleTo(u Type) bool

    // 类型是否可以比较
    Comparable() bool

    // 下面这些函数只有特定类型可以调用
    // 如：Key, Elem 两个方法就只能是 Map 类型才能调用
    
    // 类型所占据的位数
    Bits() int

    // 返回通道的方向，只能是 chan 类型调用
    ChanDir() ChanDir

    // 返回类型是否是可变参数，只能是 func 类型调用
    // 比如 t 是类型 func(x int, y ... float64)
    // 那么 t.IsVariadic() == true
    IsVariadic() bool

    // 返回内部子元素类型，只能由类型 Array, Chan, Map, Ptr, or Slice 调用
    Elem() Type

    // 返回结构体类型的第 i 个字段，只能是结构体类型调用
    // 如果 i 超过了总字段数，就会 panic
    Field(i int) StructField

    // 返回嵌套的结构体的字段
    FieldByIndex(index []int) StructField

    // 通过字段名称获取字段
    FieldByName(name string) (StructField, bool)

    // FieldByNameFunc returns the struct field with a name
    // 返回名称符合 func 函数的字段
    FieldByNameFunc(match func(string) bool) (StructField, bool)

    // 获取函数类型的第 i 个参数的类型
    In(i int) Type

    // 返回 map 的 key 类型，只能由类型 map 调用
    Key() Type

    // 返回 Array 的长度，只能由类型 Array 调用
    Len() int

    // 返回类型字段的数量，只能由类型 Struct 调用
    NumField() int

    // 返回函数类型的输入参数个数
    NumIn() int

    // 返回函数类型的返回值个数
    NumOut() int

    // 返回函数类型的第 i 个值的类型
    Out(i int) Type

    // 返回类型结构体的相同部分
    common() *rtype
    
    // 返回类型结构体的不同部分
    uncommon() *uncommonType
}
```

返回的 `rtype`类型，它和上一篇文章讲到的 `_type` 是一回事，而且源代码里也注释了：两边要保持同步：

```go
type rtype struct {
    size       uintptr
    ptrdata    uintptr
    hash       uint32
    tflag      tflag
    align      uint8
    fieldAlign uint8
    kind       uint8
    alg        *typeAlg
    gcdata     *byte
    str        nameOff
    ptrToThis  typeOff
}
```



再看一下`ValueOf`函数：

```go
func ValueOf(i interface{}) Value {
	if i == nil {
		return Value{}
	}

	// TODO: Maybe allow contents of a Value to live on the stack.
	// For now we make the contents always escape to the heap. It
	// makes life easier in a few places (see chanrecv/mapassign
	// comment below).
	escapes(i)

	return unpackEface(i)
}

// 分解eface
func unpackEface(i interface{}) Value {
	e := (*emptyInterface)(unsafe.Pointer(&i))
	// NOTE: don't read e.word until we know whether it is really a pointer or not.
	t := e.typ
	if t == nil {
		return Value{}
	}
	f := flag(t.Kind())
	if ifaceIndir(t) {
		f |= flagIndir
	}
	return Value{t, e.word, f}
}
```

从源码看，比较简单：将先将 `i` 转换成 `*emptyInterface` 类型， 再将它的 `typ` 字段和 `word` 字段以及一个标志位字段组装成一个 `Value` 结构体，而这就是 `ValueOf` 函数的返回值，它包含类型结构体指针、真实数据的地址、标志位。



Value 结构体定义了很多方法，通过这些方法可以直接操作 Value 字段 ptr 所指向的实际数据：

```go
// 设置切片的 len 字段，如果类型不是切片，就会panic
 func (v Value) SetLen(n int)
 
 // 设置切片的 cap 字段
 func (v Value) SetCap(n int)
 
 // 设置字典的 kv
 func (v Value) SetMapIndex(key, val Value)

 // 返回切片、字符串、数组的索引 i 处的值
 func (v Value) Index(i int) Value
 
 // 根据名称获取结构体的内部字段值
 func (v Value) FieldByName(name string) Value
 
 // ……
```

`Value` 字段还有很多其他的方法。例如：

```go
// 用来获取 int 类型的值
func (v Value) Int() int64

// 用来获取结构体字段（成员）数量
func (v Value) NumField() int

// 尝试向通道发送数据（不会阻塞）
func (v Value) TrySend(x reflect.Value) bool

// 通过参数列表 in 调用 v 值所代表的函数（或方法
func (v Value) Call(in []Value) (r []Value) 

// 调用变参长度可变的函数
func (v Value) CallSlice(in []Value) []Value 
```

不一一列举了，反正是非常多。可以去 `src/reflect/value.go` 去看看源码。



## 17.3.Interface(), Type()和Value()三者关系

通过 `Type()` 方法和 `Interface()` 方法可以打通 `interface`、`Type`、`Value` 三者。Type() 方法也可以返回变量的类型信息，与 reflect.TypeOf() 函数等价。Interface() 方法可以将 Value 还原成原来的 interface。

![三者关系](https://user-images.githubusercontent.com/7698088/57130652-bb060280-6dcc-11e9-9c63-6e2bc4e33509.png)

总结一下：`TypeOf()` 函数返回一个接口，这个接口定义了一系列方法，利用这些方法可以获取关于类型的所有信息； `ValueOf()` 函数返回一个结构体变量，包含类型信息以及实际值。

![value rtype](https://user-images.githubusercontent.com/7698088/56848267-6f111480-6919-11e9-826f-a809093d17ea.png)

上图中，`rtye` 实现了 `Type` 接口，是所有类型的公共部分。emptyface 结构体和 eface 其实是一个东西，而 rtype 其实和 _type 是一个东西，只是一些字段稍微有点差别，比如 emptyface 的 word 字段和 eface 的 data 字段名称不同，但是数据型是一样的。



## 17.4.反射三大定律

1. 反射将接口变量转换成反射对象 Type 和 Value；
2. 反射可以通过反射对象 Value 还原成原先的接口变量；
3. 反射可以用来修改一个变量的值，前提是这个值可以被修改。



## 17.5.未导出成员

利用反射机制，**对于结构体中未导出成员，可以读取，但不能修改其值。

注意，正常情况下，代码是不能读取结构体未导出成员的，但通过反射可以越过这层限制。另外，通过反射，结构体中可以被修改的成员只有是导出成员，也就是字段名的首字母是大写的。

可以看下面例子：

```go
package main

import (
    "reflect"
    "fmt"
)

type Child struct {
    Name     string
    handsome bool
}

func main() {
    qcrao := Child{Name: "qcrao", handsome: true}

    v := reflect.ValueOf(&qcrao)

    f := v.Elem().FieldByName("Name")
    fmt.Println(f.String())

    f.SetString("stefno")
    fmt.Println(f.String())

    f = v.Elem().FieldByName("handsome")
    
    // 这一句会导致 panic，因为 handsome 字段未导出
    //f.SetBool(true)
    fmt.Println(f.Bool())
}
```



