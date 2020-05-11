---
layout: post
title:  "成为Kubernetes运维开发（4）：认识工作负载Statefulset"
date:   2020-05-11 04:14:13 +0800
categories: Kubernetes
---


除了无状态应用外，实践中还存在另一种不那么常见的应用：有状态应用。有状态应用通常有如下特点：

* 需要固定的网络标识符。例如，Redis集群里面的多个Redis实例，相互之间需要固定的IP或hostname等网络标识进行访问。
* 需要稳定的存储。例如，Zookeeper集群中各个实例，需要单独且固定的存储。

有状态应用对应K8s工作负载是Statefulset。对于Statefulset，“**固定的网络标识符**”是通过Headless Service和集群DNS机制实现；而“**稳定的存储**”是通过PV/PVC/VolumeClaimTemplates等机制实现。



### 1. Headless Service

Headless Service是一种特殊的K8s Service。从逻辑上，普通Service相当于反向代理，发给普通Service的流量会转发给Service后端的pod；而Headless Service仅提供类似DNS的功能，返回后端pod IP。

在YAML配置上，Headless Service与普通Service的区别在于，Headless Service的spec.clusterIP必须设置为``None``。

尝试创建如下Headless Service：

```
apiVersion: v1
kind: Service
metadata:
  annotations:
  labels:
    app: eureka
  name: eureka-headless
  namespace: pg-allen-ali   #【修改为各自namespace】
spec:
  clusterIP: None
  ports:
  - port: 8761
    protocol: TCP
    targetPort: 8761
  selector:
    app: eureka
  sessionAffinity: None
  type: ClusterIP
```

查看Headless Service信息：

```
# kubectl get svc eureka-headless -n pg-allen-ali 
NAME              TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
eureka-headless   ClusterIP   None         <none>        8761/TCP   150m
```



### 2. Statefulset

创建如下statefulset

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app: eureka
  name: eureka
  namespace: pg-allen-ali   #【修改为各自namespace】
spec:
  podManagementPolicy: Parallel
  replicas: 2
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: eureka
  serviceName: eureka-headless
  template:
    metadata:
      labels:
        app: eureka
        project: k8s-playground
    spec:
      containers:
      - env:
        - name: EUREKA_CLIENT_REGISTERWITHEUREKA
          value: "true"
        - name: EUREKA_CLIENT_FETCHREGISTRY
          value: "true"
        - name: replicas
          value: "2"
        - name: EUREKA_NAMESPACE
          value: pg-allen-ali    # 【修改为各自namespace】
        image: registry-vpc.ap-southeast-1.aliyuncs.com/diangao-ym/proj.k8s-playground_app.eureka:20191227_01
        imagePullPolicy: Always
        livenessProbe:
          failureThreshold: 3
          initialDelaySeconds: 168
          periodSeconds: 20
          successThreshold: 1
          tcpSocket:
            port: 8761
          timeoutSeconds: 1
        name: eureka
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /actuator/health
            port: 8761
            scheme: HTTP
          initialDelaySeconds: 11
          periodSeconds: 20
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          limits:
            cpu: 500m
            memory: 768Mi
          requests:
            cpu: 100m
            memory: 512Mi
        volumeMounts:
        - mountPath: /dianyi/data
          name: eureka-data
      dnsPolicy: ClusterFirst
      restartPolicy: Always
  updateStrategy:
    rollingUpdate:
      partition: 0
    type: RollingUpdate
  volumeClaimTemplates:
  - metadata:
      name: eureka-data
      namespace: pg-allen-ali
    spec:
      accessModes:
      - ReadWriteOnce
      - ReadOnlyMany
      - ReadWriteMany
      storageClassName: alicloud-nas
      resources:
        requests:
          storage: 1Gi
      volumeMode: Filesystem
```

若statefulset的pod未能正常启动，则可查看statefulset状态：

```
# kubectl -n pg-allen-ali describe statefulset eureka
```

查看pod状态：

```
# kubectl -n pg-allen-ali get po
```



**查看PVC**

```
# kubectl -n pg-allen-ali get pvc
```

PVC状态是否为``Bound``状态？



**查看PV**

```
# kubectl get pv
```

PV状态是否为``Bound``状态？



为了在集群外访问eureka服务，我们可以再搞一个普通Service：

```
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.beta.kubernetes.io/alibaba-cloud-loadbalancer-additional-resource-tags: CodeName=whale
    service.beta.kubernetes.io/alicloud-loadbalancer-address-type: intranet
  labels:
    app: eureka
  name: eureka
  namespace: pg-allen-ali   #【改成各自Service】
spec:
  externalTrafficPolicy: Cluster
  ports:
  - port: 8761
    protocol: TCP
    targetPort: 8761
  selector:
    app: eureka
  sessionAffinity: None
  type: LoadBalancer
```

稍等片刻，即可通过SLB地址访问Eureka。



进入``eureka-0``这个pod：

```
# kubectl -n pg-allen-ali exec -it eureka-0 bash
# 查看挂载的volume
# df -h 
Filesystem                                                                                                                    Size  Used Avail Use% Mounted on
7fcde4a719-bsa48.ap-southeast-1.nas.aliyuncs.com:/pg-allen-ali-eureka-data-eureka-0-pvc-6c4b31b9-7fa3-11ea-b7ec-00163e02d9d6   10P  166M   10P   1% /dianyi/data
... ...
# 往数据盘写个测试信息
# echo "in eureka-0" > /dianyi/data/data_test
# cat /dianyi/data/data_test 
in eureka-0

# 尝试通过headless service访问另一个eureka实例
# ping eureka-1.eureka-headless
PING eureka-1.eureka-headless.pg-allen-ali.svc.cluster.local (172.20.27.72) 56(84) bytes of data.
64 bytes from 172-20-27-72.eureka.pg-allen-ali.svc.cluster.local (172.20.27.72): icmp_seq=1 ttl=62 time=1.03 ms
64 bytes from 172-20-27-72.eureka.pg-allen-ali.svc.cluster.local (172.20.27.72): icmp_seq=2 ttl=62 time=0.983 ms
^C
```

如有时间，还可进入``eureka-1``中执行类似的操作。



