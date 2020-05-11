---
layout: post
title:  "成为Kubernetes运维开发：Service"
date:   2020-05-11 03:14:13 +0800
categories: Kubernetes
---



### 1. Service基本操作

K8s Service有多种类型，我们常用的是如下3种：

1）ClusterIP。

* 单纯的ClusterIP，集群内部使用，其他服务可通过Service IP或Service Name（形式通常是Service名.所在Namespace）进行访问。
* ExternalIPs，开放固定端口到特定的工作节点上。

2）NodePort。在集群**所有工作节点上**开一个**随机端口**。

3）LoadBalancer。LoadBalancer类型Service，本质上是工作节点NodePort之前加一层LoadBalancer（SLB/ELB等），LoadBalancer后端就是所有工作节点上的NodePort。



**创建Service**

**1）创建ExternalIPs类型Service**

创建带ExternalIPs的Service，Spec如下：

```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: super-front
  name: super-front
  namespace: 【环境名】
spec:
  externalIPs:
  - 172.30.10.122
  ports:
  - port: 【Service端口】
    protocol: TCP
    targetPort: 80
  selector:
    app: super-front
  sessionAffinity: None
  type: ClusterIP
```



**2）创建NodePort类型Service**

先创建一个NodePort类型的Service，Spec如下：

```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: super-front
  name: super-front
  namespace: pg-allen
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: super-front
  sessionAffinity: None
  type: NodePort
```

创建完毕，我们看看该Service状态：

```
# kubectl get svc super-front -n pg-allen -o yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{},"labels":{"app":"super-front"},"name":"super-front","namespace":"pg-allen"},"spec":{"ports":[{"port":80,"protocol":"TCP","targetPort":80}],"selector":{"app":"super-front"},"sessionAffinity":"None","type":"NodePort"}}
  creationTimestamp: "2020-04-13T07:03:49Z"
  labels:
    app: super-front
  name: super-front
  namespace: pg-allen
  resourceVersion: "361027126"
  selfLink: /api/v1/namespaces/pg-allen/services/super-front
  uid: ed2cf482-7d54-11ea-94d8-005056814725
spec:
  clusterIP: 192.21.173.229
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 26916
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: super-front
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```

可看出，K8s为这个Service分配了一个NodePort端口（26916）。

我们可尝试通过这个端口访问我们的``super-front``服务：curl http://172.30.10.122:26916/



**3）创建LoadBalancer类型Service**

Service的Spec如下：

```
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/alicloud-loadbalancer-address-type: intranet
  labels:
    app: super-front
  name: super-front
  namespace: 【pg-allen-ali】
spec:
  externalTrafficPolicy: Cluster
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: super-front
  sessionAffinity: None
  type: LoadBalancer
```



稍等片刻，查看Service状态：

```
# kubectl get svc super-front -n pg-allen-ali  -o yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Service","metadata":{"annotations":{"service.beta.kubernetes.io/alicloud-loadbalancer-address-type":"intranet"},"labels":{"app":"super-front"},"name":"super-front","namespace":"pg-allen-ali"},"spec":{"externalTrafficPolicy":"Cluster","ports":[{"port":80,"protocol":"TCP","targetPort":80}],"selector":{"app":"super-front"},"sessionAffinity":"None","type":"LoadBalancer"}}
    service.beta.kubernetes.io/alicloud-loadbalancer-address-type: intranet
  creationTimestamp: "2020-04-13T07:18:37Z"
  labels:
    app: super-front
  name: super-front
  namespace: pg-allen-ali
  resourceVersion: "115422030"
  selfLink: /api/v1/namespaces/pg-allen-ali/services/super-front
  uid: fe6f1de7-7d56-11ea-9128-00163e02276d
spec:
  clusterIP: 172.21.10.137
  externalTrafficPolicy: Cluster
  ports:
  - nodePort: 31426
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: super-front
  sessionAffinity: None
  type: LoadBalancer
status:
  loadBalancer:
    ingress:
    - ip: 10.101.135.10
```

可看出，K8s自动给我们的Service绑了一个NodePort。如果有兴趣，可登录SLB控制台，查看SLB的后端信息。



### 2. Service背后原理



![pre-ccm-arch](images/post-ccm-arch.png)

关键过程：

* ``kube-proxy``对``kube-apiserver``进行List-Watch，当``Service``有变动时，``kube-proxy``更新当前节点的``iptables``规则。
* ``cloud-controller-manager``对``kube-apiserver``进行List-Watch，当Service有变动时，``cloud-controller-manager``调用云提供商的API，更新LoadBalancer配置。

``cloud-controller-manager``的逻辑通常是云供应商所关心的，我们作为用户通常无需关心。为了理解Service，有必要探究下``kube-proxy``的工作。

以``pg-allen``环境的``super-front``为例。

```
# kubectl get svc super-front -n pg-allen -o yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2020-04-11T04:15:56Z"
  labels:
    app: super-front
  name: super-front
  namespace: pg-allen
  resourceVersion: "357815892"
  selfLink: /api/v1/namespaces/pg-allen/services/super-front
  uid: 243475dd-7bab-11ea-b4d6-005056810481
spec:
  clusterIP: 192.21.144.61
  externalIPs:
  - 172.30.10.122
  ports:
  - port: 18080
    protocol: TCP
    targetPort: 80
  selector:
    app: super-front
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
 
# kubectl get po -n pg-allen -owide 
NAME                           READY   STATUS    RESTARTS   AGE   IP               NODE            NOMINATED NODE   READINESS GATES
super-front-5df958954c-j67ft   1/1     Running   0          17m   192.20.98.50     172.30.10.212   <none>           <none>
super-front-5df958954c-rvtlt   1/1     Running   0          18m   192.20.153.236   172.30.11.45    <none>           <none>
```

Service开了ExternalIPs到172.30.10.122上面，``172.30.10.122:18080``是K8s Service ExternalIPs，``192.21.144.61``是K8s Service IP，而后端两个pod IP分别是``192.20.98.50``和``192.20.153.236``。



我们登上这台``172.30.10.122``服务器，看看iptables规则。

```
# iptables -S -t nat
... ...
-A KUBE-SERVICES ! -s 192.20.0.0/16 -d 192.21.144.61/32 -p tcp -m comment --comment "pg-allen/super-front: cluster IP" -m tcp --dport 18080 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 192.21.144.61/32 -p tcp -m comment --comment "pg-allen/super-front: cluster IP" -m tcp --dport 18080 -j KUBE-SVC-2OSY6UR3D26KI2IC

-A KUBE-SERVICES -d 172.30.10.122/32 -p tcp -m comment --comment "pg-allen/super-front: external IP" -m tcp --dport 18080 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 172.30.10.122/32 -p tcp -m comment --comment "pg-allen/super-front: external IP" -m tcp --dport 18080 -m physdev ! --physdev-is-in -m addrtype ! --src-type LOCAL -j KUBE-SVC-2OSY6UR3D26KI2IC
-A KUBE-SERVICES -d 172.30.10.122/32 -p tcp -m comment --comment "pg-allen/super-front: external IP" -m tcp --dport 18080 -m addrtype --dst-type LOCAL -j KUBE-SVC-2OSY6UR3D26KI2IC

-A KUBE-SVC-2OSY6UR3D26KI2IC -m comment --comment "pg-allen/super-front:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-HTIEOR33ZJBBYITG
-A KUBE-SVC-2OSY6UR3D26KI2IC -m comment --comment "pg-allen/super-front:" -j KUBE-SEP-B5KRGOXTA6OFSG4W

-A KUBE-SEP-HTIEOR33ZJBBYITG -s 192.20.153.236/32 -m comment --comment "pg-allen/super-front:" -j KUBE-MARK-MASQ
-A KUBE-SEP-HTIEOR33ZJBBYITG -p tcp -m comment --comment "pg-allen/super-front:" -m tcp -j DNAT --to-destination 192.20.153.236:80

-A KUBE-SEP-B5KRGOXTA6OFSG4W -s 192.20.98.50/32 -m comment --comment "pg-allen/super-front:" -j KUBE-MARK-MASQ
-A KUBE-SEP-B5KRGOXTA6OFSG4W -p tcp -m comment --comment "pg-allen/super-front:" -m tcp -j DNAT --to-destination 192.20.98.50:80
... ...
```

``KUBE-MARK-MASQ``链会给数据包打上Kubernetes独有的MARK标记，用于后续处理。此处可忽略。

第2条：

```
-A KUBE-SERVICES -d 192.21.144.61/32 -p tcp -m comment --comment "pg-allen/super-front: cluster IP" -m tcp --dport 18080 -j KUBE-SVC-2OSY6UR3D26KI2IC
```

凡是目标为``192.21.144.61:18080``（Service IP端口）的TCP数据包，就转给``KUBE-SVC-2OSY6UR3D26KI2IC``链。

第4，第5条：

```
-A KUBE-SERVICES -d 172.30.10.122/32 -p tcp -m comment --comment "pg-allen/super-front: external IP" -m tcp --dport 18080 -m physdev ! --physdev-is-in -m addrtype ! --src-type LOCAL -j KUBE-SVC-2OSY6UR3D26KI2IC
-A KUBE-SERVICES -d 172.30.10.122/32 -p tcp -m comment --comment "pg-allen/super-front: external IP" -m tcp --dport 18080 -m addrtype --dst-type LOCAL -j KUBE-SVC-2OSY6UR3D26KI2IC
```

凡是目标是``172.30.10.122:18080``的TCP数据包，就转给``KUBE-SVC-2OSY6UR3D26KI2IC``链。

第6，第7条：

```
-A KUBE-SVC-2OSY6UR3D26KI2IC -m comment --comment "pg-allen/super-front:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-HTIEOR33ZJBBYITG
-A KUBE-SVC-2OSY6UR3D26KI2IC -m comment --comment "pg-allen/super-front:" -j KUBE-SEP-B5KRGOXTA6OFSG4W
```

`` KUBE-SVC-2OSY6UR3D26KI2IC``链的数据包，50%概率转给``KUBE-SEP-HTIEOR33ZJBBYITG``链，50%概率转给``KUBE-SEP-B5KRGOXTA6OFSG4W``链。

第9条：

```
-A KUBE-SEP-HTIEOR33ZJBBYITG -p tcp -m comment --comment "pg-allen/super-front:" -m tcp -j DNAT --to-destination 192.20.153.236:80
```

``KUBE-SEP-HTIEOR33ZJBBYITG``最终被NAT到``192.20.153.236:80``（其中一个super-front pod）

第11条：

```
-A KUBE-SEP-B5KRGOXTA6OFSG4W -p tcp -m comment --comment "pg-allen/super-front:" -m tcp -j DNAT --to-destination 192.20.98.50:80
```

``KUBE-SEP-B5KRGOXTA6OFSG4W``最终被NAT到``192.20.98.50:80``（另一个super-front pod）。

至此，流量就完成了从Service IP（或ExternalIPs）到pod的整个NAT过程。



### 3. 动手实践



**实验1：创建ClusterIP类型Service，并暴露ExternalIPs**

尝试创建ExternalIPs类型的Service，Service名为``super-front``，添加1个ExternalIPs。

尝试访问ExternalIPs。



**实验2：配置阿里云LoadBalancer类型Service**

在线上环境，尝试创建1个``HTTP``类型的内部Service。Service名为``super-front``，Service类型设置为``LoadBalancer``，相关的``annotation``如下：

|名|值|
|-----|--|
|alicloud-loadbalancer-address-type|intranet|
|alibaba-cloud-loadbalancer-spec|slb.s1.small|
|alicloud-loadbalancer-protocol-port|http:80|
|alicloud-loadbalancer-health-check-flag|on|
|alicloud-loadbalancer-health-check-type|http|
|alicloud-loadbalancer-health-check-connect-timeout|1|
|alicloud-loadbalancer-health-check-timeout|1|
|alicloud-loadbalancer-health-check-interval|10|
|alicloud-loadbalancer-healthy-threshold|1|
|alicloud-loadbalancer-unhealthy-threshold|2|
|alibaba-cloud-loadbalancer-health-check-uri|/|

待LoadBalancer生成出来后，尝试访问。



扩展阅读：

* 通过负载均衡（Server Load Balancer）访问服务（ https://help.aliyun.com/document_detail/86531.html）
