# Kubernetes

## 什么是 Kubernetes

Kubernetes 是一个用于容器集群的自动化部署、扩容以及运维的开源平台。

![Kubernetes](images/kubernetes.png)

以前，我们需要手动控制服务器的数目，处理各种各样的问题。现在，Kubernetes 的出现为我们的维护提供了方便。当我们使用 Kubernetes API (kubectl) 告诉了 Kubernetes 我们所需要的状态，Kubernetes 会自动执行各种任务达到并保持我们所需要的状态。

<!-- 当你使用 Kubernetes API 创建对象设置了所需的状态，Kubernetes ⾃动执⾏各种任务 -- 例如启动或重新启动容器、伸缩给定应⽤程序的副本数量等等，最终将会使集群的当前状态匹配所需的状态。 -->

Kubernetes 中的大部分概念如 node、pod、replication controller、service 等都可以看作一种“资源对象”，几乎所有的资源对象都可以通过 Kubernetes 提供的 kubectl 工具（或者API编程调用）执行增、删、改、查等操作并将其保存在 etcd 中持久化存储。从这个角度来看，Kubernetes 其实是一个高度自动化的资源控制系统，它通过跟踪对比 etcd 库里保存的“资源期望状态”与当前环境中的“实际资源状态”的差异来实现自动控制和自动纠错的高级功能。

<!-- ## Kubernetes Control Plane

Kubernetes Control Plane 由运⾏在集群上的一系列进程组成：

* 运行在 Master 上的有 kube-apiserver、kube-controller-manager 和 kube-scheduler
* 运行在 Nodes 上的有 kubelet、kube-proxy

## Kubernetes 对象

Kubernetes 的基本对象包括：

* Pod
* Service
* Volume
* Namespace

Kubernetes 还包含⼀些称为控制器的更⾼层次的抽象概念。控制器基于基本对象，并提供附加特性和便利特性。包括：

* ReplicaSet
* Deployment
* StatefulSet
* DaemonSet
* Job -->
