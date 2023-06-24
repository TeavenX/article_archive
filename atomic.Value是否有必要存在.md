TL;DR：有必要

前几天公司内论坛有人提了这么一个问题：

> 看了一遍 atomic 源码，没有理解 `atomic.Value` 的使用场景，网上的用法基本上都是用在配置更新上，感觉我用 machine word 原子读写也能保证原子性，那为啥还要有atomc.Value呢

然后贴了一段代码，大致的逻辑是这样

```go
package main

import (
	"time"
)

type Config struct {
	Name string
	Age  int
}

func main() {
	conf := &Config{Name: "foo", Age: 10}

	go func() {
		for {
			time.Sleep(time.Second)
			conf = &Config{Name: "bar", Age: 20}
			println("new config:", conf.Name, conf.Age)
		}
	}()

	for {
		time.Sleep(time.Second)
		println("current config:", conf.Name, conf.Age)
	}
}

```

他在第 18 行 `conf = &Config{Name: "bar", Age: 20}` ，通过`指针赋值`的方式替换了conf变量。

不仔细想好像确实没毛病：uintptr 替换，一条指令就搞完了，操作是原子的。

## 分析操作背景

上面操作主要面向的是多 goroutine 针对同一个变量进行读写。按照管用的套路先上 `data race detector` 看看工具分析结果。结果很明显，确实存在 `data race`

![CleanShot 2023-06-22 at 19.06.55@2x](https://teaven.oss-cn-shenzhen.aliyuncs.com/202306221907092.png)

如果换成 `atomic.Value` 就没有告警

```go
package main

import (
	"sync/atomic"
	"time"
)

type Config struct {
	Name string
	Age  int
}

func main() {
	conf := atomic.Value{}
	conf.Store(&Config{Name: "foo", Age: 10})

	go func() {
		for {
			time.Sleep(time.Second)
			conf.Store(&Config{Name: "bar", Age: 20})
			t := conf.Load().(*Config)
			println("new config:", t.Name, t.Age)
		}
	}()

	for {
		time.Sleep(time.Second)
		t := conf.Load().(*Config)
		println("current config:", t.Name, t.Age)
	}
}

```

![CleanShot 2023-06-22 at 19.15.49@2x](https://teaven.oss-cn-shenzhen.aliyuncs.com/202306221916369.png)

## 分析产生竟态的原因

可以从两个角度分析这个问题：1. data-race detector是如何检测数据竞争的；2. atomc.Value做了什么操作

### data-race detector是如何检测数据竞争的

go语言的竟态分析器主要是从Google内部针对 Chromium 做的 ThreadSanitizer 运行时库[1]中集成过来的。核心逻辑包含在 ThreadSanitizer 算法中，这个算法比较复杂，涉及到大量的数学证明，这里先认为检测结果是正确的，深入去看有点跑偏了。换一个相对简单的途径

### atomc.Value做了什么操作

#### 源码分析

atomic.value包比较简单，一共200行不到的代码。算上runtime/internal/atomic的汇编也就400行不到代码。我们把关键的几个部分拿出来分析下。

```go
type Value struct {
	v any
}
```

首先atomc.Value是一个结构体，结构体中只有一个v字段用来存放store的值，类型是any(interface{})。然后我们看下store操作做了什么事情

```go
var firstStoreInProgress byte

func (v *Value) Store(val any) {
	if val == nil {
		panic("sync/atomic: store of nil value into Value")
	}
	vp := (*ifaceWords)(unsafe.Pointer(v))
	vlp := (*ifaceWords)(unsafe.Pointer(&val))
	for {
		typ := LoadPointer(&vp.typ)
		if typ == nil {
			// Attempt to start first store.
			// Disable preemption so that other goroutines can use
			// active spin wait to wait for completion.
			runtime_procPin()
			if !CompareAndSwapPointer(&vp.typ, nil, unsafe.Pointer(&firstStoreInProgress)) {
				runtime_procUnpin()
				continue
			}
			// Complete first store.
			StorePointer(&vp.data, vlp.data)
			StorePointer(&vp.typ, vlp.typ)
			runtime_procUnpin()
			return
		}
		if typ == unsafe.Pointer(&firstStoreInProgress) {
			// First store in progress. Wait.
			// Since we disable preemption around the first store,
			// we can wait with active spinning.
			continue
		}
		// First store completed. Check type and overwrite data.
		if typ != vlp.typ {
			panic("sync/atomic: store of inconsistently typed value into Value")
		}
		StorePointer(&vp.data, vlp.data)
		return
	}
}

```

- 第7、8行用到了ifaceWords，在atomic包中的定义，ifaceWords就是interface{} 在当前包内的表示（为了方便取typ和data两个字段）

```go
// ifaceWords is interface{} internal representation.
type ifaceWords struct {
	typ  unsafe.Pointer
	data unsafe.Pointer
}
```

- 第9行的for循环实现配合下文的continue实现自旋效果

- 第10行先取了vp（原变量）类型的地址，后面会用到

- 第11行判断当前地址是否是nil（没有store过，也就是当前store操作是首次store）

- 第15行执行了runtime_procPin函数，该函数在当前包是一个shadow函数，真正的逻辑在runtime包内，主要逻辑是将当前G与P进行绑定，使P处于不可抢占状态，保证当前操作在unpin之前不会被调度器打断（如果这里被打断了，可能出现cas操作完还来不及更新typ和data，会使得其他G的store操作始终处于spin阶段，直到当前G又重新被调度并执行完赋值；也可能会出现设置完typ，但没有设置data就被打断，导致其他G取到了错误的值，而当前G又被重新调度到后，更新了一个过期的data上去）

- 第16行用到了firstStoreInProgress变量，声明在第一行是一个byte类型的零值变量（单个字节，不会存在撕裂写入的可能，应该没有4位的操作系统吧），在这里用cas需要配合26行来看，相当于设置了一个锁：第16行的cas先上了乐观锁，当前v依然为nil时才进行赋值，否则解除G与P的绑定进入下一轮自旋。这里赋值修改了v的原始类型为firstStoreInProgress的类型，设置的很巧妙，因为当v是nil时，tpy是什么都没有影响，而如果当前v的typ不是nil，就到了26行再做一个判断，如果typ是firstStoreInProgress的类型，就意味着第一次的store操作还在进行中，进入下一轮自旋

- 第21-22行更新v的类型（typ）和数据（data）为新v的值，store操作几乎就完成了

- 第23行执行runtime_procUnpin函数，解除G和P的绑定，首次的store操作完成

- 第33行对新store的值进行判断，如果类型不一致直接Panic

- 第36行存储新的data

再看下Load的部分就简单多了

```go
func (v *Value) Load() (val any) {
	vp := (*ifaceWords)(unsafe.Pointer(v))
	typ := LoadPointer(&vp.typ)
	if typ == nil || typ == unsafe.Pointer(&firstStoreInProgress) {
		// First store not yet completed.
		return nil
	}
	data := LoadPointer(&vp.data)
	vlp := (*ifaceWords)(unsafe.Pointer(&val))
	vlp.typ = typ
	vlp.data = data
	return
}
```

- 第4行先判断当前v是否还被首次store给锁住，如果锁住意味着还没有store完，直接返回nil
- 第8-11行，将typ和data组装到一个interface{}中返回

针对一开始的问题，我们单纯考虑Store（简单写）和Load（简单读）两个场景就够了，这两个也是最容易疑惑的场景。但是上面两段代码分析完好像并没有直观给到实现上一致性保证，甚至比直接赋值指针来的复杂且容易出问题的多。不过前面分析其实遗漏了两个函数`CompareAndSwapPointer`和`StorePointer`。一个CAS + Store就解决了吗？我们看下具体的实现，这两个函数包括一众的atomic操作在Go语言内都是汇编实现的

> 不同架构汇编实现完全不一样，线上跑基本上是x86平台的，这里针对x86平台的汇编（AMD64指令集）实现进行分析

```assembly
// atomic_amd64.s
TEXT ·Cas64(SB), NOSPLIT, $0-25
	MOVQ	ptr+0(FP), BX
	MOVQ	old+8(FP), AX
	MOVQ	new+16(FP), CX
	LOCK
	CMPXCHGQ	CX, 0(BX)
	SETEQ	ret+24(FP)
	RET
```

CAS操作值得关注的是第6-8行。

##### LOCK

LOCK是一个指令前缀，后面必须跟`read-modify-write`指令，有点类似于下一条指令的装饰器。

早期的LOCK实现会直接锁内存总线，防止其他核心再通过总线和内存通信，这种实现方式性能不够理想，intel在后续架构迭代中引入了一个优化：如果数据已经缓存在CPU cache中，则锁缓存，否则锁总线

##### CMPXCHGQ

CMPXCHGQ对应硬件指令其实是CMPXCHG，这里Q代表了8个字节。`CMPXCHGQ	CX, 0(BX)`的含义是：比较BX寄存器（addr）和AX寄存器（old）的值是否相等，如果相等就将CX寄存器（new）的值写入BX寄存器（addr）

##### SETEQ

SETEQ配合CMPXCHGQ设置返回结果，如果CMPXCHGQ比较是相等的，ret=1，否则ret=0

总结cas汇编做的事情：在保证单个内存地址值只针对单核心可达的情况下比较原始值是否相等，相等则替换返回替换成功；否则不替换返回替换失败。替换前先锁住内存总线保证多核心场景下不发生数据竞争，替换的动作是由单个指令实现保证了原子性。

再来分析Store操作

```assembly
// atomic_amd64.s
TEXT ·Store64(SB), NOSPLIT, $0-16
	MOVQ	ptr+0(FP), BX
	MOVQ	val+8(FP), AX
	XCHGQ	AX, 0(BX)
	RET
```

##### XCHGQ

同样Q在这里指的是8个字节，XCHGQ对应硬件指令是XCHG。做的操作是交换AX和BX寄存器的内容。同样也是单指令原子操作

#### happend-before与指令重排

`编译时优化 >> 运行时优化` 应该是大家都公认的准则，Go语言也不例外，其中一项优化就是：在保证逻辑正确的情况下调整代码前后顺序。这里就需要引入`happened-before语义`

##### happened-before

happened-before语义表示的是一个操作一定先于另一个操作发生，且具有传导性。举个实际的例子

```go
func main() {
    a := 1  // [1]
    b := 2	// [2]
    c := a + b // [3]
    println(c) // [4]
}
```

以上代码执行顺序可以是`[1,2,3,4]`也可以是`[2,1,3,4]`，因为不管是先执行1还是先执行2，都对结果没有影响。而3依赖1和2的执行结果，因此这里我们说`1和2  happened before 3`，也就是1和2一定在3之前执行；同样的`3 happened before 4`，3一定在4之前执行，因为3的结果一定得对4可见；由于该语义具有传导性，因此我们可以说`1和2 happened before 4`

对应到上面的go源码，在每一个重要步骤前都做了比较，保证了happened-before的一致性语义

##### 指令重排

不同于happened-before是编译器层面的逻辑，指令重排是汇编层面的优化，也可以叫做乱序执行。

乱序执行减少了cpu处于等待状态的可能，使得cpu在需要的时候尽可能满负荷运转。在单核cpu上，重排之后的指令遵循happened-before一致性语义，但是在多核cpu背景下，这一语义并不能保证，才有了上面汇编代码的`LOCK`指令的出现

#### atomc.Value操作总结

atomic.Value的`Store`和`Load`操作针对happened-before语义和指令重排可能造成的影响做了规避

### 使用atomic.Value如何避免数据竞争

了解了以上内容后，我们再来分析开头的两段代码

```go
func main() {
	conf := &Config{Name: "foo", Age: 10}	// [1]

	go func() {
		for {
			time.Sleep(time.Second)
			conf = &Config{Name: "bar", Age: 20}		// [3]
			println("new config:", conf.Name, conf.Age)	// [4]
		}
	}()	// [2]

	for {
		time.Sleep(time.Second)
		println("current config:", conf.Name, conf.Age)	// [5]
	}
}
```

这段代码

```
1 happened before 2
1 happened before 5
2 happened before 3
3 happened before 4
```

所以可能执行的顺序是`[1,2,3,4,5]`或者`[1,5,2,3,4]`或者`[1,2,5,3,4]`或者`[1,2,3,5,4]`，其中5相对于3的顺序是不确定的，因此5 可能发生在3之前/之后，也可能发生在3执行过程中。conf变量的读操作和写操作可能同时执行，发生了数据竞争。

再来看一下使用atomic.Value的代码

```go
func main() {
	conf := atomic.Value{}						// [1]
	conf.Store(&Config{Name: "foo", Age: 10})	// [2]

	go func() {
		for {
			time.Sleep(time.Second)
			conf.Store(&Config{Name: "bar", Age: 20})	// [4]
			t := conf.Load().(*Config)					// [5]
			println("new config:", t.Name, t.Age)		// [6]
		}
	}()	// [3]

	for {
		time.Sleep(time.Second)
		t := conf.Load().(*Config)					// [7]
		println("current config:", t.Name, t.Age)	// [8]
	}
}
```

```
1 happened before 2
2 happened before 3
2 happened before 7
3 happened before 4
4 happened before 5
5 happened before 6
7 happened before 8
```

执行顺序是`[1,2,3,7,8,4,5,6]`或者`[1,2,3,4,7,5,8,6]`等。顺序依然是不确定的，但是由于load和store操作前面我们分析过，是有一执行保证的，也就是读（load内部真实的读操作）和写（store内部真实的写操作）不会同时发生，所以atomic.Value操作不会出现数据竞争

## 总结

显而易见的结论是atomic.Value是有必要存在的。原因是在并发场景下，单纯指针的赋值操作可能会跟读操作并行执行，或者是写跟写并行，造成数据竞争。而atomic.Value提供的Store和Load方法，在语言层面（判断、简单乐观锁、执行顺序）和指令层面（CAS、锁总线、Store）都做了限制，使得读跟写、写跟写不会并行，从而避免了数据竞争

## 参考资料

[1]:  https://dave.cheney.net/2018/01/06/if-aligned-memory-writes-are-atomic-why-do-we-need-the-sync-atomic-package	"If aligned memory writes are atomic, why do we need the sync/atomic package?"
[2]:https://go.dev/blog/race-detector	"Introducing the Go Race Detector"
[3]:https://github.com/google/sanitizers/wiki/ThreadSanitizerAlgorithm	"ThreadSanitizer算法"
[4]:https://zhuanlan.zhihu.com/p/27503041	"事件和时间：Time, Clocks, and the Ordering of Events in a Distributed System 读后感"
[5]:https://www.cnblogs.com/orlion/p/14318553.html	"深入理解原子操作的本质"
[6]:https://zhuanlan.zhihu.com/p/597231892	"Golang 协程池 Ants 实现原理"
[7]:https://blog.betacat.io/post/golang-atomic-value-exploration/	"Go 语言标准库中 atomic.Value 的前世今生"
[8]:https://pkg.go.dev/sync/atomic	"go pkg atomic"
[9]:https://go.dev/ref/mem#atomic	"Atomic Values"
[10]:https://github.com/wweir/wweir.github.io/blob/src/content/post/%E6%8E%A2%E7%B4%A2%20golang%20%E4%B8%80%E8%87%B4%E6%80%A7%E5%8E%9F%E8%AF%AD.md	"探索golang一致性原语"

