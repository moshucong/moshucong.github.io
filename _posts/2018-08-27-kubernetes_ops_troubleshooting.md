---
layout: post
title:  "Kubernetes集群运维踩坑记录"
date:   2018-08-27 18:12:13 +0800
categories: Docker
---


本文目的是回顾容器集群运维过程中发现并填掉的一些坑，作为前车之鉴。



### 1. 高版本docker与老版本Linux内核不兼容，导致内存泄露

严重性：★★★★（导致机器内存泄露。在高负载情况下，有可能导致节点宕机）

#### 1.1 问题表现

某天收到告警，容器集群中某个计算节点ICMP ping失败。上线一看，机器已挂，重启后查看内核日志。

```
Jun  8 03:03:28 [node hostname] kernel: [88820.247535] SLUB: Unable to allocate memory on node -1 (gfp=0x8020)
Jun  8 03:03:28 [node hostname] kernel: [88820.247541]   cache: kmalloc-192(33:b40c3884668600191e02b3c32efaf12bdd1e6d0a4d3869662045e4193af6c26c), object size: 192, buffer size: 192, default order: 0, min order: 0
Jun  8 03:03:28 [node hostname] kernel: [88820.247543]   node 0: slabs: 202, objs: 4242, free: 0
Jun  8 03:03:28 [node hostname] kernel: [88820.254869] SLUB: Unable to allocate memory on node -1 (gfp=0x8020)
Jun  8 03:03:28 [node hostname] kernel: [88820.254873]   cache: kmalloc-192(33:b40c3884668600191e02b3c32efaf12bdd1e6d0a4d3869662045e4193af6c26c), object size: 192, buffer size: 192, default order: 0, min order: 0
Jun  8 03:03:28 [node hostname] kernel: [88820.254875]   node 0: slabs: 202, objs: 4242, free: 0
```

重启后，机器仍不断打出该log。但``free``查看内存，发现内存有空闲。



#### 1.2 原因

查阅资料，发现[docker的github上类似的issue](https://github.com/moby/moby/issues/27576)，而且[国内也有报告类似的问题](http://www.linuxfly.org/kubernetes-19-conflict-with-centos7/) 。

我们的操作系统发行版是Ubuntu 14.04.1，Linux内核版本是3.13。这次搭建的新集群使用docker1.12.6，kubernetes 1.9.6。

这个问题简而言之，原因是新版docker启用Linux CGroup memory这个feature，但这个feature在kernel 4.0以下版本中是非稳定版本（经过实践，即使是kernel 4.2也不靠谱），会导致内存泄露。



#### 1.3 解决方案

最终，将升级内核到4.9后，问题不再重现。



### 2. 默认容器文件系统采用了device mapper，导致硬盘空间频繁被占满

严重性：★★（导致硬盘空间消耗过快，需频繁清理硬盘空间）

#### 2.1 问题表现

升级内核到4.9后，集群运行正常。

但正常运行两天后，收到磁盘空间的告警，提示根目录``/``、``/var/lib/docker``、``/var/lib/kubelet`` 、``/var/lib/docker/devicemapper``的磁盘空间已经超过80%。登录机器后发现，根目录所在硬盘（30GB）快满了。



#### 2.2 原因

原因是**升级内核后，docker默认采用的文件系统由aufs变成了devicemapper，而devicemapper所占空间比较多**。

所用docker版本是1.12.6，根据[该版本的docker源码](https://github.com/moby/moby/blob/1.12.x/daemon/graphdriver/driver_linux.go#L53-L62) ，docker选择文件系统的优先级如下：

```
	// Slice of drivers that should be used in an order
	priority = []string{
		"aufs",
		"btrfs",
		"zfs",
		"devicemapper",
		"overlay",
		"vfs",
	}
```

将内核升级4.9之后，内核已经不再集成aufs模块。

```
# cat /proc/filesystems | grep aufs
# cat /proc/filesystems | grep overlay
nodev	overlay
```

而且devicemapper的优先级是高于overlay，因此docker最终选择了devicemapper作为文件系统。



#### 2.3 解决方案

解决方案是将docker文件系统改为占用硬盘空间较少的overlay2。

修改docker启动配置`` /etc/default/docker``， 修改docker配置文件中的``DOCKER_OPTS``，增加：

```
-s overlay2
```

删除docker文件，并重启docker 和kubelet：

```
service docker stop
service kubelet stop

rm /var/lib/docker -rf

service docker start
/opt/paas-deploy/script/node-start.sh
```

PS：若不清除旧文件，则docker daemon启动后将无响应。



### 3. Eviction Policy逐出kube-proxy，导致服务无法访问 

严重性：★★★★（若节点的kube-proxy，有可能导致服务部分的外部IP/端口无法访问）

#### 3.1 问题表现

某天收到告警，某个服务的外部端口连接失败。上线一看，发现节点的kube-proxy进程都没有了。



#### 3.2 原因 

由于节点的kube-proxy是以``static pod``的方式部署，因此出问题后首先查看kubelet日志，在log中发现如下信息：

```
46948 I0617 03:05:13.803095    5478 eviction_manager.go:346] eviction manager: must evict pod(s) to reclaim imagefs
46949 I0617 03:05:13.803169    5478 eviction_manager.go:364] eviction manager: pods ranked for eviction: kube-proxy-[node IP]_kube-system(dc6bd3a06883f50ce46d35fd024f66f2), config-88df876c5-pmk67_ec-prod(e134b5a2-6fc1-11e8-      b1d1-0226410ff50c), data-services-6b4df97cdf-qz8sd_ec-prod(da14a6d4-7158-11e8-ba3c-062f143c52c6), rulescript-services-5ffdfd7f7-c5ph8_ec-prod(534a6a09-7079-11e8-ba3c-062f143c52c6), mq-services-69758bfc6d-7cn94_ec-prod(      326cedc8-70e0-11e8-ba3c-062f143c52c6), fbagent-services-67d7b788cb-pmd8d_ec-prod(a07d489a-71bc-11e8-ba3c-062f143c52c6)
46950 I0617 03:05:13.803305    5478 kuberuntime_container.go:582] Killing container "docker://853a131b5d9a577590773c677898ae61ca6a411e1046cd3422fa0d97d9547258" with 30 second grace period
46951 I0617 03:05:14.053822    5478 kubelet.go:1867] SyncLoop (DELETE, "api"): "aiops-trjtv_aiops-sig(407e007c-71db-11e8-ba3c-062f143c52c6)"
46952 I0617 03:05:14.058557    5478 kubelet.go:1861] SyncLoop (REMOVE, "api"): "aiops-trjtv_aiops-sig(407e007c-71db-11e8-ba3c-062f143c52c6)"
46959 I0617 03:05:14.896640    5478 eviction_manager.go:156] eviction manager: pods kube-proxy-[node IP]_kube-system(dc6bd3a06883f50ce46d35fd024f66f2) evicted, waiting for pod to be cleaned up
46960 I0617 03:05:14.901777    5478 kubelet.go:1896] SyncLoop (PLEG): "kube-proxy-[node IP]_kube-system(dc6bd3a06883f50ce46d35fd024f66f2)", event: &pleg.PodLifecycleEvent{ID:"dc6bd3a06883f50ce46d35fd024f66f2", Type:"Contain      erDied", Data:"853a131b5d9a577590773c677898ae61ca6a411e1046cd3422fa0d97d9547258"}
46961 I0617 03:05:14.901864    5478 kubelet.go:1896] SyncLoop (PLEG): "kube-proxy-[Node IP]_kube-system(dc6bd3a06883f50ce46d35fd024f66f2)", event: &pleg.PodLifecycleEvent{ID:"dc6bd3a06883f50ce46d35fd024f66f2", Type:"Contain      erDied", Data:"b02a91b9ea5ba55f8e3889deb9ff2315c9e0d07426d5f15c9a954c9036c34218"}
```

显然，问题发生时，imagefs满了。为了回收硬盘空间，kubelet的Eviction Policy将kube-proxy逐出，导致无法通过Service的external IP/port访问容器。

为了提高运维自动化程度，kubelet配置了Eviction Policy，当硬盘空闲率低于一定百分比则自动开始逐出容器，并清理容器和镜像所占存储空间。如下：

```
 --eviction-hard=memory.available<5%,nodefs.available<25%,imagefs.available<25% --eviction-minimum-reclaim=memory.available=0Mi,nodefs.available=5%,imagefs.available=5% --system-reserved=memory=768Mi
```



#### 3.3 解决方案

临时解决方案是，暂时关闭节点的Eviction Policy。

关于这个问题，[github上也有一些相关的讨论](https://github.com/kubernetes/kubernetes/issues/40573)，官方推荐通过[critical pod](https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/)的方式解决该问题。对于我们所采用的kubernetes 1.9，[critical pod的配置](https://v1-9.docs.kubernetes.io/docs/concepts/configuration/pod-priority-preemption/)比较的麻烦。

另外，我注意到 Eviction是无法逐出daemonset所启动的pod的。因此，**只需将kube-proxy部署方式由static pod改为daemonset方式部署即可** 。



### 4. 节点open file数消耗完，导致节点的容器网络失效

严重性：★★★★★（连接数较多的情况下，有可能导致服务中断）

#### 4.1 问题表现

某天，一众业务同事、运维同事火急火燎找到我，表示业务中断。问题的具体表现包括：服务域名监控返回502，用户无法访问服务。在容器中无法连上Redis服务器、容器中无法对Redis域名进行DNS解析、容器中无法解析任何域名。



#### 4.2 原因

根据同事的反馈逐项进行排查，发现容器系统服务正常，DNS服务正常，问题似乎并不是出在容器系统服务上。

排查calico网络，发现3个节点的calico-node处于``start``状态（正常状态是``up``），而这三个异常节点正是项目externalIPs所绑定的节点。

查看calico-node的log发现如下信息：

```
Active Socket: Connection reset by peer
```

显然，这三个节点的calico-node处于异常状态。尝试重启``calico-node``，但问题并未得到解决。

查看节点的网络连接，发现大量连接处于TIME_WAIT状态。查看nofile限制，发现上限是1024（Ubuntu默认配置）：

```
# ulimit -n
1024
```

修改nofile限制，重启calico-node，问题解决。



#### 4.3 解决方案

修改节点的``/etc/security/limits.conf`` ，将``root``用户的open file数增加到102400：

```
root soft nofile 102400
root hard nofile 102400
* soft nofile 102400
* hard nofile 102400
* hard nproc 8192
* soft nproc 8192
```

将修改后的系统做成镜像，以后申请机器时采用新镜像。



### 5. 节点hostname改动，导致节点calico网络故障 

严重性：★★★★★（导致节点容器网络失效）

#### 5.1 问题表现

某天，人已经在家中坐。收到同事消息，说是容器启动失败。上线一看，发现该项目4个节点上新启动的容器确实无法进行网络通信。

检查容器集群网络状态，发现出现了4个没见过的新节点。

```
# calicoctl node status

# 4个节点的hostname
AMZ-IAD12-OpsResPool-xxx-xx
AMZ-IAD12-OpsResPool-xxx-xx
AMZ-IAD12-OpsResPool-xxx-xx
AMZ-IAD12-OpsResPool-xxx-xx

.... ....

# 4个未见过的hostname
ip-xx-xx-xxx-xx.ec2.internal
ip-xx-xx-xxx-xx.ec2.internal
ip-xx-xx-xxx-xx.ec2.internal
ip-xx-xx-xxx-xx.ec2.internal
```



#### 5.2 原因

经沟通，得知同事当天修改过这4个节点的hostname，从``AMZ-IAD12-OpsResPool-xxx-xx``改成了``ip-xx-xx-xxx-xx.ec2.internal``格式 。至此，问题的原因明确了：

容器集群的网络方案是calico网络。calico将节点的网络信息保存于etcd数据库（key-value）中，并使用节点的hostname作为数据的key。当节点的hostname发生变化时，calico会将其识别为一个新的节点。etcd中会同时存储一个节点的两份不同的配置，导致节点的calico网络出现故障。



#### 5.3 解决方案

**容器集群节点的hostname非常关键。当运维或业务同事需要修改容器集群节点的hostname时，一定要提前通知容器集群的管理员**。

当节点hostname发生变化后，集群管理员需重置该节点的calico网络，步骤如下：

* 首先，通过``calicoctl``命令删除旧的calico节点，清除etcd中该节点的信息。
* 其次，通过``kubectl``命令重启该节点上的``calico-node``容器，让节点calico网络重置。
* 最后，通过``kubectl``命令重启该节点上所有采用容器网络的pod。





