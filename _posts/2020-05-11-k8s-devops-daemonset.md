---
layout: post
title:  "成为Kubernetes运维开发（5）：认识工作负载Daemonset"
date:   2020-05-11 05:14:13 +0800
categories: Kubernetes
---

Daemonset类应用会在集群所有节点上各运行1个pod（除非有其他调度限制）。

因其特点，Daemonset适用于部署如下类型应用：

* 采集类Agent，包括Flume、Fluented等。
* 运维监控类，包括Pormetheus Node-Exporter等。
* K8s自身组件，kube-proxy、K8s虚拟网络（如flannel、calico-node）、Cloud-controller-manager等。



### 1. Daemonset基础

**创建Daemonset**

```
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  labels:
    app: node-exporter
    project: k8s-playground
  name: node-exporter
  namespace: pg-allen  #修改为自己环境
spec:
  minReadySeconds: 30
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      annotations:
        cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
      creationTimestamp: null
      labels:
        app: node-exporter
        project: k8s-playground
    spec:
      automountServiceAccountToken: false
      containers:
      - env:
        - name: web_listen_address
          value: "19100"    #修改为自己端口
        image: 172.30.10.185:15000/common/prometheus-node-exporter:2020041501
        imagePullPolicy: Always
        livenessProbe:
          failureThreshold: 3
          initialDelaySeconds: 168
          periodSeconds: 20
          successThreshold: 1
          tcpSocket:
            port: 19100   #修改为自己端口
          timeoutSeconds: 1
        name: node-exporter
        readinessProbe:
          failureThreshold: 3
          initialDelaySeconds: 11
          periodSeconds: 20
          successThreshold: 1
          tcpSocket:
            port: 19100   #修改为自己端口
          timeoutSeconds: 1
        resources:
          limits:
            cpu: 400m
            memory: 256Mi
          requests:
            cpu: 50m
            memory: 50Mi
        volumeMounts:
        - mountPath: /host/proc
          name: volume0
        - mountPath: /host/sys
          name: volume1
      dnsPolicy: Default
      hostNetwork: true
      restartPolicy: Always
      volumes:
      - hostPath:
          path: /proc
          type: ""
        name: volume0
      - hostPath:
          path: /sys
          type: ""
        name: volume1
  templateGeneration: 1
  updateStrategy:
    rollingUpdate:
      maxUnavailable: 10
    type: RollingUpdate
```



访问：

```
# curl http://172.30.10.122:19100/metrics
... ...
```



查看：

```
# kubectl describe ds node-exporter -n pg-allen
... ...

# kubectl get ds node-exporter -n pg-allen
NAME            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
node-exporter   55        55        54      55           54          <none>          2m48s

# kubectl get po -n pg-allen
... ...
```



删除daemonset：

```
# kubectl delete ds node-exporter -n pg-allen
daemonset.extensions "node-exporter" deleted
```



