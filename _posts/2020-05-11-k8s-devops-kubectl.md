---
layout: post
title:  "成为Kubernetes运维开发（1）：配置好用的kubectl"
date:   2020-05-11 01:14:13 +0800
categories: data
---


## 1.1 配置好用的kubectl

Kubernetes通过Kube-apiserver对外提供API服务。常用的访问Kube-apiserver的方式有：

* 通过调用RESTful API。例如，通过``curl``命令或客户端SDK。RESTful API是这是应用程序访问Kubernetes API的常用方式。
* 通过kubectl命令工具。kubectl是Kubernetes管理员必须掌握的命令工具。kubectl本质上是将用户的命令转为RESTful API调用。

一个安全完备的Kubernetes集群，必须启用[Kubernetes的认证-授权机制](https://kubernetes.io/docs/reference/access-authn-authz/controlling-access/)。无论通过RESTful API或kubectl访问Kubernetes API，都需要如下证书信息：

- 证书管理机构公钥证书（certificate authority certificate），文件名通常是ca.pem。
- 用户公钥证书。
- 用户秘钥证书。

访问Kubernetes API前，我们必须获取到以上三个证书。



![kubectl](images/kubectl.png)



### 1.1.1 kubectl的kubeconfig配置

随着容器技术的广泛使用，各大公司通常搭建并维护着数量众多的容器集群，包括生产、测试等不同类型，不同区域的集群。因此运维人员通常同时管理着大量的集群。

最简单的管理办法是登录各个集群的master机器执行kubectl命令。但是当集群数量很多时，这种做法的操作效率就很低了。那么有没有办法，能让运维人员本地工作机器上直接操作所有集群呢？答案是有的，而且通过原生的kubectl就可以实现。

下载``kubectl``

```
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
```

首先，我们从各个集群master机器上``~/.kube/config``目录下获取kubectl的配置。

假如我们有两个集群的config文件。集群cluster-1的config文件如下：

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: [经过base64编码的ca.pem]
    server: https://[cluster-1 api-server地址]:6443
  name: cluster-1             ## 集群名
contexts:
- context:
    cluster: cluster-1        ## 集群名，对应上面clusters[].name
    user: admin               ## 集群的认证信息，对应下面users[].name
  name: cluster-1             ## context名
current-context: cluster-1

kind: Config
preferences: {}
users:
- name: admin                 ## 集群的认证信息
  user:
    client-certificate: /var/lib/kubernetes/admin.pem   #
    client-key: /var/lib/kubernetes/admin-key.pem   #
```



另一个集群cluster-2的config文件如下：

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: [经过base64编码的ca.pem]
    server: https://[cluster-2 api-server地址]:6443
  name: cluster-2        
contexts:
- context:
    cluster: cluster-2   
    user: admin          
  name: cluster-2       
current-context: cluster-2

kind: Config
preferences: {}
users:
- name: admin           
  user:
    as-user-extra: {}
    client-certificate: /var/lib/kubernetes/admin.pem
    client-key: /var/lib/kubernetes/admin-key.pem
```

还有一种配置方式，是直接通过``client-certificate-data``和``client-key-data``直接指定Base64编码的用户证书和秘钥信息。

```
apiVersion: v1
clusters:
- cluster:
    server: https://[集群api-server地址]:6443
    certificate-authority-data: # base64编码的ca公钥
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: "admin"
  name: cluster-1
current-context: clusetr-1
kind: Config
preferences: {}
users:
- name: "admin"
  user:
    client-certificate-data: # base64编码的公钥证书
    client-key-data: # base64编码的key
```



然后，我们需要合并这两个config文件：

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: [经过base64编码的ca.pem]
    server: https://[cluster-1 api-server地址]:6443
  name: cluster-1
- cluster:
    certificate-authority-data: [经过base64编码的ca.pem]
    server: https://[cluster-2 api-server地址]:6443
  name: cluster-2
contexts:
- context:
    cluster: cluster-1
    user: admin-cluster-1
  name: cluster-1
- context:
    cluster: cluster-2
    user: admin-cluster-2
  name: cluster-2
current-context: cluster-1
kind: Config
preferences: {}
users:
- name: admin-cluster-1
  user:
    client-certificate: /root/.kube/cluster-1/admin.pem
    client-key: /root/.kube/cluster-1/admin-key.pem
- name: admin-cluster-2
  user:
    client-certificate: /root/.kube/cluster-2/admin.pem
    client-key: /root/.kube/cluster-2/admin-key.pem
```

合并过程需要区分不同集群的``user``名字。

最后，将合并后的config文件放到我们本地工作站的``~/.kube/``目录下，并分别将两个集群的admin证书保存在``~/.kube/cluster-1/``和``~/.kube/cluster-2/``目录下。



至此，我们的工作站上就可通过``kubectl config use-context``命令切换到各个集群context，并在不同的集群执行命令了：

```
## 查看context列表
# kubectl config get-contexts
CURRENT   NAME       CLUSTER    AUTHINFO         NAMESPACE
          cluster-1   cluster-1   admin-cluster-1   
*         cluster-2   cluster-2   admin-cluster-2  

## 切换context到 cluster-1
# kubectl config use-context cluster-1
# kubectl get nodes
172.30.10.155   Ready    <none>   153d   v1.9.8
# 
## 切换到cluster-2
# kubectl config use-context cluster-2
# kubectl get nodes
172.30.10.188   Ready,SchedulingDisabled      master   404d   v1.9.8
```



### 1.1.2. 通过kubectx快速切换集群上下文

除了``kubectl config get-contexts``和``kubectl config use-context``，``kubectx``工具也可实现集群上下文的切换，且操作比kubectl更快捷。

首先，仍然要编辑好``kubeconfig``配置，具体步骤见上一节。

然后，下载kubectx工具：

```
sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubectx /usr/local/bin/kubectx
```

即可通过``kubectx``命令查看当前配置的集群上下文：

```
# kubectx 
xian-188
ali-zjk
```

可通过``kubectx [context名]``命令，切换集群上下文：

```
# kubectx xian-188
Switched to context "xian-188".
```



### 1.1.3 动手实践

实验步骤：

1）安装1个虚拟机或找一台服务器。操作系统可选择Centos或Ubuntu。

2）配置``kubeconfig``、``kubectl``、``kubectx``。

3）验证配置是否成功。通过``kubectx``命令切换到集群1，执行``kubectl get nodes``查看集群1节点列表；再切换到集群2，执行``kubectl get nodes``查看集群2节点列表。


