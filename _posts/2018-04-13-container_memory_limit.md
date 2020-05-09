---
layout: post
title:  "为什么容器内的应用无法感知其内存限制"
date:   2018-04-13 17:12:13 +0800
categories: Docker
---

在实践中，我发现运行在容器内的应用其实无法获知容器实际的内存限制。


## 1. 现象

运行在容器内的应用其实无法获知容器实际的内存限制。

譬如，我有一台6G内存的测试服务器。直接host机器上通过``free``命令，查看到的总内存为5780MB：

```
root@allen-laptop:~# free -m
             total       used       free     shared    buffers     cached
Mem:          5780       5664        115          0         64        148
-/+ buffers/cache:       5452        328
Swap:         5951       3500       2451
```

在该服务器上启动一个docker容器，并限制该容器的可用内存为100M。进入容器内，再通过free查看内存：

```
# docker run -it -m 100M ubuntu /bin/bash

root@a9bc996507f8:/# free -m
              total        used        free      shared  buff/cache   available
Mem:           5780        1171         136           3        4472        4264
Swap:          5951          60        5891
```

发现容器内通过``free``命令，查看到的总内存仍然是5780MB。

那么，docker设置的100M内存限制没有生效吗？其实是起作用了，如果容器申请内存超出100M将被杀死。



## 2. 原因探究

### 2.1 /proc伪文件系统

[Linux Filesystem Hierarchy: Chapter 1. Linux Filesystem Hierarchy](http://www.tldp.org/LDP/Linux-Filesystem-Hierarchy/html/proc.html) 对``/proc``文件系统是这样描述的：

```
/proc is very special in that it is also a virtual filesystem.
It's sometimes referred to as a process information pseudo-file system. 
It doesn't contain 'real' files but runtime system information (e.g. system memory, devices mounted, hardware configuration, etc). 
For this reason it can be regarded as a control and information centre for the kernel.
In fact, quite a lot of system utilities are simply calls to files in this directory. 
For example, 'lsmod' is the same as 'cat /proc/modules' while 'lspci' is a synonym for 'cat /proc/pci'. 
By altering files located in this directory you can even read/change kernel parameters (sysctl) while the system is running.
```

意思是，``/proc``实际上是一种``伪文件系统`` （pseudo-filesystem），数据是从操作系统内核实时获取的运行时信息。一些Linux命令实质上是直接从/proc下的文件读取相关信息。



### 2.2 free命令的实现

[FREE(1) User Commands](https://www.linux.org/docs/man1/free.html) 对``free``命令的说明：

```
free \- Display amount of free and used memory in the system
... ...
displays the total amount of free and used physical and swap memory in the
system, as well as the buffers and caches used by the kernel. The
information is gathered by parsing /proc/meminfo. 
```

意思是，**``free``命令所显示的内存信息实际上是从``/proc/meminfo``文件读取出来的。**

查看一下host机器的``/proc/meminfo``文件，里面与类存相关的信息如下：

```
root@allen-laptop:~# cat /proc/meminfo
MemTotal:        5919112 kB
MemFree:          380152 kB
MemAvailable:    4431280 kB
Buffers:          255192 kB
Cached:          3907228 kB
SwapCached:         4708 kB
... ...
```

可见，``free``命令实质上只对``/proc/meminfo``文件的信息进行简单处理并输出。

我们可以大胆猜测：操作系统获取内存状态的API实现原理与``free``命令类似，而JVM调用操作系统接口获取内存状态必定也与此类似。


再进入容器内，查看``/proc/meminfo``文件：
```
root@ubuntu-1:/# cat /proc/meminfo 
MemTotal:        5919112 kB
MemFree:          361372 kB
MemAvailable:    4417396 kB
Buffers:          255660 kB
Cached:          3911608 kB
SwapCached:         4708 kB
... ...
```

容器内部的``/proc/meminfo``文件中MemTotal与host机器上是相同的。所以，容器内执行``free``命令查看到的内存总量当然就跟host机器上的一样。



### 2.3 容器资源配额的Linux内核实现

#### 2.3.1 容器资源配额 & Linux CGroup

对容器的内存限制是通过Linux的CGroup机制实现。

docker容器的内存限制记录于host机器的``/sys/fs/cgroup/memory/docker/[容器ID]/memory.limit_in_bytes``目录下，如：

```
/# cat /sys/fs/cgroup/memory/docker/a9bc996507f83e47bd66b0f49392f7081b532b1dafab8b66b6c6454090915418/memory.limit_in_bytes 
104857600
```
而Kubernetes Pod的内存限制是记录于host机器的``/sys/fs/cgroup/memory/kubepods/pod+[pod ID]/memory.limit_in_bytes``
目录下，如：
```
# cat /sys/fs/cgroup/memory/kubepods/pod5ef448a2-c296-11e7-a6b7-206a8a801923/memory.limit_in_bytes 
104857600
```

host机器上的容器的memory.limit_in_bytes文件，将挂载在该容器的``/sys/fs/cgroup/memory/memory.limit_in_bytes`` ，可以进入容器内查看该文件：

```
root@a9bc996507f8:/# cat /sys/fs/cgroup/memory/memory.limit_in_bytes
104857600
```



#### 2.3.2 Linux Namespace并未对/proc/meminfo进行容器化改造

根据
[Linux Namespace说明手册](http://man7.org/linux/man-pages/man7/namespaces.7.html) 和[Docker背后的内核知识——Namespace资源隔离](http://www.infoq.com/cn/articles/docker-kernel-knowledge-namespace-resource-isolation)， Linux目前提供如下的Namespace隔离机制：

| Namespace | 隔离内容          |
| --------- | ------------- |
| IPC       | 信号量、消息队列和共享内存 |
| UTS       | 主机名与域名        |
| PID       | 进程编号          |
| Network   | 网络设备、网络栈、端口等等 |
| Mount     | 挂载点（文件系统）     |
| User      | 用户和用户组        |


对于上表中的内容，host机与容器中是隔离的。如进程ID：由于共享内核的原因，容器中进程实质上就是host系统上的一个进程；由于namespace PID隔离的原因，但同一个进程在host系统与在容器中的进程ID是不同的。

但上表之外的内容，如``/proc/meminfo``和``/proc/vmstat``等尚未实现namespace隔离。



这就是问题产生的根源：

* 容器与宿主机是共享内核的，Linux下``/proc``文件系统尚未实现完全的容器化namespace隔离，``/proc/meminfo`` 是来自host机。
* 操作系统通过CGroup来限制容器的内存，但获取内存的途径，如``free``命令，并不会去检查CGroup信息。





## 3. 展望 

针对本文所描述的问题，业界已经在研发相关解决方案。例如，
[JDK 9将针对容器调整内存限制](https://www.infoq.com/news/2017/02/java-memory-limit-container) 。我们可以去围观一下[OpenJDK hotspot针对容器正在进行的调整](http://hg.openjdk.java.net/jdk9/jdk9/hotspot/rev/5f1d1df0ea49) 。

但在JDK改造完成之前，对于已经运行在容器中的Java应用，我们也有相关方案避免出现严重问题。
