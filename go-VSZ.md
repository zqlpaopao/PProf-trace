

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



## memstats

```
DebugGC false
Frees 1084
HeapAlloc 1242816
HeapIdle 64372736
HeapInuse 2277376
HeapReleased 64307200
HeapObjects 9819

```



# 4、Go FAQ

```go
The Go memory allocator reserves a large region of virtual memory as an arena for allocations. 
This virtual memory is local to the specific Go process; the reservation does not deprive other processes of memory.
To find the amount of actual memory allocated to a Go process, use the Unix top command and consult the RES (Linux) or 
RSIZE (macOS) columns.
```



```
Go内存分配器保留了较大的虚拟内存区域作为分配空间。该虚拟内存在特定的Go进程本地；保留不会剥夺其他进程的内存。
要查找分配给Go进程的实际内存量，请使用Unix top命令并查询RES（Linux）或RSIZE（macOS）列。
```

这个 FAQ 是在 2012 年 10 月 [提交](https://github.com/golang/go/commit/2100947d4a25dcf875be1941d0e3a409ea85051e) 的，这么多年了也没有更进一步的说明，再翻了 issues 和 forum，一些关闭掉的 issue 都指向了 FAQ，这显然无法满足我的求知欲，因此我继续往下探索，看看里面到底都摆了些什么。



# 5、查看内存映射



```
vmmap --wide 83358
Process:         go [83358]
Path:            /usr/local/go/bin/go
Load Address:    0x1000000
Identifier:      go
Version:         ???
Code Type:       X86-64
Parent Process:  zsh [81343]

Date/Time:       2020-12-31 10:01:09.005 +0800
Launch Time:     2020-12-31 10:00:11.825 +0800
OS Version:      Mac OS X 10.13.6 (17G14033)
Report Version:  7
Analysis Tool:   /usr/bin/vmmap

Physical footprint:         10.2M
Physical footprint (peak):  12.2M
----

Virtual Memory Map of process 83358 (go)
Output report format:  2.4  -- 64-bit process
VM page size:  4096 bytes

==== Non-writable regions for process 83358
REGION TYPE                      START - END             [ VSIZE  RSDNT  DIRTY   SWAP] PRT/MAX SHRMOD PURGE    REGION DETAIL
__TEXT                 0000000001000000-0000000001a7c000 [ 10.5M  4528K     0K     0K] r-x/rwx SM=COW          /usr/local/go/bin/go
MALLOC metadata        0000000001bee000-0000000001bef000 [    4K     4K     4K     0K] r--/rwx SM=COW          DefaultMallocZone_0x1bee000 zone structure
 combined __LINKEDIT
shared memory          00007fffffe00000-00007fffffe01000 [    4K     4K     4K     0K] r--/r-- SM=SHM          
shared memory          00007fffffea9000-00007fffffeaa000 [    4K     4K     4K     0K] r-x/r-x SM=SHM          

==== Writable regions for process 83358
REGION TYPE                      START - END             [ VSIZE  RSDNT  DIRTY   SWAP] PRT/MAX SHRMOD PURGE    REGION DETAIL
__DATA                 0000000001a7c000-0000000001aca000 [  312K   152K   104K     0K] rw-/rw- SM=COW          /usr/local/go/bin/go
__DATA                 0000000001aca000-0000000001afc000 [  200K   136K   132K     0K] rw-/rw- SM=PRV          /usr/local/go/bin/go
__LINKEDIT             0000000001afc000-0000000001bec000 [  960K   208K     0K     0K] rw-/rwx SM=COW          /usr/local/go/bin/go
Kernel Alloc Once      0000000001bec000-0000000001bee000 [    8K     4K     4K     0K] rw-/rwx SM=PRV          
MALLOC metadata        0000000001bef000-0000000001bf0000 [    4K     4K     4K     0K] rw-/rwx SM=PRV          
MALLOC metadata        0000000001bf1000-0000000001bf5000 [   16K    16K    16K     0K] rw-/rwx SM=COW          
MALLOC metadata        0000000001bf7000-0000000001bfb000 [   16K    16K    16K     0K] rw-/rwx SM=COW          
MALLOC metadata        0000000001bfe000-0000000001bff000 [    4K     4K     4K     0K] rw-/rwx SM=PRV          
MALLOC_TINY            0000000001c00000-0000000001d00000 [ 1024K    60K    60K     0K] rw-/rwx SM=COW          DefaultMallocZone_0x1bee000
MALLOC metadata        0000000001d01000-0000000001d05000 [   16K    12K    12K     0K] rw-/rwx SM=PRV          
MALLOC metadata        0000000001d07000-0000000001d0b000 [   16K    12K    12K     0K] rw-/rwx SM=PRV          
VM_ALLOCATE            0000000001d0c000-0000000001d4c000 [  256K    44K    44K     0K] rw-/rwx SM=PRV          
VM_ALLOCATE            0000000001d4c000-0000000001d6c000 [  128K    64K    48K     0K] rw-/rwx SM=COW          
VM_ALLOCATE            0000000001dec000-0000000001ded000 [    4K     4K     4K     0K] rw-/rwx SM=COW          
VM_ALLOCATE            0000000001e6c000-0000000001f7c000 [ 1088K    12K    12K     0K] rw-/rwx SM=COW          
VM_ALLOCATE            0000000001f7c000-0000000001f8c000 [   64K    64K    64K     0K] rw-/rwx SM=PRV          
VM_ALLOCATE            0000000001f8c000-0000000001fec000 [  384K   176K   176K     0K] rw-/rwx SM=COW          
MALLOC_SMALL           0000000002000000-0000000002800000 [ 8192K    40K    40K     0K] rw-/rwx SM=COW          DefaultMallocZone_0x1bee000
VM_ALLOCATE            0000000002c06000-0000000002c07000 [    4K     4K     4K     0K] rw-/rwx SM=COW          
VM_ALLOCATE            0000000003000000-00000000032d1000 [ 2884K   488K   488K     0K] rw-/rwx SM=PRV          
MALLOC_TINY (empty)    0000000003300000-0000000003400000 [ 1024K    12K    12K     0K] rw-/rwx SM=COW          DefaultMallocZone_0x1bee000
__DATA                 00000000034ed000-0000000003525000 [  224K    88K    88K     0K] rw-/rwx SM=COW          /usr/lib/dyld
VM_ALLOCATE            0000000005570000-0000000005571000 [    4K     4K     4K     0K] rw-/rwx SM=COW          
VM_ALLOCATE            00000000176c0000-00000000176c1000 [    4K     4K     4K     0K] rw-/rwx SM=COW          
VM_ALLOCATE            0000000027540000-0000000029540000 [ 32.0M    20K    20K     0K] rw-/rwx SM=COW          
MALLOC_TINY            0000000029600000-0000000029700000 [ 1024K    12K    12K     0K] rw-/rwx SM=COW          DefaultMallocZone_0x1bee000
VM_ALLOCATE            0000000029700000-0000000029860000 [ 1408K   132K   132K     0K] rw-/rwx SM=COW          
MALLOC_TINY (empty)    0000000029900000-0000000029a00000 [ 1024K    12K    12K     0K] rw-/rwx SM=COW          DefaultMallocZone_0x1bee000
MALLOC_SMALL (empty)   000000002a000000-000000002b800000 [ 24.0M    56K    56K     0K] rw-/rwx SM=COW          DefaultMallocZone_0x1bee000
VM_ALLOCATE            000000c000000000-000000c004000000 [ 64.0M  10.5M  8456K     0K] rw-/rwx SM=PRV          
Stack                  0000700009f21000-0000700009fa3000 [  520K     8K     8K     0K] rw-/rwx SM=PRV          thread 1
Stack                  0000700009fa4000-000070000a026000 [  520K     8K     8K     0K] rw-/rwx SM=COW          thread 2
Stack                  000070000a027000-000070000a0a9000 [  520K     8K     8K     0K] rw-/rwx SM=COW          thread 3
Stack                  000070000a0aa000-000070000a12c000 [  520K     8K     8K     0K] rw-/rwx SM=COW          thread 4
Stack                  000070000a12d000-000070000a1af000 [  520K     8K     8K     0K] rw-/rwx SM=COW          thread 5
Stack                  000070000a1b0000-000070000a232000 [  520K     8K     8K     0K] rw-/rwx SM=COW          thread 6
Stack                  000070000a233000-000070000a2b5000 [  520K     8K     8K     0K] rw-/rwx SM=COW          thread 7
Stack                  000070000a2b6000-000070000a338000 [  520K     8K     8K     0K] rw-/rwx SM=PRV          thread 8
Stack                  000070000a339000-000070000a3bb000 [  520K     8K     8K     0K] rw-/rwx SM=COW          thread 9
Stack                  000070000a3bc000-000070000a43e000 [  520K     8K     8K     0K] rw-/rwx SM=COW          thread 10
Stack                  000070000a43f000-000070000a4c1000 [  520K     8K     8K     0K] rw-/rwx SM=COW          thread 11
Stack                  000070000a4c2000-000070000a544000 [  520K     8K     8K     0K] rw-/rwx SM=COW          thread 12
Stack                  00007ffeef400000-00007ffeefc00000 [ 8192K    36K    36K     0K] rw-/rwx SM=PRV          thread 0

SM=sharing mode:  
        COW=copy_on_write PRV=private NUL=empty ALI=aliased 
        SHM=shared ZER=zero_filled S/A=shared_alias
PURGE=purgeable mode:  
        V=volatile N=nonvolatile E=empty   otherwise is unpurgeable

==== Summary for process 83358
ReadOnly portion of Libraries: Total=306.4M resident=63.0M(21%) swapped_out_or_unallocated=243.5M(79%)
Writable regions: Total=152.4M written=11.4M(7%) resident=12.0M(8%) swapped_out=0K(0%) unallocated=140.4M(92%)

                                VIRTUAL RESIDENT    DIRTY  SWAPPED VOLATILE   NONVOL    EMPTY   REGION 
REGION TYPE                        SIZE     SIZE     SIZE     SIZE     SIZE     SIZE     SIZE    COUNT (non-coalesced) 
===========                     ======= ========    =====  ======= ========   ======    =====  ======= 
Kernel Alloc Once                    8K       4K       4K       0K       0K       0K       0K        2 
MALLOC guard page                   32K       0K       0K       0K       0K       0K       0K        8 
MALLOC metadata                     84K      76K      76K       0K       0K       0K       0K       10 
MALLOC_SMALL                      8192K      40K      40K       0K       0K       0K       0K        2         see MALLOC ZONE table below
MALLOC_SMALL (empty)              24.0M      56K      56K       0K       0K       0K       0K        2         see MALLOC ZONE table below
MALLOC_TINY                       2048K      72K      72K       0K       0K       0K       0K        3         see MALLOC ZONE table below
MALLOC_TINY (empty)               2048K      24K      24K       0K       0K       0K       0K        3         see MALLOC ZONE table below
STACK GUARD                       56.0M       0K       0K       0K       0K       0K       0K       14 
Stack                             14.1M     132K     132K       0K       0K       0K       0K       14 
VM_ALLOCATE                      687.1M    11.5M    9456K       0K       0K       0K       0K       21 
__DATA                            14.5M    9180K     688K       0K       0K       0K       0K      177 
__FONT_DATA                          4K       0K       0K       0K       0K       0K       0K        2 
__LINKEDIT                       194.1M    17.1M       0K       0K       0K       0K       0K        4 
__TEXT                           112.3M    45.8M       0K       0K       0K       0K       0K      181 
__UNICODE                          560K     440K       0K       0K       0K       0K       0K        2 
shared memory                        8K       8K       8K       0K       0K       0K       0K        3 
===========                     ======= ========    =====  ======= ========   ======    =====  ======= 
TOTAL                              1.1G    84.3M    10.3M       0K       0K       0K       0K      432 

                               VIRTUAL   RESIDENT      DIRTY    SWAPPED ALLOCATION      BYTES DIRTY+SWAP          REGION
MALLOC ZONE                       SIZE       SIZE       SIZE       SIZE      COUNT  ALLOCATED  FRAG SIZE  % FRAG   COUNT
===========                    =======  =========  =========  =========  =========  =========  =========  ======  ======
DefaultMallocZone_0x1bee000      36.0M       192K       192K         0K        645        66K       126K     66%       6
GFXMallocZone_0x1bfd000             0K         0K         0K         0K          0         0K         0K      0%       0
===========                    =======  =========  =========  =========  =========  =========  =========  ======  ======
TOTAL                            36.0M       192K       192K         0K        645        66K       126K     66%       6


```



## linux查看

 macOS 的 `vmmap` 命令去查看内存映射情况，这样就可以知道这个进程的内存映射情况，从输出分析来看，**这些关联共享库占用的空间并不大，导致 VSZ 过高的根本原因不在共享库和二进制文件上，但是并没有发现大量保留内存空间的行为，这是一个问题点**。

注：若是 Linux 系统，可使用 `cat /proc/PID/maps` 或 `cat /proc/PID/smaps` 查看。



# 6、查看系统调用









