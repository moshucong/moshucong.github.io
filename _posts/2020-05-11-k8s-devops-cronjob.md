---
layout: post
title:  "成为Kubernetes运维开发（6）：认识工作负载Cronjob"
date:   2020-05-11 01:14:13 +0800
categories: Kubernetes
---



### 1. Cronjob基础

创建Cronjob：

```
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  creationTimestamp: null
  generateName: reload-prometheus
  namespace: 【pg-allen-job】
  name: reload-prometheus
  labels:
    app: reload-prometheus
spec:
  concurrencyPolicy: Replace
  failedJobsHistoryLimit: 2
  schedule: '*/3 * * * *'
  startingDeadlineSeconds: 60
  successfulJobsHistoryLimit: 5
  suspend: false
  jobTemplate:
    spec:
      backoffLimit: 3
      template:
        metadata:
          labels:
            app: reload-prometheus
        spec:
          containers:
          - command:
            - curl
            args:
            - -X
            - PUT
            - --connect-timeout
            - "10"
            - -m
            - "60"
            - https://gears.yeahmobi.com/paas/ops/reload-prometheus
            image: 172.30.10.185:15000/common/curl-alpine:3.8
            imagePullPolicy: IfNotPresent
            name: reload-prometheus
            resources:
              limits:
                cpu: 800m
                memory: 100Mi
              requests:
                cpu: 100m
                memory: 10Mi
          dnsPolicy: Default
          hostNetwork: true
          restartPolicy: Never
```



最常用的两个配置：

- schedule：定时任务的crontab时间配置，按 <minute> <hour> <day of month> <month> <day of week>格式填写。
- command：定时执行的命令。需要确保命令在容器内能够正常执行。

其他配置项大部分情况保持默认值即可。这些配置项的含义简单说明如下：

- concurrency policy：并发策略。默认值Replace。
- suspend：若job未执行完毕，是否挂起后续job。默认值false。
- completions：表示job完成所需的成功执行的容器数量。例如，completions为1，表示有1个容器成功，则认为本次job执行完毕。默认值1。
- parallelism：并发执行的个数。默认值1。
- startingDeadlineSeconds：job启动超时时间。
- activeDeadlineSeconds：执行超时时间。
- restartPolicy：重启策略。
- backoffLimit：重启次数限制，重启达到次数后，则不再重启。
- failedJobsHistoryLimit：保留的失败记录数。为了减少资源消耗，不建议配置过大。
- successfulJobsHistoryLimit：保留的成功记录数。为了减少资源消耗，不建议配置过大。
- envs：环境变量。key-value形式配置，容器的全局环境变量。
- volumes：文件卷挂载。
- dns policy：容器的dns策略。



等待数分钟，即可查看状态：

```
# kubectl get cronjob -n pg-allen-job
NAME                SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
reload-prometheus   */3 * * * *   False     0        39s             7m13s

# kubectl get job -n pg-allen-job
NAME                           COMPLETIONS   DURATION   AGE
reload-prometheus-1586920500   1/1           10s        40s

# kubectl get po -n pg-allen-job
NAME                                 READY   STATUS      RESTARTS   AGE
reload-prometheus-1586920500-2qg8b   0/1     Completed   0          48s

# kubectl logs -f reload-prometheus-1586920500-2qg8b -n pg-allen-job
... ...
```



研究完毕。删除创建的cronjob：

```
# kubectl delete cronjob reload-prometheus -n pg-allen-job --cascade=true
cronjob.batch "reload-prometheus" deleted
```

删除cronjob时，若添加``--cascade=true``，则会级联删除job和pod。



扩展阅读：

* Gears Cronjob使用说明：https://confluence.yeahmobi.com/pages/viewpage.action?pageId=37159409
* CronJob：https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/
* Running Automated Tasks with a CronJob：https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/



