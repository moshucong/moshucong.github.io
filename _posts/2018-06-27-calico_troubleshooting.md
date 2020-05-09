---
layout: post
title:  "Calico Troubleshooting"
date:   2018-06-27 16:12:13 +0800
categories: Kubernetes
---


在目前众多的容器网络方案中，[Calico网络的高性能](https://www.projectcalico.org/calico-dataplane-performance/)让人印象深刻。

但同时，calico的原理也是较为复杂的，出现问题时常常让人束手无策。本文介绍解决calico网络troubleshooting的常见方法。


## 1. 环境检查，故障诊断

当我们发现容器网络出现故障（例如某个node上的pod IP突然ping不通），首先当然是要确定问题的原因。

### 1.1 检查calico-node状态

```
# calicoctl node status
Calico process is running.

IPv4 BGP status
+---------------+-------------------+-------+------------+-------------+
| PEER ADDRESS  |     PEER TYPE     | STATE |   SINCE    |    INFO     |
+---------------+-------------------+-------+------------+-------------+
| 172.30.51.95  | node-to-node mesh | up    | 2018-03-29 | Established |
| 172.30.10.186 | node-to-node mesh | up    | 2018-03-29 | Established |
| 172.30.10.187 | node-to-node mesh | up    | 02:28:44   | Established |
+---------------+-------------------+-------+------------+-------------+
```

检查calico log，注意其中的Error信息。

```
# kubectl logs -f calico-node-2wtkj -n kube-system -c calico-node | more 
... ...

# kubectl logs -f calico-kube-controllers-5c884cdd5d-glvws -n kube-system | more 
... ...
```



此处，应该确认的内容包括：

* 节点的hostname是否符合要求。部分calico版本对hostname格式有要求。
* 节点的hostname是否唯一。calico以hostname作为节点的标识符。当节点hostname重复时，后面加入的重名calico节点会启动失败。
* 节点的hostname近期是否发生过变动。当节点hostname发生变化后，calico会认为是一个新的节点加入网络，从而导致异常。
* 网络设备名是否符合要求（通常要求是eth0）。
* 检查有没有其他网络方案在运行，如flannel。两种容器网络方案如果同时运行，通常就会出现难以预料的问题。



### 1.2 检查路由信息

以下是一个正常的calico节点的路由信息。

```
# route -n 
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.30.10.254   0.0.0.0         UG    0      0        0 eth0
10.200.55.192   0.0.0.0         255.255.255.192 U     0      0        0 *
10.200.55.237   0.0.0.0         255.255.255.255 UH    0      0        0 calia45f40cd54e
10.200.55.239   0.0.0.0         255.255.255.255 UH    0      0        0 cali6acd5ed56a0
10.200.153.64   172.30.10.187   255.255.255.192 UG    0      0        0 tunl0
10.200.168.128  172.30.51.95    255.255.255.192 UG    0      0        0 tunl0
10.200.232.128  172.30.10.186   255.255.255.192 UG    0      0        0 tunl0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
172.30.10.0     0.0.0.0         255.255.255.0   U     0      0        0 eth0
```



第1行： Destination为0.0.0.0的流量，最终都交给eth0。

第2行：当前节点的Pod cluster IP range范围。例如，上面表示本节点的Pod IP范围是10.200.55.192/26（10.200.55.193~10.200.55.254）。

第3~4行：当前节点每运行一个k8s Pod，就会生成一条interface以“cali”开头的路由。其中Destination即为该Pod的IP。

第5~7行：每行代表集群中一个calico-node的cluster IP范围，都交由calico tunl0处理。其中Gateway即为node的host IP。

第8行：安装docker后生成的docker0虚拟网桥。与calico网络方案无关。

第9行：Destination为172.30.10.0/24范围的流量，都交给eth0。



## 2. 解决问题后，重置calico-node

通过第1步定位到问题原因，并解决之后，通常需要重置calico node。

重置calico node时，首先要通过calicoctl命令删除calico node。

```
# calicoctl get nodes
NAME           
allen-laptop   
paas-186       
paas-187       
paas-188       

# calicoctl delete node paas-187
Successfully deleted 1 'node' resource(s)
```
由于calico节点的配置信息是保存于ETCD中。如果不先删除calico node，那么calico-node容器重启后还是会到ETCD中读取旧的配置信息，也就起不到更新calico-node配置的效果。

然后，才重启该节点上的calico-node容器。
```
# kubectl delete po calico-node-2wtkj  -n kube-system
pod "calico-node-2wtkj" deleted
```



## 3. 清除路由、网络设备

通常，经过第2步“重置calico-node”之后，故障通常就能排除。

但如果网络仍然不通，那么可以尝试第3步“清除路由、网络设备”。

首先，删除不正常的路由：

```
# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.30.10.254   0.0.0.0         UG    0      0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
172.18.0.0      0.0.0.0         255.255.0.0     U     0      0        0 br-1691e1aaa0bf
172.19.0.0      0.0.0.0         255.255.0.0     U     0      0        0 br-1d6d3edc38f7
172.30.10.0     0.0.0.0         255.255.255.0   U     0      0        0 eth0

# route del -net 172.18.0.0 netmask 255.255.0.0
# route del -net 172.19.0.0 netmask 255.255.0.0

# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.30.10.254   0.0.0.0         UG    0      0        0 eth0
10.200.152.192  0.0.0.0         255.255.255.192 U     0      0        0 *
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
172.30.10.0     0.0.0.0         255.255.255.0   U     0      0        0 eth0
```

然后，删除不正常的网络设备：

```
# ifconfig
br-1691e1aaa0bf Link encap:Ethernet  HWaddr 02:42:9d:6e:51:7a  
          inet addr:172.18.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:9dff:fe6e:517a/64 Scope:Link
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:200 errors:0 dropped:0 overruns:0 frame:0
          TX packets:202 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:10582 (10.5 KB)  TX bytes:34731 (34.7 KB)

br-1d6d3edc38f7 Link encap:Ethernet  HWaddr 02:42:09:2c:96:0a  
          inet addr:172.19.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:9ff:fe2c:960a/64 Scope:Link
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:8 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:536 (536.0 B)  TX bytes:648 (648.0 B)

cali980e384bbd4 Link encap:Ethernet  HWaddr ee:ee:ee:ee:ee:ee  
          inet6 addr: fe80::ecee:eeff:feee:eeee/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:7 errors:0 dropped:2 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:558 (558.0 B)  TX bytes:648 (648.0 B)

docker0   Link encap:Ethernet  HWaddr 02:42:75:de:e3:41  
          inet addr:172.17.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:75ff:fede:e341/64 Scope:Link
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:3 errors:0 dropped:0 overruns:0 frame:0
          TX packets:3 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:216 (216.0 B)  TX bytes:258 (258.0 B)

eth0      Link encap:Ethernet  HWaddr 00:50:56:81:47:25  
          inet addr:172.30.10.187  Bcast:172.30.10.255  Mask:255.255.255.0
          inet6 addr: fe80::250:56ff:fe81:4725/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:1183502 errors:0 dropped:3980 overruns:0 frame:0
          TX packets:218160 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:514474513 (514.4 MB)  TX bytes:19733780 (19.7 MB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:2875 errors:0 dropped:0 overruns:0 frame:0
          TX packets:2875 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:273405 (273.4 KB)  TX bytes:273405 (273.4 KB)

tunl0     Link encap:IPIP Tunnel  HWaddr   
          inet addr:10.200.153.0  Mask:255.255.255.255
          UP RUNNING NOARP  MTU:1440  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

# ip link del  br-1691e1aaa0bf
# ip link del br-1d6d3edc38f7 
# ip link del cali980e384bbd4
```



calico-node重置后，节点上的pod应全部重启，以便更新IP和路由。


