---
layout: post
title:  "一次容器内存事故复盘"
date:   2018-04-13 17:12:13 +0800
categories: Kubernetes
---

公司项目组同事反馈，容器平台中某个容器应用有时会出现“容器没挂，但里面Java进程挂掉”的情况。
这个容器应用是一位已离职的同事所创建，我不了解情况，为此还专门花时间研究了一番。

## 1.  问题复盘

依据反馈的情况，容器存活（livenessProbe为成功状态），但处于未准备状态（readinessProbe为失败状态）。

首先，ssh到问题容器所在的机器上看内核日志(``/var/log/kern.log``)，发现进程挂掉的原因是OOM kill：
```
166233 Apr 18 00:37:34   kernel: [14049899.298694] exe invoked oom-killer: gfp_mask=0xd0, order=0, oom_score_adj=0
166234 Apr 18 00:37:34   kernel: [14049899.298698] exe cpuset=de9d91dd8ee42dfd5e7465045afacea550ffb67315755584adbf8ff515eb0809 mems_allowed=0
166235 Apr 18 00:37:34   kernel: [14049899.298701] CPU: 3 PID: 12983 Comm: exe Not tainted 3.13.0-135-generic #184-Ubuntu
166236 Apr 18 00:37:34   kernel: [14049899.298703] Hardware name: Xen HVM domU, BIOS 4.2.amazon 08/24/2006
166237 Apr 18 00:37:34   kernel: [14049899.298705]  0000000000000000 ffff8800ead57c50 ffffffff8172d959 ffff880088e89800
166238 Apr 18 00:37:34   kernel: [14049899.298709]  ffff880115dd1c00 ffff8800ead57cd8 ffffffff81727ef8 ffff880115dd1c00
166239 Apr 18 00:37:34   kernel: [14049899.298712]  ffff8800ead57c90 0000000000000046 ffff8800ead57ca8 ffffffff81156937
166240 Apr 18 00:37:34   kernel: [14049899.298715] Call Trace:
166241 Apr 18 00:37:34   kernel: [14049899.298723]  [<ffffffff8172d959>] dump_stack+0x64/0x82
166242 Apr 18 00:37:34   kernel: [14049899.298726]  [<ffffffff81727ef8>] dump_header+0x7f/0x1f1
166243 Apr 18 00:37:34   kernel: [14049899.298731]  [<ffffffff81156937>] ? find_lock_task_mm+0x47/0xa0
166244 Apr 18 00:37:34   kernel: [14049899.298734]  [<ffffffff81156db1>] oom_kill_process+0x201/0x360
166245 Apr 18 00:37:34   kernel: [14049899.298739]  [<ffffffff812de125>] ? security_capable_noaudit+0x15/0x20
166246 Apr 18 00:37:34   kernel: [14049899.298743]  [<ffffffff811b921c>] mem_cgroup_oom_synchronize+0x51c/0x560
166247 Apr 18 00:37:34   kernel: [14049899.298745]  [<ffffffff811b8750>] ? mem_cgroup_charge_common+0xa0/0xa0
166248 Apr 18 00:37:34   kernel: [14049899.298748]  [<ffffffff81157594>] pagefault_out_of_memory+0x14/0x80
166249 Apr 18 00:37:34   kernel: [14049899.298753]  [<ffffffff81726525>] mm_fault_error+0x67/0x140
166250 Apr 18 00:37:34   kernel: [14049899.298756]  [<ffffffff81739de2>] __do_page_fault+0x4a2/0x560
166251 Apr 18 00:37:34   kernel: [14049899.298759]  [<ffffffff81185a61>] ? do_mmap_pgoff+0x381/0x400
166252 Apr 18 00:37:34   kernel: [14049899.298764]  [<ffffffff81170349>] ? vm_mmap_pgoff+0x99/0xc0
166253 Apr 18 00:37:34   kernel: [14049899.298767]  [<ffffffff81739eba>] do_page_fault+0x1a/0x70
166254 Apr 18 00:37:34   kernel: [14049899.298771]  [<ffffffff817361e8>] page_fault+0x28/0x30
166255 Apr 18 00:37:34   kernel: [14049899.298773] Task in /docker/de9d91dd8ee42dfd5e7465045afacea550ffb67315755584adbf8ff515eb0809 killed as a result of limit of /docker/de9d91dd8ee42dfd5e7465045aface       a550ffb67315755584adbf8ff515eb0809
166256 Apr 18 00:37:34   kernel: [14049899.298776] memory: usage 3145728kB, limit 3145728kB, failcnt 22832
166257 Apr 18 00:37:34   kernel: [14049899.298777] memory+swap: usage 0kB, limit 18014398509481983kB, failcnt 0
166258 Apr 18 00:37:34   kernel: [14049899.298779] kmem: usage 0kB, limit 18014398509481983kB, failcnt 0
166259 Apr 18 00:37:34   kernel: [14049899.298780] Memory cgroup stats for /docker/de9d91dd8ee42dfd5e7465045afacea550ffb67315755584adbf8ff515eb0809: cache:188KB rss:3145540KB rss_huge:595968KB mapped_f       ile:8KB writeback:0KB inactive_anon:0KB active_anon:3145688KB inactive_file:4KB active_file:8KB unevictable:0KB
166260 Apr 18 00:37:34   kernel: [14049899.298790] [ pid ]   uid  tgid total_vm      rss nr_ptes swapents oom_score_adj name
166261 Apr 18 00:37:34   kernel: [14049899.298828] [28234]     0 28234     1109       48       7        0           666 sh
166262 Apr 18 00:37:34   kernel: [14049899.298831] [28238]     0 28238     4892      106      14        0           666 bash
166263 Apr 18 00:37:34   kernel: [14049899.298833] [28282]     0 28282  1591618   333513     808        0           666 java
166264 Apr 18 00:37:34   kernel: [14049899.298836] [28297]     0 28297    15338      234      34        0           666 sshd
166265 Apr 18 00:37:34   kernel: [14049899.298839] [12417]     0 12417   616805   448697    1011        0           666 python
166266 Apr 18 00:37:34   kernel: [14049899.298841] [12983]     0 12983    13399      230      27        0             0 exe
166267 Apr 18 00:37:34   kernel: [14049899.298843] Memory cgroup out of memory: Kill process 12417 (python) score 1237 or sacrifice child
166268 Apr 18 00:37:34   kernel: [14049899.310153] Killed process 12417 (python) total-vm:2467220kB, anon-rss:1794532kB, file-rss:256kB

```
确定容器中的jvm是被内核OOM kill之后。接下来需要搞清楚的问题有两个：
* 为什么Java进程会被OOM kill ？ 
* Java进程被干掉之后，kubernetes为什么没有自动重启这个pod ？ 


## 2. 为什么Java会被OOM kill ？

### 2.1 Kubernetes的内存配置

Kubernetes是通过Pod配置中的memory request和memory limit，来配置容器的内存控制策略。
* memory request（内存最低保障）：表示容器正常运行必须要提供的内存，Kubernetes将 **确保** 容器获得该内存量。例如，某容器设置memory request为1G，那么Kubernetes将为其预留1G内存。
* memory limit（内存最高上限）：表示容器所能使用的内存量的上限。

容器的memory request和memory limit配置会产生如下影响：
* 影响Kubernetes的容器调度。宿主机上运行的容器的memory request之和，一定小于宿主机的可供分配的内存量。举个例子，某节点N可分配内存是600M，上面已经运行了一个memory request为500M的容器A，现在有一个memory request是200M新容器B需要调度。那么即便当前容器A实际只消耗了1M的内存，容器B也不能调度到节点N上面，因为两个容器的memory request之和超过了节点可分配内存，Kubernetes调度策略必须要满足容器的memory request。
* 影响Kubernetes杀死容器的顺序。 当节点出现内存竞争时，  Kubernetes将  （1）优先杀死内存用量达到其memory limit 的容器； （2）在1的基础上，若无法解决内存竞争，则继续杀死内存用量超过其memory request的容器。

### 2.2.  JVM内存参数配置

在容器环境下，采用JVM默认的XMX配置常常会出现问题。

根据 [Oracle JVM Garbage Collector Ergonomics](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gc-ergonomics.html) ，若JVM启动时没有配置堆内存参数，那么在默认情况下JVM设置xmx为宿主机的1/4，xms为宿主机的1/64。

例如，下面这台服务器内存总量为2GB，所以JVM默认设置MaxHeapSize（xmx）为500M（1/4），设置InitialHeapSize（xms）为32M（1/64）：

```
$ free -m
             total       used       free     shared    buffers     cached
Mem:          2001       1788        213          1         28       1343
-/+ buffers/cache:        416       1585
Swap:         2047        612       1435

$ java -XX:+PrintFlagsFinal -version | grep HeapSize
    uintx ErgoHeapSizeLimit                         = 0                                   {product}
    uintx HeapSizePerGCThread                       = 87241520                            {product}
    uintx InitialHeapSize                          := 33554432                            {product}
    uintx LargePageHeapSizeThreshold                = 134217728                           {product}
    uintx MaxHeapSize                              := 526385152                           {product}
java version "1.8.0_66"
Java(TM) SE Runtime Environment (build 1.8.0_66-b17)
Java HotSpot(TM) 64-Bit Server VM (build 25.66-b17, mixed mode)
```

根据《容器内存机制探讨：为什么容器内的应用无法感知其内存限制》的结论：容器内应用感知到的内存与host机内存一致。

举个例子：某台机器内存8GB，我们在上面运行一个Java容器，容器内存上限设置为1GB。假设我们Java启动命令忘记设置XMX，那么JVM默认会将XMX设置为2GB。那么当Java申请内存超过1GB时，容器将会重启。


### 2.3  当JVM XMX配置遇上Kubernetes内存限制

根据JVM xmx、容器内存需求（memory request）、容器内存上限（memory limit）配置的不同情况，JVM与容器会擦出不同的火花。具体情况如下：

#### 2.3.1 xmx < memReq < memLimit

在这种情况下，当Java申请堆内存超出xmx时，OS内核OOM-killer就会干掉Java进程，但由于容器内存实际仍未达到memory request，因此容器不会被干掉。
因此，**有出现“容器没挂，但里面Java进程挂掉“的可能**。

#### 2.3.2 memReq < xmx < memLimit

这种情况下，当Java申请内存超出容器memory request、但小于xmx，若宿主机的内存有富余，则容器和Java均正常运行；若所在宿主机内存无富余，则容器和Java进程将同时被OS内核OOM kill。**当Java申请内存超出xmx时，若宿主机内存仍有富余，则容器不会挂掉，但Java进程会被OS内核OOM kill。这种情况下，同样有出现“容器没挂，但里面Java进程挂掉”的可能。**
回到本文开头所说的事故正是这种情况：memory request 2G、xmx3000M、memory limit 4296MB， 容器没挂，但Java挂掉。


#### 2.3.3 memReq < memLimit < xmx

这种情况下，当Java申请内存超出容器memory request但小于memory limit：若宿主机内存富余，则容器和Java均正常运行；若宿主机内存无富余，则容器将被杀死并调度到其他机器。当Java申请内存超出memory limit时，容器将被杀死并调度到其他机器。
这种情况下，不存在“容器没挂，但里面Java进程挂掉”的情况。


## 3. 为什么Kubernetes没有重启Java挂掉的容器？

我们配置容器restartPolicy为always，因此当容器内主进程挂掉后，Kubernetes会因为容器CrashLoopBackOff将其重启。

但目前我们遇到的问题是“Java挂掉，但容器没挂”，那么可推测原因有两个：

* 容器内除了Java外，一定还有其他在前台执行的进程。否则Java一挂，容器内没有前台执行的进程，容器必定立即重启。
* 容器没有设置livenessProbe，或者livenessProbe没有检测出Java挂掉。

带着这样的疑问，进入容器内。首先看一下进程：
```
# ps -ef                    
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 Apr19 ?        00:00:00 /bin/sh -c bash /paas-bootstrap.
root         6     1  0 Apr19 ?        00:00:00 bash /paas-bootstrap.sh
root        49     1  1 Apr19 ?        00:32:41 java -Djava.awt.headless=true -D
root        64     6  0 Apr19 ?        00:00:02 /usr/sbin/sshd -D
... ...
```
发现除了Java进程外，还有一个ssh服务器（sshd）。

再看一下启动脚本（/paas-bootstrap.sh）：
```
nohup java $JAVA_OPTS $JAVA_MEM_OPTS $JAVA_GC_OPTS $JAVA_DEBUG_OPTS $JAVA_JMX_OPTS -classpath $CONF_DIR:$LIB_JARS com.alibaba.dubbo.container.Main $SPRINGBOOT_OPTS >/dev/null 2>$ERROR_OUT_FILE &
/usr/sbin/sshd -D
```

可看出，Java是以nohup方式后台启动的，而且最后还以前台运行的方式启动了一个sshd进程。

再检查容器配置，看看有没有配置livenessProbe：
```
# kubectl --namespace=retargeting-01 get po ec-core-services-61954117-wn9hj -o yaml | grep liveness
#
```
确定没有配置livenessProbe。

至此，我们明确得出结论：

* 在这个Pod中，前台执行的进程是sshd进程，而不是Java进程，所以Java进程挂掉并不会导致容器CrashLoopBackOff。
* 这个Pod没有设置检查Java进程的livenessProbe，所以Kubernetes根本检测不出Java进程已经挂掉。


## 4.  解决方案

### 4.1  配置livenessProbe，让k8s检测Java进程状态

增加一个检查Java进程的``livenessProbe.sh``：

```
#!/bin/bash
PIDS=`jps | grep $APP_NAME | awk '{print $1}'`

if [ -z "$PIDS" ]; then
    echo "not ok"
    exit 1
else
    echo 'ok'
    exit 0
fi
```

修改Deployment配置：

````
{
    "spec":{
        "template":{
            "spec":{
                "containers":[
                    {
                    ... ...
                        "readinessProbe":{
                            "exec":{
                                "command":[
                                    "bash",
                                    "-c",
                                    "/dianyi/app/xxxx/xxxx-java/bin/readinessProbe.sh"
                                ]
                            },
                            "initialDelaySeconds":3
                        },
                        "livenessProbe":{
                            "exec":{
                                "command":[
                                    "bash",
                                    "-c",
                                    "/dianyi/app/xxxx/xxxx-java/bin/livenessProbe.sh"
                                ]
                            },
                            "initialDelaySeconds":15
                        }
                        ... ...
                    }
                ]
            }
        }
    }
}
````


### 4.2 根据容器cgroup限制来配置XMX，而不是根据宿主机内存总量。

根据上述分析可知，当Java因超XMX被OOM kill时，实际消耗内存并未达到容器的memory limit。
这说明Java XMX配置可能不合理，我们可以增加XMX配置。这里有两种方案：

* 手动配置XMX。这也是我们目前的配置方式。可以将XMX配置得更加接近memory limit。
* 自动配置方式。即Java启动前，通过脚本自动计算一个合理的XMX值。

手动方式简单，下面我们重点说明自动配置方式。

新版JDK能够根据CGroup内存限制自动设置XMX，即使我们不打算升级JDK，但[OpenJDK针对容器化改进方案](http://hg.openjdk.java.net/jdk9/jdk9/hotspot/rev/5f1d1df0ea49)  也是很有参考价值的。其原理很简单：

```
Java启动前，先从/sys/fs/cgroup/memory/memory.limit_in_bytes 读取当前系统/容器的内存cgroup限制，从/proc/meminfo获取宿主机的内存总量；
将两者中较小的一个当作可用内存，然后根据这个可用内存计算XMX默认值。
```

同理，我们可以写个脚本，在Java应用启动之前，根据CGroup和``/proc/meminfo`` 计算一个合理的XMX值。

可以参考如下代码：

```
#!/bin/bash

# Return reasonable JVM options to run inside a Docker container where memory and
# CPU can be limited with cgroups.
# https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html
#
# The script can be used in a custom CMD or ENTRYPOINT
#
# export _JAVA_OPTIONS=$(/usr/local/bin/docker-jvm-opts.sh)

# Options:
#   JVM_HEAP_RATIO=0.5 Ratio of heap size to available memory

# If Xmx is not set the JVM will use by default 1/4th (in most cases) of the host memory
# This can cause the Kernel to kill the container if the JVM memory grows over the cgroups limit
# because the JVM is not aware of that limit and doesn't invoke the GC
# Setting it by default to 0.5 times the memory limited by cgroups, customizable with JVM_HEAP_RATIO
CGROUPS_MEM=$(cat /sys/fs/cgroup/memory/memory.limit_in_bytes)
MEMINFO_MEM=$(($(awk '/MemTotal/ {print $2}' /proc/meminfo)*1024))
MEM=$(($MEMINFO_MEM>$CGROUPS_MEM?$CGROUPS_MEM:$MEMINFO_MEM))
JVM_HEAP_RATIO=${JVM_HEAP_RATIO:-0.5}
XMX=$(awk '{printf("%d",$1*$2/1024^2)}' <<<" ${MEM} ${JVM_HEAP_RATIO} ")

# TODO handle cpu limits into -XX:ParallelGCThreads

echo "-Xmx${XMX}m"
```

代码出处是[这位老外的github](https://github.com/carlossg/openjdk/blob/e8bfbbc39ef4aea0fcf07ad6dc43bd11993d3f5b/docker-jvm-opts.sh) 。



### 4.3 移除容器内的SSHD进程

SSHD服务实际上是造成这次"容器没挂、Java挂掉"问题的原因之一：假如没有这个前台执行的SSHD、Java进程放前台执行，那么Java挂掉后，Kubernetes必然会重启容器。


### 4.4 配置Node Eviction Policy，以免容器反复重启导致节点内存耗尽

内核将进程OOM kill其实是一种内核的一种自我保护机制：以免内存耗尽导致宕机。
从kube-scheduler将一个pod调度到某个工作节点开始，直到pod的生命周期结束，该pod都不会在节点之间发生迁移。
这实际上是一个隐患，在最严重的情况下，甚至会导致负载过重的节点因内存耗尽而死机：
```
节点内存不足，内核将容器进程OOM kill  --->  livenessProbe检测出进程挂掉，kubernetes重启容器 
 --->  内核再将容器进程OOM killer   --->  kubernetes再救   
 ----> ... 经过kubernetes和内核的若干轮扯皮...   --->  系统内存耗尽，宕机~
```

[kubernetes社区早就有rescheduler的提案](https://github.com/kubernetes/kubernetes/issues/12140) ，而最近我发现[kubelet Eviction Policy](https://kubernetes.io/docs/tasks/administer-cluster/out-of-resource/#eviction-policy) 正好能够完美地完成rescheduler的工作 。给kubelet加上Eviction Policy配置:
```
--eviction-hard=memory.available<600Mi --eviction-minimum-reclaim=memory.available=0Mi --system-reserved=memory=768Mi 
```
这样，当节点内存紧张时就能自动逐出Pod，从而避免因内存耗尽而宕机。
