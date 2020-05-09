---
layout: post
title:  "配置docker nofile"
date:   2018-04-13 17:12:13 +0800
categories: Docker
---


Linux通过nofile配置来限制进程能够打开的文件句柄。Linux默认nofile配置为1024，在高并发场景下已经无法满足需求。因此，通常需要将docker容器的nofile配置修改得大一些。


## 1. docker nofile介绍

Linux上通过``ulimit``命令查看nofile，该命令实质上是从``/proc/self/limits ``读取相关信息，所以在``/proc/self/limits``中我们也可以找到``Max open files``信息：
```
root@allen-laptop:~# cat /proc/self/limits 
Limit                     Soft Limit           Hard Limit           Units     
... ...   
Max processes             22813                22813                processes 
Max open files            1024                 4096                 files     
... ...
```

由于Linux内核已经将nofile纳入cgroup控制，因此**docker的nofile与宿主机的nofile是相互独立的**。

例如，我在host机器上查看nofile：
```
root@allen-laptop:~# ulimit -n
1024
```

进入docker容器内，查看nofile：

```
root@allen-laptop:~# docker exec -it a44a6d87f9e9 /bin/bash
root@ubuntu-1:/# ulimit -n
524288
```

**这意味着：我们修改宿主机上的nofile配置，实质上并不能影响docker容器内的nofile配置。**

## 2. docker nofile配置

针对docker不同的启动方式，nofile配置也不一样。

### 2.1 在docker默认配置文件中修改nofile配置

如果docker采用``service docker start``启动，那么docker将默认从``/etc/init/docker.conf``配置文件中获取nofile配置 ：

```
root@allen-10:/home/allen# cat /etc/init/docker.conf
description "Docker daemon"

start on (filesystem and net-device-up IFACE!=lo)
stop on runlevel [!2345]
limit nofile 524288 1048576
limit nproc 524288 1048576
```
可看出，docker容器默认nofile默认配置为524288（软限制）和1048576（硬限制）。
如果我们使用``service docker start`` ，那么docker启动时将读取这个nofile配置，并应用到该宿主机上启动的所有容器。

### 2.2 在docker daemon命令参数中修改nofile配置

如果通过docker daemon命令启动docker ，那么需在命令中加入"--default-ulimit nofile"配置并重启docker即可。如：

```
docker daemon --default-ulimit nofile=524288:1048576 --default-ulimit nproc=524288:1048576 --bip=${FLANNEL_SUBNET} --mtu=${FLANNEL_MTU} ${insecure_registries} --log-level=${DOCKER_LOG_LEVEL} --storage-driver=${STORAGE} >> $DOCKER_LOG_FILE 2>&1 &
```


