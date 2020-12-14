https://segmentfault.com/a/1190000016412013

# PProf

- runtime/pprof：采集程序（非 Server）的运行数据进行分析
- net/http/pprof：采集 HTTP Server 的运行时数据进行分析

# 支持什么使用模式

- Report generation：报告生成
- Interactive terminal use：交互式终端使用
- Web interface：Web 界面

# 可以做什么

- CPU Profiling：CPU 分析，按照一定的频率采集所监听的应用程序 CPU（含寄存器）的使用情况，可确定应用程序在主动消耗 CPU 周期时花费时间的位置
- Memory Profiling：内存分析，在应用程序进行堆分配时记录堆栈跟踪，用于监视当前和历史内存使用情况，以及检查内存泄漏
- Block Profiling：阻塞分析，记录 goroutine 阻塞等待同步（包括定时器通道）的位置
- Mutex Profiling：互斥锁分析，报告互斥锁的竞争情况



# demo

```
package main

import (
	"log"
	"net/http"
	_ "net/http/pprof"
	"time"
)

func main() {
	go func() {
		for {
			log.Println("https://github.com/EDDYCJY")
			time.Sleep(2 * time.Second)
		}
	}()

	http.ListenAndServe("0.0.0.0:6060", nil)
}

```

远程访问

```
/debug/pprof/

Types of profiles available:
Count	Profile
2	allocs
0	block
0	cmdline
5	goroutine
2	heap
0	mutex
0	profile
8	threadcreate
0	trace
full goroutine stack dump
Profile Descriptions:

allocs: A sampling of all past memory allocations
block: Stack traces that led to blocking on synchronization primitives
cmdline: The command line invocation of the current program
goroutine: Stack traces of all current goroutines
heap: A sampling of memory allocations of live objects. You can specify the gc GET parameter to run GC before taking the heap sample.
mutex: Stack traces of holders of contended mutexes
profile: CPU profile. You can specify the duration in the seconds GET parameter. After you get the profile file, use the go tool pprof command to investigate the profile.
threadcreate: Stack traces that led to the creation of new OS threads
trace: A trace of execution of the current program. You can specify the duration in the seconds GET parameter. After you get the trace file, use the go tool trace command to investigate the trace.
```

- cpu（CPU Profiling）: `$HOST/debug/pprof/profile`，默认进行 30s 的 CPU Profiling，得到一个分析用的 profile 文件
- block（Block Profiling）：`$HOST/debug/pprof/block`，查看导致阻塞同步的堆栈跟踪
- goroutine：`$HOST/debug/pprof/goroutine`，查看当前所有运行的 goroutines 堆栈跟踪
- heap（Memory Profiling）: `$HOST/debug/pprof/heap`，查看活动对象的内存分配情况
- mutex（Mutex Profiling）：`$HOST/debug/pprof/mutex`，查看导致互斥锁的竞争持有者的堆栈跟踪
- threadcreate：`$HOST/debug/pprof/threadcreate`，查看创建新OS线程的堆栈跟踪

# 通过交互式终端使用

（1）go tool pprof [http://localhost](http://localhost/):6060/debug/pprof/profile?seconds=60















































































