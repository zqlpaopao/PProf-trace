# 1、runtime.GOARCH 平台架构

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {

	fmt.Println(runtime.GOARCH)
}

```



```
-> % go run test2.go
amd64
```



# 2、目标系统

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {

	fmt.Println(runtime.GOOS)
}

```



```
-> % go run test2.go
darwin

```



# 3、go的根目录

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {

	fmt.Println(runtime.GOROOT())
}

```



```
-> % go run test2.go
/usr/local/go
```



# 4、go的版本

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {

	fmt.Println(runtime.Version())
}

```



```
-> % go run test2.go
go1.14.3
```



# 5、逻辑CPU个数

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	fmt.Println(runtime.NumCPU())
}

```



```
-> % go run test2.go
4
```



# 6、GOMAXPROCS() 设置最大可用CPU个数

GOMAXPROCS设置可同时执行的最大CPU数，并返回先前的设置。 若 n < 1，它就不会更改当前设置。本地机器的逻辑CPU数可通过 NumCPU 查询。本函数在调度程序优化后会去掉。

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	//超过最大可用，返回最大可用
	fmt.Println(runtime.GOMAXPROCS(9))
}

```



```
-> % go run test2.go
4
```



# 7、设置CPU profile记录的速率

SetCPUProfileRate设置CPU profile记录的速率为平均每秒hz次。如果hz<=0，SetCPUProfileRate会关闭profile的记录。如果记录器在执行，该速率必须在关闭之后才能修改。

绝大多数使用者应使用runtime/pprof包或testing包的-test.cpuprofile选项而非直接使用SetCPUProfileRate。

```go
package main

import (
	"runtime"
)

func main() {
	//超过最大可用，返回最大可用
	runtime.SetCPUProfileRate(2)
}

```



# 8、CPU profile

```

```

![image-20201225151320568](runtime/image-20201225151320568.png)



# 9、GC() 执行一次GC

```go
package main

import (
	"runtime"
)

func main() {
	aa := make(map[string]string, 10000)
	aa["aa"] = "bb"
	runtime.GC()

}

```



```
-> % GODEBUG=gctrace=1 go run ./test2.go
gc 1 @0.023s 0%: 0.008+0.39+0.003 ms clock, 0.032+0.25/0.094/0.97+0.013 ms cpu, 4->4->0 MB, 5 MB goal, 4 P
gc 2 @0.039s 1%: 0.047+1.5+0.028 ms clock, 0.19+1.2/1.0/0.70+0.11 ms cpu, 4->4->0 MB, 5 MB goal, 4 P
gc 3 @0.074s 1%: 0.016+0.39+0.002 ms clock, 0.067+0.14/0.13/0.59+0.011 ms cpu, 4->4->0 MB, 5 MB goal, 4 P
# command-line-arguments
gc 1 @0.001s 22%: 0.026+3.4+0.014 ms clock, 0.10+2.0/2.2/3.7+0.057 ms cpu, 5->6->6 MB, 6 MB goal, 4 P
gc 2 @0.010s 11%: 0.002+3.2+0.002 ms clock, 0.011+0.061/1.9/2.9+0.010 ms cpu, 10->13->13 MB, 12 MB goal, 4 P
gc 1 @0.000s 2%: 0.004%   
```



# 10、逃避本次垃圾回收 runtime.SetFinalizer

如果需要在一个对象object被从内存中移除前执行一些特殊操作，比如写日志等，可以通过调用以下方式调用函数来实现

```
runtime.SetFinalizer(obj, func(obj *typeObj))
```

  在对象被GC进程选中并从内存中移除前，SetFinalizer都不会执行，即使程序正常结束或者发生错误

SetFinalizer将x的终止器设置为f。当垃圾收集器发现一个不能接触的（即引用计数为零，程序中不能再直接或间接访问该对象）具有终止器的块时，它会清理该关联（对象到终止器）并在独立go程调用f(x)。这使x再次可以接触，但没有了绑定的终止器。如果SetFinalizer没有被再次调用，下一次垃圾收集器将视x为不可接触的，并释放x。

<font color=red size=5x>第一次有runtime.SetFinalizer绑定的函数时候，会执行对应的方法，但是不会释放内存，第二次垃圾回收的时候才会进行回收</font>

demo

```
package main

import (
	"fmt"
	"runtime"
	"runtime/debug"
	"time"
)

func main() {
	var dic = new(map[string]string)
	runtime.SetFinalizer(dic, func(dic *map[string]string) {
		fmt.Println("内存回收")
	})
	debug.FreeOSMemory()
	time.Sleep(time.Second)
}
```



```
go run main.go
# command-line-arguments
gc 1 @0.009s 6%: 0.007+3.1+0.003 ms clock, 0.030+0.86/2.5/2.0+0.012 ms cpu, 4->5->4 MB, 5 MB goal, 4 P
内存回收
```

结果看出，执行`debug.FreeOSMemory()`进行了垃圾回收，然后执行的内存回收，第一次是垃圾回收，第二次是释放内存操作，<font color=green size=5x>可以当作析构函数使用</font>



# 11、内存申请和分配统计信息

```
func ReadMemStats(m *MemStats)
```

```
type MemStats struct {
    // 一般统计
    Alloc      uint64 // 已申请且仍在使用的字节数
    TotalAlloc uint64 // 已申请的总字节数（已释放的部分也算在内）
    Sys        uint64 // 从系统中获取的字节数（下面XxxSys之和）
    Lookups    uint64 // 指针查找的次数
    Mallocs    uint64 // 申请内存的次数
    Frees      uint64 // 释放内存的次数
    // 主分配堆统计
    HeapAlloc    uint64 // 已申请且仍在使用的字节数
    HeapSys      uint64 // 从系统中获取的字节数
    HeapIdle     uint64 // 闲置span中的字节数
    HeapInuse    uint64 // 非闲置span中的字节数
    HeapReleased uint64 // 释放到系统的字节数
    HeapObjects  uint64 // 已分配对象的总个数
    // L低层次、大小固定的结构体分配器统计，Inuse为正在使用的字节数，Sys为从系统获取的字节数
    StackInuse  uint64 // 引导程序的堆栈
    StackSys    uint64
    MSpanInuse  uint64 // mspan结构体
    MSpanSys    uint64
    MCacheInuse uint64 // mcache结构体
    MCacheSys   uint64
    BuckHashSys uint64 // profile桶散列表
    GCSys       uint64 // GC元数据
    OtherSys    uint64 // 其他系统申请
    // 垃圾收集器统计
    NextGC       uint64 // 会在HeapAlloc字段到达该值（字节数）时运行下次GC
    LastGC       uint64 // 上次运行的绝对时间（纳秒）
    PauseTotalNs uint64
    PauseNs      [256]uint64 // 近期GC暂停时间的循环缓冲，最近一次在[(NumGC+255)%256]
    NumGC        uint32
    EnableGC     bool
    DebugGC      bool
    // 每次申请的字节数的统计，61是C代码中的尺寸分级数
    BySize [61]struct {
        Size    uint32
        Mallocs uint64
        Frees   uint64
    }
}
```



demo

```
package main

import (
	"fmt"
	"runtime"
)

func main() {
	m := runtime.MemStats{}
	runtime.ReadMemStats(&m)
	fmt.Printf("%#v", m.Alloc)
	fmt.Println()
	fmt.Println(m.Alloc)
}

```



```
0x147e8{83944 83944 71387144 0 155 2 83944 66879488 66609152 270336 66543616 153 229376 229376 19448 32768 6944 16384 3106 
3436808 789214 4473924 0 0 [0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0] [0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 
0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0] 0 0 0 true false [{0 0 0} {8 5 0} {16 42 0} {32 15 0} {48 19 0} {64 4 0} {80 4 0} {96 5 0} {112 2 0} {128 1 0} {144 0 0} {160 0 0} {176 0 0} {192 0 0} {208 17 0} {224 0 0} {240 0 0} {256 0 0} {288 0 0} {320 0 0} {352 0 0} {384 14 0} {416 4 0} {448 0 0} {480 1 0} {512 0 0} {576 0 0} {640 2 0} {704 0 0} {768 0 0} {896 4 0} {1024 4 0} {1152 0 0} {1280 0 0} {1408 0 0} {1536 0 0} {1792 4 0} {2048 0 0} {2304 0 0} {2688 0 0} {3072 0 0} {3200 0 0} {3456 0 0} {4096 1 0} {4864 0 0} {5376 0 0} {6144 0 0} {6528 0 0} {6784 0 0} {6912 0 0} {8192 1 0} {9472 0 0} {9728 0 0} {10240 4 0} {10880 0 0} {12288 0 0} {13568 0 0} {14336 0 0} {16384 0 0} {18432 0 0} {19072 0 0}]}
```



# 12、MemProfileRecord用于描述某个调用栈序列申请和释放的活动对象等信息。

```
package main

import (
	"fmt"
	"runtime"
)

func main() {
	a := make(map[string]string, 10000)
	a = a
	mem := runtime.MemProfileRecord{
		AllocBytes:   10000,
		FreeBytes:    1000,
		AllocObjects: 1000,
		FreeObjects:  1000,
		Stack0:       [32]uintptr{},
	}
	fmt.Println(mem.InUseBytes())
}

```



```
 go run main.go
9000
```



# 13、打印堆栈信息 stack，建议使用debug

```go
package main

import (
	"fmt"
	log "github.com/sirupsen/logrus"
	"runtime"
	"runtime/debug"
	"time"
)

func main() {
	go func() {
		fmt.Println(12)
	}()
	log.Infof("stack %s", debug.Stack())
	fmt.Println("----------------------------")
	buf := make([]byte, 1<<16)
	runtime.Stack(buf, true)
	fmt.Println(string(buf))
	time.Sleep(3 * time.Second)
}

```



```go
go run main.go
12
INFO[0000] stack goroutine 1 [running]:
runtime/debug.Stack(0xbff2d25b00000000, 0x10ee688, 0x11a7a60)
        /usr/local/go/src/runtime/debug/stack.go:24 +0x9d
main.main()
        /Users/zhangsan/go/src/workspace/test1/main.go:14 +0x3e 
----------------------------
goroutine 1 [running]:
main.main()
        /Users/zhangsan/go/src/workspace/test1/main.go:17 +0x14e

```



# 14、打印行号和文件名 Caller

<font color=red size=5x>有性能问题</font>

```
package main

import (
	"fmt"
	"runtime"
)

func main() {
	call()
}

func call() {
	var calldepth = 1
	fmt.Println(runtime.Caller(calldepth))
}

```



```
go run main.go
17420239 /Users/zhangsan/go/src/workspace/test1/main.go 9 true
调用栈标别 文件名。行号 true表示获取成功
```



# 15、描述单条调用栈信息StackRecord

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	call()
}

func call() {
	call()
	st := runtime.StackRecord{}
	fmt.Println(st.Stack0)
	fmt.Println(st.Stack())
}

```



```
go run main.go
runtime: goroutine stack exceeds 1000000000-byte limit
runtime: sp=0xc0200e0348 stack=[0xc0200e0000, 0xc0400e0000]
fatal error: stack overflow

runtime stack:
runtime.throw(0x10ce232, 0xe)
        /usr/local/go/src/runtime/panic.go:1116 +0x72
runtime.newstack()
        /usr/local/go/src/runtime/stack.go:1034 +0x6ce
runtime.morestack()
        /usr/local/go/src/runtime/asm_amd64.s:449 +0x8f

goroutine 1 [running]:
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:12 +0x13e fp=0xc0200e0358 sp=0xc0200e0350 pc=0x109d1fe
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e03d0 sp=0xc0200e0358 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e0448 sp=0xc0200e03d0 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e04c0 sp=0xc0200e0448 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e0538 sp=0xc0200e04c0 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e05b0 sp=0xc0200e0538 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e0628 sp=0xc0200e05b0 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e06a0 sp=0xc0200e0628 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e0718 sp=0xc0200e06a0 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e0790 sp=0xc0200e0718 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e0808 sp=0xc0200e0790 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e0880 sp=0xc0200e0808 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e08f8 sp=0xc0200e0880 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e0970 sp=0xc0200e08f8 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e09e8 sp=0xc0200e0970 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e0a60 sp=0xc0200e09e8 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e0ad8 sp=0xc0200e0a60 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e0b50 sp=0xc0200e0ad8 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e0bc8 sp=0xc0200e0b50 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e0c40 sp=0xc0200e0bc8 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e0cb8 sp=0xc0200e0c40 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e0d30 sp=0xc0200e0cb8 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e0da8 sp=0xc0200e0d30 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e0e20 sp=0xc0200e0da8 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e0e98 sp=0xc0200e0e20 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e0f10 sp=0xc0200e0e98 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e0f88 sp=0xc0200e0f10 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1000 sp=0xc0200e0f88 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1078 sp=0xc0200e1000 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e10f0 sp=0xc0200e1078 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1168 sp=0xc0200e10f0 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e11e0 sp=0xc0200e1168 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1258 sp=0xc0200e11e0 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e12d0 sp=0xc0200e1258 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1348 sp=0xc0200e12d0 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e13c0 sp=0xc0200e1348 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1438 sp=0xc0200e13c0 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e14b0 sp=0xc0200e1438 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1528 sp=0xc0200e14b0 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e15a0 sp=0xc0200e1528 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1618 sp=0xc0200e15a0 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1690 sp=0xc0200e1618 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1708 sp=0xc0200e1690 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1780 sp=0xc0200e1708 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e17f8 sp=0xc0200e1780 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1870 sp=0xc0200e17f8 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e18e8 sp=0xc0200e1870 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1960 sp=0xc0200e18e8 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e19d8 sp=0xc0200e1960 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1a50 sp=0xc0200e19d8 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1ac8 sp=0xc0200e1a50 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1b40 sp=0xc0200e1ac8 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1bb8 sp=0xc0200e1b40 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1c30 sp=0xc0200e1bb8 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1ca8 sp=0xc0200e1c30 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1d20 sp=0xc0200e1ca8 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1d98 sp=0xc0200e1d20 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1e10 sp=0xc0200e1d98 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1e88 sp=0xc0200e1e10 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1f00 sp=0xc0200e1e88 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1f78 sp=0xc0200e1f00 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e1ff0 sp=0xc0200e1f78 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2068 sp=0xc0200e1ff0 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e20e0 sp=0xc0200e2068 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2158 sp=0xc0200e20e0 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e21d0 sp=0xc0200e2158 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2248 sp=0xc0200e21d0 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e22c0 sp=0xc0200e2248 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2338 sp=0xc0200e22c0 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e23b0 sp=0xc0200e2338 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2428 sp=0xc0200e23b0 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e24a0 sp=0xc0200e2428 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2518 sp=0xc0200e24a0 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2590 sp=0xc0200e2518 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2608 sp=0xc0200e2590 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2680 sp=0xc0200e2608 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e26f8 sp=0xc0200e2680 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2770 sp=0xc0200e26f8 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e27e8 sp=0xc0200e2770 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2860 sp=0xc0200e27e8 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e28d8 sp=0xc0200e2860 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2950 sp=0xc0200e28d8 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e29c8 sp=0xc0200e2950 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2a40 sp=0xc0200e29c8 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2ab8 sp=0xc0200e2a40 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2b30 sp=0xc0200e2ab8 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2ba8 sp=0xc0200e2b30 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2c20 sp=0xc0200e2ba8 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2c98 sp=0xc0200e2c20 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2d10 sp=0xc0200e2c98 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2d88 sp=0xc0200e2d10 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2e00 sp=0xc0200e2d88 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2e78 sp=0xc0200e2e00 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2ef0 sp=0xc0200e2e78 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2f68 sp=0xc0200e2ef0 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e2fe0 sp=0xc0200e2f68 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e3058 sp=0xc0200e2fe0 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e30d0 sp=0xc0200e3058 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e3148 sp=0xc0200e30d0 pc=0x109d0e6
main.call()
        /Users/zhangsan/go/src/workspace/test1/main.go:13 +0x26 fp=0xc0200e31c0 sp=0xc0200e3148 pc=0x109d0e6
...additional frames elided...
exit status 2
```



# ==16、返回方法的入口指针==

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	call()
}

func call() {
	var a int
	a = 1
	pc, _, _, _ := runtime.Caller(a)
	fmt.Println(pc)
	fmt.Println(runtime.FuncForPC(pc).Name())       //调用者函数名
	fmt.Println(runtime.FuncForPC(pc).Entry())      //调用者函数指针
	fmt.Println(runtime.FuncForPC(pc).FileLine(pc)) //调用者函数行号
}

```

```
go run main.go
17420415
main.main
17420384
/Users/zhangsan/go/src/workspace/test1/main.go 9
```



# ==17、返回协程数量==

```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

func main() {
	go call()
	time.Sleep(2 * time.Second)
}

func call() {
	fmt.Println(runtime.NumGoroutine())
}

```



```
-> % go run main.go
2
```



# ==18、终止调用它的协程Goexit==

```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

func main() {
	go call()
	time.Sleep(2 * time.Second)
	fmt.Println(1)
}

func call() {
	fmt.Println("执行了")
	runtime.Goexit()
	fmt.Println("没有执行")
}

```



```
 go run main.go
执行了
1
```



# ==19、让出处理器，让其他协程执行==

Gosched使当前go程放弃处理器，以让其它go程运行。它不会挂起当前go程，因此当前go程未来会恢复执行。

```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

func main() {
	go call()
	//time.Sleep(1 * time.Second)
	go calls()
	go calls()
	go calls()

	time.Sleep(3 * time.Second)
	fmt.Println(1)
}

func call() {
	fmt.Println("执行了")
	runtime.Gosched()
	fmt.Println("再次执行")
}

func calls() {
	fmt.Println("现在我执行")
}

```

起了4个协程 因为是4核，不然会有空闲的p立马把让出的额执行了

```
go run main.go
现在我执行
现在我执行
执行了---
现在我执行---让出了
再次执行---
1
```



# ==20、协程和线程绑定-解绑==

```
func LockOSThread()
```

将调用的go程绑定到它当前所在的操作系统线程。除非调用的go程退出或调用UnlockOSThread，否则它将总是在该线程中执行，而其它go程则不能进入该线程。

```
func UnlockOSThread
```

将调用的go程解除和它绑定的操作系统线程。若调用的go程未调用LockOSThread，UnlockOSThread不做操作。

```go
package main

// #include <pthread.h>
import "C"
import (
	"fmt"
	"runtime"
)

// xiaorui.cc
func main() {
	runtime.GOMAXPROCS(runtime.NumCPU())
	ch1 := make(chan bool)
	ch2 := make(chan bool)
	fmt.Println("main", C.pthread_self())
	go func() {
		runtime.LockOSThread()
		fmt.Println("locked", C.pthread_self())
		go func() {
			fmt.Println("locked child", C.pthread_self())
			ch1 <- true
		}()
		ch2 <- true
	}()
	<-ch1
	<-ch2
}

```

# 21、阻塞堆栈信息

