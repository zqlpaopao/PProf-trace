

# 代码

```go
package main

import (
	"context"
	"fmt"
	"os"
	"runtime"
	"runtime/trace"
	"sync"
)

func main() {
	// 为了看协程抢占，这里设置了一个cpu 跑
	runtime.GOMAXPROCS(1)

	f, _ := os.Create("trace.dat")
	defer f.Close()

	_ = trace.Start(f)
	defer trace.Stop()

	ctx, task := trace.NewTask(context.Background(), "sumTask")
	defer task.End()

	var wg sync.WaitGroup
	wg.Add(10)
	for i := 0; i < 10; i++ {
		// 启动10个协程，只是做一个累加运算
		go func(region string) {
			defer wg.Done()

			// 标记region
			trace.WithRegion(ctx, region, func() {
				var sum, k int64
				for ; k < 1000000000; k++ {
					sum += k
				}
				fmt.Println(region, sum)
			})
		}(fmt.Sprintf("region_%02d", i))
	}
	wg.Wait()
}

```

启动10个协程，做sum运算

开启trace，将文件写入trace.dat

# 生成文件

trace.dat

```
go build trace_example.go
go tool trace trace.dat
```

```
go tool trace trace.dat
2020/12/23 18:38:57 Parsing trace...
2020/12/23 18:38:57 Splitting trace...
2020/12/23 18:38:57 Opening browser. Trace viewer is listening on http://127.0.0.1:54940
```

![image-20201223184103604](trace/image-20201223184103604.png)

# 3、查看web

- View trace：查看跟踪
- Goroutine analysis：Goroutine 分析
- Network blocking profile：网络阻塞概况
- Synchronization blocking profile：同步阻塞概况
- Syscall blocking profile：系统调用阻塞概况
- Scheduler latency profile：调度延迟概况
- User defined tasks：用户自定义任务
- User defined regions：用户自定义区域
- Minimum mutator utilization：最低 Mutator 利用率

![image-20201223203620262](trace.assets/image-20201223203620262.png)



## 一、调度延迟情况

在刚开始查看问题时，除非是很明显的现象，否则不应该一开始就陷入细节，因此我们一般先查看 “Scheduler latency profile”，我们能通过 Graph 看到整体的调用开销情况，如下：

![image-20201223205958055](trace.assets/image-20201223205958055.png)



## 二、协程情况

![image-20201223210051870](trace.assets/image-20201223210051870.png)

通过上图我们可以看到共有 3 个 goroutine，分别是 `runtime.main`、 `runtime/trace.Start.func1`、 `main.main.func1`，那么它都做了些什么事呢，接下来我们可以通过点击具体细项去观察。如下：



![image-20201223210130376](trace.assets/image-20201223210130376.png)



| Execution Time        | 执行时间     | 48ns |
| --------------------- | ------------ | ---- |
| Network Wait Time     | 网络等待时间 | 0ns  |
| Sync Block Time       | 同步阻塞时间 | 0ns  |
| Blocking Syscall Time | 调用阻塞时间 | 0ns  |
| Scheduler Wait Time   | 调度等待时间 | 14ns |
| GC Sweeping           | GC 清扫      | 0ns  |
| GC Pause              | GC 暂停      | 0ns  |

##  三、View trace

![image-20201223212443192](trace.assets/image-20201223212443192.png)



![image-20201223212606005](trace.assets/image-20201223212606005.png)

- Start：开始时间
- Wall Duration：持续时间
- Self Time：执行时间
- Start Stack Trace：开始时的堆栈信息
- End Stack Trace：结束时的堆栈信息
- Incoming flow：输入流
- Outgoing flow：输出流
- Preceding events：之前的事件
- Following events：之后的事件
- All connected：所有连接的事件

### View Events

我们可以通过点击 View Options-Flow events、Following events 等方式，查看我们应用运行中的事件流情况。如下：

![image-20201223212832061](trace.assets/image-20201223212832061.png)

通过分析图上的事件流，我们可得知这程序从 `G1 runtime.main` 开始运行，在运行时创建了 2 个 Goroutine，先是创建 `G18 runtime/trace.Start.func1`，然后再是 `G19 main.main.func1` 。而同时我们可以通过其 Goroutine Name 去了解它的调用类型，如：`runtime/trace.Start.func1` 就是程序中在 `main.main` 调用了 `runtime/trace.Start` 方法，然后该方法又利用协程创建了一个闭包 `func1` 去进行调用。



![img](trace.assets/640.jpeg)



## 协程信息

```
http://127.0.0.1:57886/usertask?type=sumTask&complete=true&latmin=2.511886431s&latmax=3.981071705s
```



![image-20201223184709784](trace/image-20201223184709784.png)





