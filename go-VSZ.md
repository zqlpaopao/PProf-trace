

https://segmentfault.com/a/1190000037683112

# 1、top

```
PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
25356 golanger  20   0  585m  37m 5884 S 21.1  0.1  29977:05 bin_wx_health
 5193 root      20   0  103m 1440  732 R 19.4  0.0   0:00.59 netstat
24976 golanger  20   0  550m  53m 6972 S 18.1  0.1   1551:08 bin_tag_index
```



# 2、VSZ进程虚拟内存

> **进程虚拟内存，它包含了你的代码、数据、堆、栈段和共享库**。
>
> 虚拟内存作为内存保护的工具，能够保证进程之间的内存空间独立，不受其他进程的影响，因此每一个进程的 VSZ 大小都不一样，互不影响。
>
> 虚拟内存的存在，系统给各进程分配的内存之和是可以大于实际可用的物理内存的，因此你也会发现你进程的物理内存总是比虚拟内存低的多的多。



# 3、程序分析

```go
package main

import "github.com/gin-gonic/gin"

func main() {
	r := gin.Default()
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.Run(":8001")
}

```



```
ps aux 70093  
USER       PID  %CPU %MEM      VSZ    RSS   TT  STAT STARTED      TIME COMMAND
zhangsan 70093   0.0  0.2  5016948  20164 s006  S+    5:35PM   0:01.23 go run main.go
```

vsz. 5016948k == 5G左右



## 查看pprof和memstats

```
go tool pprof -inuse_space http://127.0.0.1:6060/debug/pprof/heap
Fetching profile over HTTP from http://127.0.0.1:6060/debug/pprof/heap
Saved profile in /Users/zhangsan/pprof/pprof.alloc_objects.alloc_space.inuse_objects.inuse_space.009.pb.gz
Type: inuse_space
Time: Dec 30, 2020 at 5:55pm (CST)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top
Showing nodes accounting for 517.33kB, 100% of 517.33kB total
      flat  flat%   sum%        cum   cum%
  517.33kB   100%   100%   517.33kB   100%  regexp/syntax.(*compiler).inst (inline)
         0     0%   100%   517.33kB   100%  github.com/go-playground/validator/v10.init
         0     0%   100%   517.33kB   100%  regexp.Compile (inline)
         0     0%   100%   517.33kB   100%  regexp.MustCompile
         0     0%   100%   517.33kB   100%  regexp.compile
         0     0%   100%   517.33kB   100%  regexp/syntax.(*compiler).compile
         0     0%   100%   517.33kB   100%  regexp/syntax.(*compiler).rune
         0     0%   100%   517.33kB   100%  regexp/syntax.Compile
         0     0%   100%   517.33kB   100%  runtime.doInit
         0     0%   100%   517.33kB   100%  runtime.main
(pprof) 
```







