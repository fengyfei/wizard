# Master and Node

## Master

Kubernetes 里的 Master 指的是集群控制节点，每个 Kubernetes 集群里需要有一个 Master 节点来负责整个集群的管理和控制，基本上 Kubernetes 所有的控制命令都是发给它，它来负责具体的执行过程，我们后面所有执行的命令基本都是在 Master 节点上运行的。Master 节点通常会占据一个独立的 X86 服务器（或者一个虚拟机)，一个主要的原因是它太重要了，它是整个集群的“首脑”，如果它宕机或者不可用，那么我们所有的控制命令都将失效。

Master节点上运行着以下一组关键进程。

* Kubernetes API Server (kube-apiserver)，提供了 HTTP Rest 接口的关键服务进程，是 Kubernetes 里所有资源的增、删、改、查等操作的唯一入口，也是集群控制的入口进程。
* Kubemetes Controller Manager (kube-controller-manager)，Kubemetes 里所有资源对象的自动化控制中心，可以理解为资源对象的“大总管”。
* Kubemetes Scheduler (kube-scheduler)，负责资源调度（Pod 调度）的进程，相当于公交公司的“调度室”。

其实 Master 节点上往往还启动了 一个 etcd Server 进程，因为 Kubemetes 里的所有资源对象的数据全部是保存在 etcd 中的。

## Node

除了 Master， Kubemetes 集群中的其他机器被称为 Node 节点，在较早的版本中也被称为 Minion。与 Master —样，Node 节点可以是一台物理主机，也可以是一台虚拟机。Node 节点才是Kubemetes集群中的工作负载节点，每个 Node 都会被 Master 分配一些工作负载（Docker 容器），当某个 Node 宕机时，其上的工作负载会被 Master 自动转移到其他节点上去。

每个Node节点上都运行着以下一组关键进程。

* kubelet: 负责 Pod 对应的容器的创建、启停等任务，同时与 Master 节点密切协作，实现集群管理的基本功能。
* kube-proxy: 实现 Kubemetes Service 的通信与负载均衡机制的重要组件。
* Docker Engine (docker): Docker 引擎，负责本机的容器创建和管理工作。

Node 节点可以在运行期间动态增加到 Kubemetes 集群中，前提是这个节点上已经正确安装、配置和启动了上述关键进程，在默认情况下 kubelet 会向 Master 注册自己，这也是 Kubemetes 推荐的 Node 管理方式。一旦 Node 被纳入集群管理范围， kubelet 进程就会定时向 Master 节点汇报自身的情报，例如操作系统、Docker 版本、机器的 CPU 和内存情况，以及之前有哪些 Pod 在运行等，这样 Master 可以获知每个 Node 的资源使用情况，并实现高效均衡的资源调度策略。 而某个 Node 超过指定时间不上报信息时，会被 Master 判定为“失联”，Node 的状态被标记为不可用（Not Ready），随后 Master 会触发“工作负载大转移”的自动流程。

Event 是一个事件的记录，记录了事件的最早产生时间、最后重现时间、重复次数、发起者、类型，以及导致此事件的原因等众多信息。Event 通常会关联到某个具体的资源对象上，是排查故障的重要参考信息，使用 `kubectl describe node xxx` 获取到的 Node 的描述信息中包括了 Event。