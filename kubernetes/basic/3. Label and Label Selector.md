# Label and Label Selector

## Label

Label 是 Kubernetes 系统中另外一个核心概念。一个 Label 是一个 key=value 的键值对， 其中 key 与 value 由用户自己指定。Label 可以附加到各种资源对象上，例如 Node、Pod、 Service、RC 等，一个资源对象可以定义任意数量的 Label，同一个 Label 也可以被添加到任意数量的资源对象上去，Label 通常在资源对象定义时确定，也可以在对象创建后动态添加或者删除。

我们可以通过给指定的资源对象捆绑一个或多个不同的 Label 来实现多维度的资源分组管理功能，以便于灵活、方便地进行资源分配、调度、配置、部署等管理工作。例如:部署不同 版本的应用到不同的环境中；或者监控和分析应用（日志记录、监控、告警）等。一些常用的 Label 示例如下。

* 版本标签："release":"stable", "release":"canary"...
* 环境标签："environment":"devn", "environment":"qa", "environment":"production"
* 架构标签："tier":"frontend", "tier":"backend", "tier":"middleware"
* 分区标签："partition":"customerA", "partition":"customerB"...
* 质量管控标签："track":daily", "track":"weekly"

## Label Selector

Label Selector 可以被类比为 SQL 语句中的 where 查询条件，例如，name=redis-slave 这个 Label Selector 作用于 Pod 时，可以被类比为 select * from pod where pod’s name = 'redis-slave' 这样的语句。当前有两种 Label Selector 的表达式：基于等式的（Equality-based）和基于集合的 (Set-based)，釆用“等式类”表达式匹配标签的例子：name=redis-slave，env!=production，使用集合操作的表达式例子：name in (redis-master, redis-slave)，name not in (php-frontend)，多个条件使用 “,” 号连接。（注：name in (redis-master, redis-slave) 表示 匹配所有具有标签 name=redis-master 或者 name=redis-slave 的资源对象）

Label Selector 在 Kubemetes 中的重要使用场景有以下几处。

* kube-controller 进程通过资源对象 RC 上定义的 Label Selector 来筛选要监控的 Pod 副本的数量，从而实现 Pod 副本的数量始终符合预期设定的全自动控制流程。
* kube-proxy 进程通过 Service 的 Label Selector 来选择对应的 Pod，自动建立起每个 Service 到对应 Pod 的请求转发路由表，从而实现 Service 的智能负载均衡机制。
* 通过对某些 Node 定义特定的 Label，并且在 Pod 定义文件中使用 NodeSelector 这种标 签调度策略，kube-scheduler进程可以实现 Pod “定向调度”的特性。

总结：使用 Label 可以给对象创建多组标签，Label 和 Label Selector 共同构成了 Kubernetes 系统中最核心的应用模型，使得被管理对象能够被精细地分组管理，同时实现了整个集群的高可用性。