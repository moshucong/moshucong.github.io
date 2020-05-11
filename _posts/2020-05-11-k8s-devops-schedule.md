---
layout: post
title:  "成为Kubernetes运维开发（9）：常用调度"
date:   2020-05-11 09:14:13 +0800
categories: Kubernetes
---


调度，指的是k8s选择一个node并将其余pod进行绑定的过程。



### 1. node的常用调度操作

**1）隔离节点**

当节点维护时，不希望pod调度进来，可通过``kubectl cordon``命令隔离节点：

```
# kubectl cordon [node name]
node/172.30.10.127 cordoned
```

pod无法调度到被隔离的node上，但如下两类pod除外：

* Daemonset生成的pod。
* 容忍所有污点的pod。例如pod添加如下toleration后，可调度到被隔离的node：

```
tolerations:
- operator: "Exists"
```

解除节点隔离：

```
# kubectl uncordon [node name]
node/172.30.10.127 uncordoned
```



**2）排空节点**

用于隔离，然后删除节点上的所有pod。

```
# kubectl drain [node name] --ignore-daemonsets
node/172.30.10.127 cordoned
WARNING: ignoring DaemonSet-managed Pods: aiops-188/aiops-agent-t77r6, kube-system/calico-node-dpsql, kube-system/kube-proxy-rt5bh, pg-summer/node-exporter-dndwf
evicting pod "data-collect-c78574bd7-j64kj"
pod/data-collect-c78574bd7-j64kj evicted
node/172.30.10.127 evicted
```

由于daemonset生成的pod不受cordon限制，因此添加``--ignore-daemonsets``忽略daemonset。

在回收节点前，通常会先执行drain命令排空节点。



### 2. nodeSelector

nodeSelector是基本的pod调度机制。通过pod的nodeSelector与node的label进行匹配，来选择pod绑定的node。

首先，给node打上label。这里我们给node打了一个label，label的value是``proj/k8s-playground``，key是``true``。

```
# kubectl label node [node name] proj/k8s-playground=true
node/172.30.10.127 labeled
# kubectl label node 172.30.10.127 app/pg-allen=true      #修改为个人namespace
node/172.30.10.127 labeled
```

查看label：

```
# kubectl get node [node name] --show-labels 
NAME            STATUS   ROLES    AGE    VERSION   LABELS
172.30.10.127   Ready    <none>   314d   v1.9.8    app/pg-allen=true,beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=172.30.10.127,proj/k8s-playground=true
```



然后，给pod加上nodeSelector配置：

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: super-front
  namespace: pg-allen
spec:
  template:
... ...
    spec:
      nodeSelector:
        proj/k8s-playground: "true"
        app/pg-allen: "true"     #修改为个人namespace
  ... ...
```

新启动的pod是否被调度到你打了label的节点？



去掉node上的label：

```
# kubectl label node [node name] proj/k8s-playground-
node/172.30.10.127 labeled
# kubectl label node 172.30.10.127 app/pg-allen-
node/172.30.10.127 labeled
```



### 3. taint与toleration

taint（污点）/toleration（容忍）是另一种调度机制，两者须结合使用。

可通过``kubectl taint``命令给节点加污点：

```
# kubectl taint node/nodes [node name] [key]=[value]:[PreferNoSchedule/NoSchedule/NoExecute]
```

taint有三个级别：

* PreferNoSchedule：对于不满足taint/toleration要求的pod，**尽量**不调度到该node。
* NoSchedule：对于不满足taint/toleration要求的pod，**不允许**调度到该node。
* NoExecute：对于不满足taint/toleration要求的pod，**不允许**调度到该node。若不满足要求的pod已运行在node上，则**逐出**。

其中最常用的是NoSchedule。

去掉taint：

```
# kubectl taint node/nodes [node name] [key]:[PreferNoSchedule/NoSchedule/NoExecute]-
```



**场景1：设置node仅允许运行特定的pod**

首先，给node加上taint。

```
# kubectl taint node [node name] proj=k8s-playground:NoSchedule
```

上述命令给节点添加了污点``proj=k8s-playground:NoSchedule``，pod须加上key为proj，value为k8s-playground，effect为NoSchedule的toleration，才允许调度到该节点上。

删除super-front pod，看自动重启的pod还能调度到上述节点吗？



修改deployment，添加toleration：

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: super-front
  namespace: pg-allen
spec:
  template:
... ...
    spec:
      tolerations:
      - effect: NoSchedule
        key: proj
        value: k8s-playground
        operator: Equal
... ...
```

添加toleration后，滚动升级的pod可以调度到上述节点吗？



**场景2：让特定的pod运行在master上**

集群master默认添加了污点``node-role.kubernetes.io/master:NoSchedule``。

如果希望pod运行在master上，可给spec添加如下toleration：

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: super-front
  namespace: pg-allen
spec:
  template:
... ...
    spec:
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
        operator: Equal
... ...
```

对于部分系统级应用（如kube-proxy、flanneld等），可以添加一个容忍所有污点的toleration：

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: super-front
  namespace: pg-allen
spec:
  template:
... ...
    spec:
      tolerations:
      - effect: NoSchedule
        operator: Equal
... ...
```



**场景3：逐出Daemonset的pod**

在实践中，有时需要将Daemonset的pod暂时从特定的node上逐出。但由于Daemonset的特殊性，节点隔离对Daemonset是不起作用的。这种情况，可通过taint机制完成。

例如，我们要逐出``172.30.10.127``上的``aiops-188``空间的``aiops-agent``。

首先，需要给node添加taint：

```
# kubectl taint node [node name] ops=true:NoSchedule
node/172.30.10.127 tainted
```

然后，删除该节点上的daemonset pod（可以aiops-agnet为例），该daemonset pod在节点上自动重启吗？



### 4.  亲和/反亲和

提供如下4中亲和/反亲和策略：
* podAntiAffinity。pod反亲和，反亲和的pod倾向于调度到不同node上。
* podAffinity。pod亲和，亲和的pod倾向于调度到同一个node上。
* nodeAffinity。pod与node亲和，pod倾向于调度到亲和的node上。由于``operator``可以取反（NotIn），因此nodeAffinity也可用于配置pod与node的反亲和。

在实践中，最常用的是**podAntiAffinity**。podAntiAffinity最常见的使用场景是，为了避免同一应用的多个pod实例被调度到同一个node上，因单个node故障而整体失效，设置应用的pod与自身反亲和。



对于上述策略，又各有两种具体类型：

* requiredDuringSchedulingIgnoredDuringExecution。pod调度过程**必须选择**符合亲和/反亲和的node，但已在运行中的pod无需逐出。
* preferredDuringSchedulingIgnoredDuringExecution。pod调度过程**优先选择**符合亲和/反亲和的node，但已在运行中的pod无需逐出。



尝试给super-front添加如下``podAntiAffinity``配置：

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: super-front
  namespace: pg-allen
spec:
  template:
... ...
    specffinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - super-front
              topologyKey: kubernetes.io/hostname
            weight: 100
```

滚动升级的pod可以调度成功吗？

尝试把``preferredDuringSchedulingIgnoredDuringExecution``改为``requiredDuringSchedulingIgnoredDuringExecution``：

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: super-front
  namespace: pg-allen
spec:
... ...
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - super-front
            topologyKey: kubernetes.io/hostname
```

滚动升级的pod还能调度成功吗？




### 5. 其他影响调度的因素

**1）CPU/Memory request**

节点上pod的资源request之和，必然小于等于节点可供分配（allocatable）资源。

查看node可供pod分配的资源量：

```
# kubectl describe node 172.30.10.127
Name:               172.30.10.127
... ...
Capacity:
 cpu:     4
 memory:  6110476Ki
 pods:    110
Allocatable:
 cpu:     4
 memory:  5138964681
 pods:    110
... ...
```



**2）hostNetwork**

当pod配置成hostNwtwork时，k8s会根据pod监听的端口，将pod调度到不会发生端口冲突的节点上。



