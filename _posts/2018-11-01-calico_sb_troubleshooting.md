---
layout: post
title:  "一次沙雕的calico troubleshooting过程"
date:   2018-11-01 12:14:13 +0800
categories: calico
---

某台机器加入calico网络失败。本文介绍troubleshooting过程，希望引以为鉴。

环境说明如下。

| 组件         | 说明                                       |
| ---------- | ---------------------------------------- |
| kubernetes | 版本1.9.6。3个k8s master部署在aws新加坡区域，工作节点分布在新加坡、美东、东京、美西等区域 |
| etcd       | 版本3.2.17。3个etcd节点部署在k8s master所在的3台机器上   |
| calico     | 版本2.6.5。容器部署，calico-node以daemonset部署     |

 

### 1. 问题表现

我将某台新机器（aws美东区域）加到容器集群之后，发现该节点没法加入到容器集群里面。

该节点的calico-node启动后不久反复的crash重启。crash前的log如下：

```
root@master-host:/# kubectl logs -f calico-node-wm6bb  -n kube-system -c calico-node
Skipping datastore connection test
Using autodetected IPv4 address 10.12.13.12/24 on matching interface eth0
```

恰巧新加坡和美东区域各有一台闲置机器，尝试将这两台机器加入calico网络。发现新加坡机器成功加入到calico网络，而美东机器加入calico失败，且失败表现相同。

这究竟是什么鬼原因？



### 2.troubleshooting过程

#### 2.1 排除ETCD连接问题

calico-node启动阶段会访问etcd获取集群网络配置，所以首先怀疑会不会是节点连接etcd失败。

在问题节点上，尝试访问etcd：

```
root@hostname:/#  curl --cacert /etc/cni/net.d/calico-tls/etcd-ca --cert /etc/cni/net.d/calico-tls/etcd-cert --key /etc/cni/net.d/calico-tls/etcd-key https://[ETCD服务IP]:2379/health
{"health": "true"}
```

发现可以连上etcd，很失望。那么可以排除etcd连接问题。




#### 2.2 排除AWS路由表限制

[AWS EC2路由表有50条的数量限制](https://docs.aws.amazon.com/vpc/latest/userguide/amazon-vpc-limits.html#vpc-limits-route-tables) ，这有可能会限制集群的节点数上限。[但根据calico官方文档，calico应该不受aws 50节点的限制](https://docs.projectcalico.org/v3.2/reference/public-cloud/aws) :

```
No 50 Node Limit: Calico allows you to surpass the 50 node limit, which exists as a consequence of the AWS 50 route limit when using the VPC routing table.
```

而且，我们的集群节点数已经超出50（目前节点数53）。

至此，排除AWS路由表原因。



#### 2.3 排除aws security groups影响

仔细看了[calico的aws部署说明，其中提到需要配置aws安全组，允许BGP和IPIP通信](https://docs.projectcalico.org/v2.6/reference/public-cloud/aws)，会不会因为公司aws美东区域安全组配置的原因？

查看美东那两台问题机器的security-groups是 DY-Default-10.12，而其他机器的 security-groups是DY-Default。会不会是因为安全组配置的原因，导致加入网络失败？能否将美东机器也加到 DY-Default这个安全组里面？

咨询运维同事后得知，不同区域的安全组虽然名字可以一致，但仍然是独立的安全组，并且已经开放了所需的端口访问权限。

很失望。可以排除aws security group的影响。



#### 2.4 问题原因

至此，问题原因仍毫无头绪。只好找来[calico node的启动代码](https://github.com/projectcalico/node/blob/4db4e815e47885db77957e113a18269fa1ce0ffd/pkg/startup/startup.go#L233)来看看。

期间发现，calico node启动时是依据``CALICO_STARTUP_LOGLEVEL``环境变量来设置log级别。考虑到出问题的calico node输出的log实在是太少，修改calico node的daemonset配置，将log等级设置成最低的DEBUG级别：

```
    spec:
      containers:
      - env:
        - name: CALICO_STARTUP_LOGLEVEL
          value: DEBUG
```

重启问题节点的calico-node，果然输出了更多的log：

```
root@hostname:/tmp# kubectl logs -f calico-node-jz7cz -n kube-system -c calico-node 
time="2018-10-31T07:47:16Z" level=info msg="Early log level set to debug" 
time="2018-10-31T07:47:16Z" level=info msg="NODENAME environment not specified - check HOSTNAME" 
time="2018-10-31T07:47:16Z" level=info msg="Loading config from environment" 
time="2018-10-31T07:47:16Z" level=debug msg="Using datastore type 'etcdv2'" 
Skipping datastore connection test
time="2018-10-31T07:47:16Z" level=debug msg="Validate name: AMZ-IAD12-OpsResPool-13-33" 
time="2018-10-31T07:47:16Z" level=debug msg="Get Key: /calico/v1/host/AMZ-IAD12-OpsResPool-13-33/metadata" 
time="2018-10-31T07:47:17Z" level=debug msg="Key not found error" 
time="2018-10-31T07:47:17Z" level=debug msg="Key not found error" 
time="2018-10-31T07:47:17Z" level=info msg="Building new node resource" Name=AMZ-IAD12-OpsResPool-13-33 
time="2018-10-31T07:47:17Z" level=info msg="Initialise BGP data" 
time="2018-10-31T07:47:17Z" level=debug msg="Querying interface addresses" Interface=eth0 
time="2018-10-31T07:47:17Z" level=debug msg="Found valid IP address and network" CIDR=10.12.13.33/24 
time="2018-10-31T07:47:17Z" level=debug msg="Check interface" Name=eth0 
time="2018-10-31T07:47:17Z" level=debug msg="Check address" CIDR=10.12.13.33/24 
Using autodetected IPv4 address 10.12.13.33/24 on matching interface eth0
time="2018-10-31T07:47:17Z" level=info msg="Node IPv4 changed, will check for conflicts" 
time="2018-10-31T07:47:17Z" level=debug msg="Listing all host metadatas" 
time="2018-10-31T07:47:17Z" level=debug msg="Parse host directories." 
time="2018-10-31T07:47:17Z" level=debug msg="Get Node key from /calico/v1/host/AMZ-IAD12-Coupon-35-221/metadata" 
time="2018-10-31T07:47:17Z" level=debug msg="Get Node key from /calico/v1/host/AMZ-IAD12-Coupon-35-222/metadata" 
time="2018-10-31T07:47:17Z" level=debug msg="Get Node key from /calico/v1/host/AMZ-IAD12-OpsResPool-13-31/metadata" 

.... 省略类似log ....

time="2018-10-31T07:47:26Z" level=debug msg="Get Key: /calico/bgp/v1/host/AMZ-OR39-CTR-135-60/ip_addr_v4" 
time="2018-10-31T07:47:26Z" level=debug msg="Get Key: /calico/bgp/v1/host/AMZ-OR39-CTR-135-60/network_v4" 
time="2018-10-31T07:47:26Z" level=debug msg="Get Key: /calico/bgp/v1/host/AMZ-OR39-CTR-135-60/ip_addr_v6" 
time="2018-10-31T07:47:26Z" level=debug msg="Get Key: /calico/bgp/v1/host/AMZ-OR39-CTR-135-60/network_v6" 
time="2018-10-31T07:47:26Z" level=debug msg="Key not found error" 
time="2018-10-31T07:47:26Z" level=debug msg="Get Key: /calico/bgp/v1/host/AMZ-OR39-CTR-135-60/as_num" 
time="2018-10-31T07:47:27Z" level=debug msg="Key not found error" 
time="2018-10-31T07:47:27Z" level=debug msg="Get Key: /calico/v1/host/AMZ-OR39-CTR-135-60/orchestrator_refs" 

.... 省略类似log ....

time="2018-10-31T07:48:12Z" level=debug msg="Get Key: /calico/bgp/v1/host/AMZ-SIN8-OpsResPool-33-62/ip_addr_v4" 
time="2018-10-31T07:48:13Z" level=debug msg="Get Key: /calico/bgp/v1/host/AMZ-SIN8-OpsResPool-33-62/network_v4" 
time="2018-10-31T07:48:13Z" level=debug msg="Get Key: /calico/bgp/v1/host/AMZ-SIN8-OpsResPool-33-62/ip_addr_v6" 
time="2018-10-31T07:48:13Z" level=debug msg="Get Key: /calico/bgp/v1/host/AMZ-SIN8-OpsResPool-33-62/network_v6" 
time="2018-10-31T07:48:13Z" level=debug msg="Key not found error" 
time="2018-10-31T07:48:13Z" level=debug msg="Get Key: /calico/bgp/v1/host/AMZ-SIN8-OpsResPool-33-62/as_num" 
time="2018-10-31T07:48:14Z" level=debug msg="Key not found error" 
time="2018-10-31T07:48:14Z" level=debug msg="Get Key: /calico/v1/host/AMZ-SIN8-OpsResPool-33-62/orchestrator_refs" 
time="2018-10-31T07:48:14Z" level=debug msg="Get Key: /calico/bgp/v1/host/AMZ-SIN8-OpsResPool-33-63/ip_addr_v4" 
root@hostname:/tmp# 
```

输出完上面的log后，calico-node就被重启了，是什么原因呢？

根据calico node的代码，[如果calico node启动失败，那退出前会先打一行log：Terminating](https://github.com/projectcalico/node/blob/4db4e815e47885db77957e113a18269fa1ce0ffd/pkg/startup/startup.go#L1003)，但在上面的log中并未发现有``Terminating``。

从log看出，calico-node终止前是在查询etcd中的节点数据。于是又试了试在问题节点上查询etcd：

```
root@hostname:/# etcdctl  --ca-file=/var/lib/kubernetes/ca.pem --cert-file=/var/lib/kubernetes/kubernetes.pem --key-file=/var/lib/kubernetes/kubernetes-key.pem --endpoints=https://[etcd服务ip]:2379 get /calico/bgp/v1/host/AMZ-SIN8-OpsResPool-33-63/ip_addr_v4
10.8.33.63
```

看结果很正常，绝不会因此而出错。



纠结之际，突然想到：会不会calico-node启动超时了？

马上看了下calico-node daemonset的livenessProbe：

```
        livenessProbe:
          failureThreshold: 6
          httpGet:
            path: /liveness
            port: 9099
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
```

尝试着把initialDelaySeconds加到60秒，failureThreshold加到10。

```
        livenessProbe:
          failureThreshold: 10
          httpGet:
            path: /liveness
            port: 9099
            scheme: HTTP
          initialDelaySeconds: 60
          periodSeconds: 10
          successThreshold: 1
          timeoutSeconds: 1
```

改完重启了问题节点的calico-node。等待了数分钟，发现居然启动成功了。

至此，问题原因就明确了：**由于集群节点越来越多，calico-node启动所需时间也随着变长了，超出了liveness probe的重启时间限制，从而被k8s干掉重启。** 



那么为什么新加坡区域的节点启动成功，而美东的节点启动失败？

原因应该是，我们的master与etcd都在新加坡区域，因此新加坡节点从etcd获取数据较快，calico-node启动速度也就更快，可以在限定的时间内启动完毕。而美东节点访问新加坡etcd的延迟较长，因此美东calico-node启动更慢。

真是八十岁老娘崩倒孩儿。

