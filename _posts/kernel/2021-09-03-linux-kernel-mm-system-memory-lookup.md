---
title: linux kernel mm(1):查看系统内存信息
date: 2021-09-03 0:33:50
tags: linux, kernel
layout: post
---

- 通过熟悉Linux系统中常用的内存监测工具来感性地认识和了解内存管理

- 查看系统内存信息



### top命令

```
top - 20:39:01 up  1:58,  1 user,  load average: 0.20, 0.10, 0.15
Tasks: 293 total,   1 running, 197 sleeping,   0 stopped,   0 zombie
%Cpu(s):  1.1 us,  2.1 sy,  0.0 ni, 96.8 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  3049524 total,  1670272 free,   591500 used,   787752 buff/cache
KiB Swap:  2097148 total,  1781756 free,   315392 used.  2298300 avail Mem 

   PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                      
  7844 rlk       20   0 1419840 138896  21212 S  22.8  4.6  11:22.55 qemu-system-arm                                                              
  1174 root      20   0  421264  76048   7660 S  10.9  2.5   2:38.22 Xorg                                                                         
  2799 rlk       20   0  691372  30000  19980 S   4.3  1.0   0:25.34 mate-terminal                                                                
  2900 rlk       20   0  727680  11716   8312 S   2.3  0.4   0:14.93 python3                                                                      
  3022 rlk        9 -11  642460   8172   6620 S   1.7  0.3   1:25.35 pulseaudio                                                                   
  2777 rlk       20   0  197540  11260   8020 S   1.3  0.4   0:09.88 xfwm4                                                                        
 10478 rlk       20   0   45964   4128   3424 R   0.7  0.1   0:00.27 top                                                                          
  1590 root      20   0  178716   4456   3636 S   0.3  0.1   0:10.13 vmtoolsd                                                                     
  1744 root      20   0  508352   8932   7168 S   0.3  0.3   0:07.04 ManagementAgent                                                              
  1756 kernoops  20   0   56940    108      0 S   0.3  0.0   0:00.16 kerneloops                                                                   
  2810 rlk       20   0  814240  10280   8944 S   0.3  0.3   0:02.99 ukui-window-swi                                                              
  2956 rlk       20   0  192148   6844   5568 S   0.3  0.2   0:10.78 vmtoolsd                                                                     
  9058 root      20   0       0      0      0 I   0.3  0.0   0:00.36 kworker/u256:0      
```

​    VIRT：进程占用的虚拟内存

​    RES：进程占用的物理内存

​    SHR：进程使用的共享内存

   %MEM：进程使用的物理内存和总内存的百分





### free命令：

命令分类：

free   用KB为单位展示数据

free -m   用MB为单位展示数据

free -h   用GB为单位展示数据

```
rlk@ubuntu:lab2_get_system_mem_info$ free
              total        used        free      shared  buff/cache   available
Mem:        3049524      590928     1670584        2424      788012     2298856
Swap:       2097148      315392     1781756
```

total : 总计物理内存的大小

used : 已使用内存的大小

free : 可用内存的大小

shared : 多个进程共享的内存总额

buff/cache : 磁盘缓存大小

available : 可用内存大小



### cat /proc/meminfo 命令：

```
MemTotal:        3049524 kB
MemFree:         1670268 kB
MemAvailable:    2298596 kB
Buffers:           32800 kB
Cached:           711556 kB
SwapCached:        38936 kB
Active:           497656 kB
Inactive:         581084 kB
Active(anon):     194536 kB
Inactive(anon):   142280 kB
Active(file):     303120 kB
Inactive(file):   438804 kB
Unevictable:          96 kB
Mlocked:              96 kB
SwapTotal:       2097148 kB
SwapFree:        1782012 kB
Dirty:                 4 kB
Writeback:             0 kB
AnonPages:        322140 kB
Mapped:           106268 kB
Shmem:              2424 kB
Slab:             102924 kB
SReclaimable:      43704 kB
SUnreclaim:        59220 kB
KernelStack:        9984 kB
PageTables:        27808 kB
NFS_Unstable:          0 kB
Bounce:                0 kB
WritebackTmp:          0 kB
CommitLimit:     3621908 kB
Committed_AS:    3262348 kB
VmallocTotal:   34359738367 kB
VmallocUsed:           0 kB
VmallocChunk:          0 kB
HardwareCorrupted:     0 kB
AnonHugePages:     65536 kB
ShmemHugePages:        0 kB
ShmemPmdMapped:        0 kB
CmaTotal:              0 kB
CmaFree:               0 kB
HugePages_Total:       0
HugePages_Free:        0
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:       2048 kB
DirectMap4k:      124800 kB
DirectMap2M:     1972224 kB
DirectMap1G:     1048576 kB
```

