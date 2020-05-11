---
layout: post
title:  "成为Kubernetes运维开发（10）：Prometheus"
date:   2020-05-11 10:14:13 +0800
categories: Kubernetes
---


### 1. 原理

[prometheus](https://prometheus.io/)是当今云原生监控的事实标准。Prometheus监控的对象是Metrics（指标）。个人认为，与Zabbix等相比，Prometheus更适合当今盛行的微服务架构：

- Restful API方式拉取metrics。被监控的服务只需提供metrics API，处理非常的灵活。
- 支持服务发现。支持Kubernetes、consul等服务发现方式，Prometheus可对各个微服务实例分别进行metrics采集。

![prometheus architecture](images/prometheus architecture.png)



Prometheus相关说明：

* **AlertManager**，负责Prometheus与告警API的对接。AlertManager原生支持一些告警客户端（如slack、邮件等），同时提供Web hook方式，供程序员进行扩展。我司就是通过Web hook方式进行集成，具体可参考[Prometheus AlertManager Webhook API 说明](https://confluence.yeahmobi.com/pages/viewpage.action?pageId=38438355)。
* **PromQL**。PromQL是Prometheus**查询**语句，只提供数据查询功能。告警规则、数据展示的配置都要用到PromQL。
* **PushGateway**。除了常规的拉取式metrics采集外，PushGateway提供了推送式的metrics采集。



### 2. Metrics Exporter/API

Prometheus将以HTTP API方式导出（export）metrics的组件称为``Exporter``。社区提供大量现成的[第三方Exporter](https://prometheus.io/docs/instrumenting/exporters/)。我司Kubernetes应用监控使用了如下两个Exporter：

* [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)。
* [node_exporter](https://github.com/prometheus/node_exporter)。

我们可[开发自己的Metrics API](https://prometheus.io/docs/instrumenting/writing_exporters/)，只需HTTP API的响应符合Prometheus Metrics要求即可。更便捷的方式是，使用现成的SDK。例如，[prometheus的golang sdk](https://github.com/prometheus/client_golang)。



### 3. Prometheus告警配置

我司prometheus告警规则以ConfigMap方式配置：

```
# kubectl edit cm prometheus-rule-config  -n monitor-ali-sg-ym
```

在rule.yml下添加如下规则：

```
apiVersion: v1
data:
  rule.yml: |
    ---
    groups:
    - name: pg-allen-ali
      rules:
      - alert: PodRestart
        expr: changes(kube_pod_container_status_restarts_total{namespace="pg-allen-ali"}[8m])
          > 0
        labels:
          receiver_group: [receivers email/phone]
          service: '{{ $labels.pod }}'
          severity: WARN
        annotations:
          container: '{{ $labels.container }}'
          ns: '{{ $labels.namespace }}'
          pod: '{{ $labels.pod }}'
          summary: Pod Restart
        for: 1m
      - alert: PodWaiting
        expr: kube_pod_container_status_waiting_reason{namespace="pg-allen-ali"} == 1
        labels:
          receiver_group: [receivers email/phone]
          service: '{{ $labels.pod }}'
          severity: ERROR
        annotations:
          container: '{{ $labels.container }}'
          ns: '{{ $labels.namespace }}'
          pod: '{{ $labels.pod }}'
          summary: Pod Waiting
        for: 10m
      - alert: DeploymentUnavailable
        expr: kube_deployment_status_replicas_unavailable{namespace="pg-allen-ali"} >
          0
        labels:
          receiver_group: [receivers email/phone]
          service: '{{ $labels.deployment }}'
          severity: FATAL
        annotations:
          deploy: '{{ $labels.deployment }}'
          ns: '{{ $labels.namespace }}'
          summary: Deployment Unavailable
        for: 10m
... ...
```



修改完prometheus rule后，reload：

```
curl -X POST http://[prometheus service ip]:9090/-/reload
```



### 4. 通过Prometheus-operator快速完成部署配置

[prometheus-operator](https://github.com/coreos/prometheus-operator)关注的是，如何快速部署和配置一套prometheus监控告警。

使用prometheus-operator完整的完成监控告警，至少需配置如下3个核心资源：

* **Prometheus**，关于Prometheus的配置。
* **ServiceMonitor**，定义Prometheus监控service，包含服务发现和metrics采集所需信息。
* **PrometheusRule**，告警相关信息。

创建一个``Prometheus``资源：

```
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  generation: 1
  labels:
    namespace: pg-allen-ali   ### 【修改为自己的namespace】
  name: prometheus
spec:
  alerting:            ### 配置告警所用的alertManager
    alertmanagers:
    - name: alertmanager
      namespace: kube-system
      port: http
  enableAdminAPI: false
  nodeSelector: {}
  retention: 60d
  ruleSelector:
    matchLabels:
      namespace: pg-allen-ali         ### 【修改为自己的namespace】
  serviceAccountName: prometheus
  serviceMonitorSelector:   ### 匹配的serviceMonitor
    matchLabels:
      namespace: pg-allen-ali
  storage:
    volumeClaimTemplate:     ### 配置存储数据所用Volume
      apiVersion: v1
      kind: PersistentVolumeClaim
      metadata:
        name: prometheus-db
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 50Gi
        storageClassName: alicloud-nas
```

创建一个``ServiceMonitor``：

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    namespace: pg-allen-ali   ### 【修改为自己的namespace】
  name: super-backend
spec:
  endpoints:
  - interval: 30s
    path: /metrics
    port: pg-allen-ali-super-backend-0      ### 【修改为自己的service的port名字】
    scheme: ""
    scrapeTimeout: 3s
    targetPort: ""
  selector:              ### 匹配service所用selector
    matchLabels:
      app: super-backend
```

创建一个``PrometheusRule``：

```
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  labels:
    namespace: pg-allen-ali      ### 【修改为自己的namespace】
  name: prometheusrule
spec:
  groups:
  - name: pg-allen-ali         ### 【修改为自己的namespace名】
    rules:
    - alert: go-threads
      annotations:
        summary: go_threads>2
      expr: go_threads>2
      labels:
        receiver_group: allen.mo@yeahmobi.com        ### 【修改为自己的公司邮箱】
        service: pg-allen-ali      ### 【修改为自己的namespace】
        severity: ERROR
```

创建资源完毕后，可查看资源状态：

```
# kubectl get prometheus -n pg-allen-ali 
NAME         AGE
prometheus   27m
# kubectl get servicemonitor -n pg-allen-ali 
NAME            AGE
super-backend   27m
# kubectl get prometheusrule -n pg-allen-ali 
NAME             AGE
prometheusrule   27m
```

查看prometheus-operator控制的其他资源：

```
# kubectl get po -n pg-allen-ali  | grep prometheus
prometheus-prometheus-0          3/3     Running   1          30m
# kubectl get statefulset -n pg-allen-ali 
NAME                    READY   AGE
prometheus-prometheus   1/1     30m
```

等待一会（15分钟？），看看告警是否被触发？



