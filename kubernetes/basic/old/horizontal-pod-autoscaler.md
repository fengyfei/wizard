# Horizontal Pod Autoscaler

通过手工执行 kubectl scale 命令，我们可以实现 Pod 扩容或缩容。如果仅仅到此为止，显然不符合谷歌对 Kubernetes 的定位目标 -- 自动化、智能化。在谷歌看来， 分布式系统要能够根据当前负载的变化情况自动触发水平扩展或缩容的行为，因为这一过程可能是频繁发生的、不可预料的，所以手动控制的方式是不现实的。

Horizontal Pod Autoscaling 简称 HPA，意思是 Pod 横向自动扩容，与之前的 RC、Deployment 一样，也属于一种 Kubernetes 资源对象。通过追踪分析 RC 控制的所有目标 Pod 的负载变化情况，来确定是否需要针对性地调整目标 Pod 的副本数，这是 HPA 的实现原理。当前， HPA 可以有以下两种方式作为 Pod 负载的度量指标。

* CPUUtilizationPercentage
* 应用程序自定义的度量指标，比如服务在每秒内的相应的请求数（TPS或QPS）

CPUUtilizationPercentage 是一个算术平均值，即目标 Pod 所有副本自身的 CPU 利用率的平均值。如果某一时刻 CPUUtilizationPercentage 的值超过80%，则意味着当前的 Pod 副本数很可能不足以支撑接下来更多的请求，需要进行动态扩容，而当请求高峰时段过去后，Pod 的 CPU 利用率又会降下来，此时对应的 Pod 副本数应该自动减少到一个合理的水平。

除了使用 YAML 定义文件，还可以使用命令创建 HPA 对象，例如：`kubectl autoscale deployment php-apache --cpu-percent=90 --min=l --max=10`