---
layout: post
title:  "kubeflow介绍"
date:   2019-04-12 12:12:13 +0800
categories: Kubeflow
---

本文主要介绍kubeflow解决的问题、简单的使用、原理。

## 1. 深度学习的革命性进展

深度学习近年在众多领域取得令人瞩目的成就：

* 人机对弈（代表成果：AlphaGo击败人类围棋冠军、[AlphaStar击败星际2职业选手](https://deepmind.com/blog/alphastar-mastering-real-time-strategy-game-starcraft-ii/)）

* 计算机视觉（代表成果：ImageNet图片分类问题上，传统机器学习算法识别错误率为26%， 深度学习技术将错误率降低到3%，甚至超越人工标注的错误率5%）
* 语音识别（代表成果：苹果Siri、谷歌语音识别等）
* 自然语言处理（代表成果：Google Translate）

值得注意的是，深度学习在精准广告领域也备受青睐，如百度网盟（采用百度自研[PaddlePaddle](https://github.com/PaddlePaddle/Paddle)深度学习框架）、淘宝网盟（采用阿里妈妈自研的[x-deeplearning](https://github.com/alibaba/x-deeplearning)）。



## 2. 深度学习分布式训练的痛点

面对海量的训练数据，单台机器难以满足需求。为了提升训练速度，需要将tensorflow分布式的运行在多台机器上。大部分深度学习框架本身支持分布式训练，但实践中存在一些问题。

1）配置、部署麻烦。以分布式tensorflow为例，tf集群中每个job的配置各不相同，每当集群配置发生变化时（如机器扩缩容），就要将所有机器的配置改动一遍。因此，如何方便的进行配置必须解决。

2）在基础设施层面的运维难度高。开始训练前如何快速启动大量机器？训练过程中，竞价实例被回收，导致某个Job运行被中止了，怎么办？训练完成如何快速释放大量机器？



## 3. kubeflow

### 3.1 kubeflow：机器学习平台 ON Kubernetes

kubeflow的最终目标：借助k8s，高效的完成基础设施层面的工作，解决机器学习工程化落地的问题，让AI工程师/科学家专注于算法和模型。

kubeflow的定位是“机器学习平台 ON Kubernetes”。主要关注如下问题：

- 便捷的发布部署方式（以容器镜像方式）
- 便捷的参数配置方式（以环境变量方式将配置提供给job）
- 资源管理（支持CPU、内存、GPU的调度）

Kubeflow并不仅局限于Tensorflow，同时支持多种深度学习引擎：

- [Tensorflow](https://www.tensorflow.org/)（对应TFJob，[tf-operator](https://github.com/kubeflow/tf-operator)）
- [PyTorch](https://pytorch.org/)（对应PyTorchJob，[pytorch-operator](https://github.com/kubeflow/pytorch-operator)）
- [MXNet](https://mxnet.apache.org/)（对应MXJob，[mxnet-operator](https://github.com/kubeflow/mxnet-operator)）
- [MPI](https://www.open-mpi.org/)（对应MPIJob，[mpi-operator](https://github.com/kubeflow/mpi-operator)）
- [Chainer](https://chainer.org/)（对应ChainerJob，[chainer-operator](https://github.com/kubeflow/chainer-operator)）
- [Caffe2](https://caffe2.ai/)（对应Caffe2Job，[caffe2-operator](https://github.com/kubeflow/caffe2-operator)）

其中，Tensorflow是默认提供支持，其他引擎需在k8s上部署相应的operator。



### 3.2 kubeflow：使用&实现原理简析

#### 3.2.1 使用简介

**1）基本部署**

kubeflow是无侵入的基础设施，用户写好分布式代码，无需任何改动就能运行在kubeflow之上。

以tensorflow为例，工程师编写好分布式tf代码后，首先要完成源码编译，并构建成镜像（可通过CI/CD自动化实现）。然后只需在k8s上创建一个TFJob，即可完成tensorflow程序的部署。

以下是一个TFJob yaml实例：

```yaml
apiVersion: kubeflow.org/v1alpha2
kind: TFJob
metadata:
  labels:
    experiment: experiment10
  name: tfjob
  namespace: kubeflow
spec:
  tfReplicaSpecs:
    Ps:
      replicas: 1
      template:
        metadata:
          creationTimestamp: null
        spec:
          containers:
          - args:
            - python
            - tf_cnn_benchmarks.py
            - --batch_size=32
            - --model=resnet50
            - --variable_update=parameter_server
            - --flush_stdout=true
            - --num_gpus=1
            - --local_parameter_device=cpu
            - --device=cpu
            - --data_format=NHWC
            image: gcr.io/kubeflow/tf-benchmarks-cpu:v20171202-bdab599-dirty-284af3
            name: tensorflow
            ports:
            - containerPort: 2222
              name: tfjob-port
            resources: {}
            workingDir: /opt/tf-benchmarks/scripts/tf_cnn_benchmarks
          restartPolicy: OnFailure
    Worker:
      replicas: 1
      template:
        metadata:
          creationTimestamp: null
        spec:
          containers:
          - args:
            - python
            - tf_cnn_benchmarks.py
            - --batch_size=32
            - --model=resnet50
            - --variable_update=parameter_server
            - --flush_stdout=true
            - --num_gpus=1
            - --local_parameter_device=cpu
            - --device=cpu
            - --data_format=NHWC
            image: gcr.io/kubeflow/tf-benchmarks-cpu:v20171202-bdab599-dirty-284af3
            name: tensorflow
            ports:
            - containerPort: 2222
              name: tfjob-port
            resources: {}
            workingDir: /opt/tf-benchmarks/scripts/tf_cnn_benchmarks
          restartPolicy: OnFailure
```



``TFJob``配置主要包含Ps和Worker两部分，这两部分的配置与k8s Pod配置基本相同。

镜像 gcr.io/kubeflow/tf-benchmarks-cpu:v20171202-bdab599-dirty-284af3 的源码见 https://github.com/tensorflow/benchmarks 。



``TFJob``创建完毕后，``tf-operator``将根据``TFJob``配置，自动完成容器配置部署。

```bash
# kubectl get tfjob -n kubeflow 
NAME      AGE
tfjob     3m

# kubectl get po -n kubeflow -o wide
tfjob-ps-0                                                1/1       Running   0          8m        172.20.235.158   172.30.10.124
tfjob-worker-0                                            1/1       Running   0          8m        172.20.253.19    172.30.10.123
```



**2）数据存储**

问题1：训练数据从哪里来？

- 方法1：训练前，由worker自行下载（从S3、OSS等）
- 方法2：通过k8s Persistent Volume方式挂载外部文件系统（OSS、EBS、NFS等）到容器里面，worker程序即可像读取本地文件一样读取到训练数据。

问题2：训练结果保存到哪里？

- 方法1：训练完毕，由ps自行上传（到S3、OSS等）
- 方法2：通过k8s Persistent Volume方式挂载外部文件系统（OSS、EBS、NFS等）到容器里面，ps程序即可像写本地文件一样将模型数据写到外部文件系统。



**3）适合算法工程师的工具**

通过``kubectl``或k8s api操控``TFJob``的方式，要求用户对k8s有较深的了解。

对于算法工程师，可选用其他经过进一步包装的工具，如阿里云开源出来的[arena](https://github.com/kubeflow/arena)，屏蔽了k8s和kubeflow层面的细节，适合算法工程师使用。



#### 3.2.2 实现原理简介

**1）背后机制原理**

kubeflow是借助Kubernetes的[operator](https://coreos.com/operators/)机制实现。

以TFJob为例。TFJob是一个k8s [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)。除了k8s提供的Pod、Deployment、StatefulSet等标准类型的资源外，k8s允许自定义资源。资源的定义称为Custom Resource Definition，具体资源则称为Custom Resources。TFJob背后的逻辑实现是由tf-operator负责。tf-operator是一个operator，封装了与tensorflow相关的领域逻辑。tf-operator一直暗中观察TFJob，当TFJob发生变化时（CRUD等操作），tf-operator默默进行具体的处理（如容器部署配置等）。

**2）对GPU的支持**

对GPU的支持是通过[nvidia-docker](https://github.com/NVIDIA/nvidia-docker)和[kubelet device plugin](https://github.com/NVIDIA/k8s-device-plugin)实现。



### 3.3 kubeflow与k8s集群自动扩缩容机制

在实际使用中，kubeflow可以结合k8s的集群自动扩缩容机制，起到提高效率、节省成本的效果。

流程说明如下：

1）工程师通过工具在k8s上创建TFJob。

2）tf-operator观察到TFJob，根据TFJob的配置自动创建若干worker和ps容器。

3）集群机器资源不够，容器处于Pending状态。

4）k8s [cluster-autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)观察到容器因资源不够而Pending后，启动自动扩容，根据预先配置创建新机器并加到集群中。

5）k8s scheduler将容器调度到新扩容出来的机器。

6）容器在新机器上启动—》tensorflow程序运行—》运行完毕—》容器自动退出。

7）k8s cluster-autoscaler观察到机器资源占用率降低到特定阈值，启动自动缩容，删除富余的机器。直到tensorflow所有worker均运行完毕，机器也全部被回收。



这种模式有如下的优点：

1）全程自动化，响应迅速。需要使用机器时，自动扩容出所需的机器。这比运维人员手动操作快，并且减轻了运维的工作负担。

2）随用随启，用完释放，节省成本。模型训练完毕后，机器自动释放。不会出现程序跑完后忘记关机而造成浪费的情况。

3）扩缩容的机器可以采用较便宜的竞价实例。若竞价实例在使用过程被回收，则Job会自动被调度到其他机器。



### 3.4 kubeflow的其他组件

kubeflow还集成一些有趣或强大的组件，简单列出如下：

|功能组件|作用|说明|
|----|----|----|
|TF Hub|机器学习在线开发/实验环境|kubeflow版的JupyterHub。借助TF Hub，开发人员可在浏览器中开发调试tensorflow程序。TF Hub的数据以k8s 持久卷的形式保存在外部文件系统中，工作产出可长期保存。另外，TF Hub可创建多个工作空间，每个工作空间可用不同的容器镜像，因此非常便于切换开发/运行环境。|
|[Katib](https://github.com/kubeflow/katib)|超参数（Hyper-parameter）自动化训练|超参数的黑盒优化系统，可用于优化神经网络的层数、激活函数、学习率等超参数|
|模型服务|模型线上服务和访问|封装了[TensorFlow Serving](https://github.com/tensorflow/serving)，线上服务部署和预估服务|
|TensorBoard|深度学习可视化|帮助AI工程师/科学家分析训练效果，理解训练框架和优化算法|



### 3.5 kubeflow in production：coming soon

目前（2019-04-11）kubeflow刚发布0.5.0版，[官方计划在2019上半年发布1.0版](https://medium.com/kubeflow/kubeflow-in-2018-a-year-in-perspective-49c273b490f4)，但有可能会跳票。

