---
layout: post
title:  "容器内访问k8s api-server"
date:   2018-06-26 17:12:13 +0800
categories: Kubernetes
---


一般的应用通常并不需要访问k8s API。但如果你需要开发用于k8s集群管理的应用，而且需要以容器方式运行在k8s集群中，那么就有必要了解在pod中访问k8s API的方法。
本文介绍在pod内访问k8s api-server的两种方式：curl方式，client-go方式。对于go语言编写的app，推荐直接使用client-go访问。


### 1. 在pod内，通过curl访问k8s api-server

pod启动之后，会将访问k8s api-server所需的文件保存在pod的``/var/run/secrets/kubernetes.io/serviceaccount``目录下。

```
# ll /var/run/secrets/kubernetes.io/serviceaccount    
total 4
drwxrwxrwt 3 root root  140 Jun 22 11:25 ./
drwxr-xr-x 3 root root 4096 Jun 22 11:26 ../
drwxr-xr-x 2 root root  100 Jun 22 11:25 ..2018_06_22_11_25_56.296581011/
lrwxrwxrwx 1 root root   13 Jun 22 11:25 ca.crt -> ..data/ca.crt
lrwxrwxrwx 1 root root   31 Jun 22 11:25 ..data -> ..2018_06_22_11_25_56.296581011/
lrwxrwxrwx 1 root root   16 Jun 22 11:25 namespace -> ..data/namespace
lrwxrwxrwx 1 root root   12 Jun 22 11:25 token -> ..data/token
```



通过CURL访问k8s api-server时，需携上``ca.crt``和``token``信息，示例如下：

```
TOKEN_VALUE=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
NAMESPACE=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)
curl -k --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt -H  "Authorization: Bearer $TOKEN_VALUE" https://kubernetes.default/apis/apps/v1/namespaces/$NAMESPACE/daemonsets/gateway?resourceVersion=0
```

响应示例：

```
{
  "kind": "DaemonSet",
  "apiVersion": "apps/v1",
  "metadata": {
    "name": "gateway",
    "namespace": "test",
    "selfLink": "/apis/apps/v1/namespaces/test/daemonsets/gateway",
    "uid": "5ac6df67-786a-11e8-a355-0050568156a5",
    "resourceVersion": "6342312",
    "generation": 3,
    "creationTimestamp": "2018-06-25T11:24:41Z",
    "labels": {
      "k8s-app": "gateway"
    },
    "annotations": {
      "kubectl.kubernetes.io/last-applied-configuration": "{\"apiVersion\":\"apps/v1\",\"kind\":\"DaemonSet\",\"metadata\":{\"annotations\":{},\"labels\":{\"k8s-app\":\"gateway\"},\"name\":\"gateway\",\"namespace\":\"test\"},\"spec\":{\"selector\":{\"matchLabels\":{\"name\":\"gateway\"}},\"template\":{\"metadata\":{\"labels\":{\"name\":\"gateway\"}},\"spec\":{\"containers\":[{\"image\":\"172.30.10.185:15000/common/ubuntu:14.04\",\"name\":\"gateway\",\"resources\":{\"limits\":{\"memory\":\"200Mi\"},\"requests\":{\"cpu\":\"100m\",\"memory\":\"200Mi\"}}}]}}}}\n",
      "deprecated.daemonset.template.generation": "3"
    }
  },
  "spec": {
    "selector": {
      "matchLabels": {
        "name": "gateway"
      }
    },
    "template": {
      "metadata": {
        "creationTimestamp": null,
        "labels": {
          "name": "gateway"
        }
      },
      "spec": {
        "containers": [
          {
            "name": "gateway",
            "image": "172.30.10.185:15000/common/ubuntu:14.04",
            "resources": {
              "limits": {
                "memory": "200Mi"
              },
              "requests": {
                "cpu": "100m",
                "memory": "200Mi"
              }
            },
            "terminationMessagePath": "/dev/termination-log",
            "terminationMessagePolicy": "File",
            "imagePullPolicy": "Always"
          }
        ],
        "restartPolicy": "Always",
        "terminationGracePeriodSeconds": 30,
        "dnsPolicy": "ClusterFirst",
        "securityContext": {
          
        },
        "schedulerName": "default-scheduler"
      }
    },
    "updateStrategy": {
      "type": "RollingUpdate",
      "rollingUpdate": {
        "maxUnavailable": 1
      }
    },
    "revisionHistoryLimit": 10
  },
  "status": {
    "currentNumberScheduled": 20,
    "numberMisscheduled": 0,
    "desiredNumberScheduled": 20,
    "numberReady": 19,
    "observedGeneration": 3,
    "updatedNumberScheduled": 1,
    "numberAvailable": 19,
    "numberUnavailable": 1
  }
}
```


### 2. 在pod内，通过client-go访问k8s api-server

client-go官方提供了详细的[安装说明](https://github.com/kubernetes/client-go/blob/master/INSTALL.md)，以及[示例代码](https://github.com/kubernetes/client-go/tree/master/examples/in-cluster-client-configuration)。
此处提供[一个完整的示例](https://github.com/moshucong/client-go-example)

参考资料：
1. [Access Clusters Using the Kubernetes API](https://kubernetes.io/docs/tasks/administer-cluster/access-cluster-api/)

