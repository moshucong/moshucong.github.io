---
layout: post
title:  "成为Kubernetes运维开发（2）：认识工作负载Deployment"
date:   2020-05-11 02:14:13 +0800
categories: Kubernetes
---


Kubernetes将设置pod的部署规则的对象称为工作负载（workload）。在部署应用时，我们通常不会直接创建pod。而是通过创建工作负载，让Kubernetes为我们创建和管理所需的pod。

常用的工作负载有如下5种：

* Deployment
* Statefulset
* Daemonset
* Job
* Cronjob



### 1. Deployment基本操作

Kubernetes Deployment是用于部署无状态应用。在实践中，我们开发的**绝大部分应用都属于无状态应用**，因此Deployment也是五类工作负载中最常用的。



作为示例，我们来创建一个``deployment``。



**创建Deployment**

首先编辑一个``deploy.yml``，内容如下：

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    app: super-front
    codename: k8s-playground
    project: k8s-playground
  name: super-front
  namespace: 【你的环境名】
spec:
  minReadySeconds: 30     # pod启动后，当liveness和readiness均为true之后，经过min ready seconds的时间后，则认为容器启动成功
  replicas: 1    # pod实例数。如果启用HPA，则以HPA预期实例数为准。
  selector:
    matchLabels:
      app: super-front
  strategy:
    rollingUpdate:
      maxSurge: 1    # 在滚动升级过程中，可超出预期replicas的pod个数。max unavailable和max surge不能同时为0，否则将无法滚动升级。
      maxUnavailable: 0   # 在滚动升级过程中，处于不可用状态的pod数的上限。max unavailable和max surge不能同时为0，否则将无法滚动升级。
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: super-front
        project: k8s-playground
    spec:
      containers:
      - env:   # 环境变量。key-value的形式。key需符合Linux环境变量名字要求
        - name: LANG
          value: en_US.UTF-8
        - name: LANGUAGE
          value: en_US:en
        - name: LC_ALL
          value: en_US.UTF-8
        image: nginx:latest
        imagePullPolicy: Always
        livenessProbe:   # container的生存状态探针。若该探针返回异常状态，则重启container。
          failureThreshold: 3
          initialDelaySeconds: 168   # container启动后，首次执行liveness probe的延迟时间
          periodSeconds: 20   # liveness probe执行周期时间
          successThreshold: 1
          tcpSocket:
            port: 80
          timeoutSeconds: 1
        name: super-front
        readinessProbe:   # container准备状态探针。若该探针返回异常状态，则认为container未准备好处理请求
          failureThreshold: 3
          httpGet:
            path: /
            port: 80
            scheme: HTTP
          initialDelaySeconds: 11
          periodSeconds: 20
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          limits:
            cpu: 10m    #container占用的cpu资源上限
            memory: 64Mi    #container占用的内存资源上限
          requests:
            cpu: 10m    #container预留的cpu资源
            memory: 64Mi   #container预留的内存资源
      dnsPolicy: Default
      restartPolicy: Always
```



通过``kubectl apply``创建Deployment：

```
# kubectl apply -f ./deploy.yml 
deployment.extensions/super-front created
```



**获取查看Deployment**

```
# kubectl get deploy -n pg-allen
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
super-front   1/1     1            1           4m28s
```



**查看对应的pod**

```
# kubectl get po -n pg-allen -l app=super-front
NAME                           READY   STATUS              RESTARTS   AGE
super-front-5788dc997d-k8prp   0/1     ContainerCreating   0          10s
```



**编辑修改**

```
# kubectl edit deploy -n pg-allen
# 按下回车后，将打开vi，可编辑deployment配置
```

在vi中保存并退出后，kubectl会检查配置，若配置无误将立即更新Deployment。

若Deployment的``spec.strategy``配置的升级策略为``rollingUpdate``，那么deployment更新后，pod将自动升级。

尝试将``image``改为``nginx:1.16.1-alpine``。

查看修改后Deployment的image：

```
# kubectl get deploy super-front -n pg-allen  -oyaml |grep image:
        image: nginx:1.16.1-alpine
```

查看当前Deployment的``revision``：

```
# kubectl get deploy super-front -n pg-allen  -oyaml  |grep revision
    deployment.kubernetes.io/revision: "2"
  revisionHistoryLimit: 10
  ... ...
```

可看出修改image后，revision为2。



**查看Deployment历史记录**

```
# kubectl rollout history deployment/super-front -n pg-allen
deployment.extensions/super-front 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```



**回滚Deployment**

``kubectl rollout undo``命令可将deployment回滚到某次revision：

```
# kubectl rollout undo deployment/super-front -n pg-allen --to-revision=1
deployment.extensions/super-front rolled back
```

查看Deployment回滚后的image：

```
# kubectl get deploy super-front -n pg-allen  -oyaml |grep image:
        image: nginx:latest
```



**查看事件**

```
# kubectl describe deploy super-front -n pg-allen 
Name:                   super-front
Namespace:              pg-allen
CreationTimestamp:      Wed, 08 Apr 2020 10:53:40 +0800
Labels:                 app=super-front
                        codename=k8s-playground
                        project=k8s-playground
Annotations:            deployment.kubernetes.io/revision: 2
Selector:               app=super-front
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        30
RollingUpdateStrategy:  0 max unavailable, 1 max surge
Pod Template:
  Labels:           app=super-front
                    codename=k8s-playground
                    project=k8s-playground
  Annotations:      cluster-autoscaler.kubernetes.io/safe-to-evict: true
  Service Account:  default
  Containers:
   super-front:
    Image:      nginx:latest
    Port:       <none>
    Host Port:  <none>
    Limits:
      cpu:     10m
      memory:  64Mi
    Requests:
      cpu:      10m
      memory:   64Mi
    Liveness:   tcp-socket :80 delay=168s timeout=1s period=20s #success=1 #failure=3
    Readiness:  http-get http://:80/ delay=11s timeout=1s period=20s #success=1 #failure=3
    Environment:
      LANG:      en_US.UTF-8
      LANGUAGE:  en_US:en
      LC_ALL:    en_US.UTF-8
    Mounts:      <none>
  Volumes:       <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   super-front-5b9db6d78c (1/1 replicas created)
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  5m17s  deployment-controller  Scaled up replica set super-front-75cb9b56fc to 1
  Normal  ScalingReplicaSet  4m28s  deployment-controller  Scaled up replica set super-front-5b9db6d78c to 1
  Normal  ScalingReplicaSet  3m38s  deployment-controller  Scaled down replica set super-front-75cb9b56fc to 0
```



**导出Deployment配置**

``kubectl get deploy``添加``--export``后，可导出工作负载的配置：

```
# kubectl get deploy super-front -n pg-allen -oyaml --export  > deploy.yml
```



**删除Deployment**

```
# kubectl delete deploy super-front -n pg-allen 
deployment.extensions "super-front" deleted
```



另一种删除方式：

```
# kubectl delete -f ./deploy.yml
deployment.extensions "super-front" deleted
```

PS：当要清除/停用某个App时，只删除pod是不行的，必须删除产生这个pod的工作负载。



扩展阅读：

* Deployments：https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
