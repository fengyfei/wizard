# Deployment

Deployment 是 Kubemetes 1.2 引入的概念，引入的目的是为了更好地解决 Pod 的编排问题。为此，Deployment 在内部使用了 Replica Set 来实现目的，无论从 Deployment 的作用与目的、它的 YAML 定义，还是从它的具体命令行操作来看，我们都可以把它看作 RC 的一次升级， 两者的相似度超过90%。

Deployment 相对于 RC 的一个最大升级是我们可以随时知道当前 Pod “部署”的进度。实际上由于一个 Pod 的创建、调度、绑定节点及在目标 Node 上启动对应的容器这一完整过程需要一定的时间，所以我们期待系统启动 N 个 Pod 副本的目标状态，实际上是一个连续变化的“部署过程”导致的最终状态。

Deployment 的典型使用场景有以下几个。

* 创建一个 Deployment 对象来生成对应的 Replica Set 并完成 Pod 副本的创建过程。
* 检查 Deployment 的状态来看部署动作是否完成（Pod 副本的数量是否达到预期的值)。
* 更新 Deployment 以创建新的 Pod（比如镜像升级）。
* 如果当前 Deployment 不稳定，则回滚到一个早先的 Deployment 版本。
* 挂起或者恢复一个 Deployment。

查看Deployment的信息

``` shell
# kubectl get deployments
NAME          DESIRED CURRENT UP-TO-DATE AVAILABLE AGE
tomcat-deploy 1       1       1          1         4m
```

* DESIRED: Pod 副本数量的期望值，即 Deployment 里定义的 Replica。
* CURRENT: 当前 Replica 的值，实际上是 Deployment 所创建的 Replica Set 里的 Replica 值，这个值不断增加，直到达到 DESIRED 为止，表明整个部署过程完成。
* UP-TO-DATE: 最新版本的 Pod 的副本数量，用于指示在滚动升级的过程中，有多少个 Pod 副本已经成功升级。
* AVAILABLE: 当前集群中可用的 Pod 副本数量，即集群中当前存活的 Pod 数量。