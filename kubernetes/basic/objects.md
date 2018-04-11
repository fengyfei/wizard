# Kubernetes Objects

## 什么是 Kubernetes Objects

<!-- 在 Kubernetes 系统中，Kubernetes Objects 是描述了 Kubernetes 集群期望状态的持久化条目。一旦创建对象，Kubernetes 系统将持续工作以确保对象存在。通过创建对象，可以有效地告知 Kubernetes 系统，所需要的集群工作负载看起来是什么样子的。它们描述了如下信息： -->

Kubernetes 对象是 Kubernetes 系统中的持久化实体。Kubernetes 使用这些实体来表示群集的状态。具体来说，他们可以描述：

* 什么容器化应用在运行（以及在哪个节点上）
* 可以被应用使用的资源
* 应用程序如何运行的策略，比如重启策略、升级策略，以及容错策略

## Objects 的规格和状态

每个 Kubernetes 对象包含两个嵌套的对象字段，它们负责管理对象的配置：对象 spec 和 对象 status。spec 必须提供，它描述了对象的期望状态 -- 希望对象所具有的特征。status 描述了对象的实际状态，它是由 Kubernetes 系统提供和更新。在任何时刻，Kubernetes Control Plane 一直处于活跃状态，管理着对象的实际状态以与我们所期望的状态相匹配。

<!-- ## Objects 的描述

在 Kubernetes 中创建对象时，必须提供描述其所需状态的对象规范以及有关该对象的一些基本信息（如名称）。下面是一个示例文件：

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

使用 `kubectl create -f nginx-deployment.yaml --record` 创建对象 -->

## Objects 描述必需字段

创建的 Kubernetes 对象需要配置的字段如下：

* apiVersion: 创建该对象所使用的 Kubernetes API 的版本
* kind: 想要创建的对象的类型
* metadata: 帮助识别对象唯一性的数据，包括一个 name 字符串、UID 和可选的 namespace

也需要提供对象的 spec 字段。对象 spec 的精确格式对每个 Kubernetes 对象来说是不同的，包含了特定于该对象的嵌套字段。具体可以参考：[Kubernetes API Reference](https://kubernetes.io/docs/reference)

## Names

在 Kubernetes REST API 中，所有的对象都能够通过 name 或 UID 准确识别出来。

客户端提供的字符串指向一个对象（资源 URL）

```url
/api/v1/pods/some-name
/group/version/kind/name
```

在同一时间，对象的名字不能重复。Kubernetes 资源的名称应该最长为 253 个字符，由小写字母、数字、'-' 和 '.' 组成，并且某些资源有更多特定的限制。

## UIDs

UID 是 Kubernetes 系统生成的唯一标识。在 Kubernetes 集群的整个生命周期内创建的每个对象都有一个不同的 UID。它旨在区分类似实体的历史事件。

## Namespaces

Kubernetes 支持在同一物理集群上的多个虚拟集群。这些虚拟集群被称为 namespaces。Namespaces 用于多个团队或项目有许多用户的环境中，将他们能访问的资源隔离开。Kubernetes 从三个初始名称空间开始：default、kube-system 和 kube-public。

## Labels and Selectors

### Labels

Labels 与 name 和 UID 不同，不一定唯一。Labels 是绑定在对象（例如 pods）上的标识，用于选择满足特定条件的对象集合。Labels 可以是有意义的、和用户有关的对象属性。Labels 可以在创建时附加到对象上也可以在之后修改。每个对象可以有一组 labels，对每个对象 key 不能重复。

Labels 让用户用自己想要的分类方式给对象进行分类，让用户可以方便地管理这些对象。下面是一些 labels 的例子：

* "release" : "stable", "release" : "canary"
* "environment" : "dev", "environment" : "qa", "environment" : "production"
* "tier" : "frontend", "tier" : "backend", "tier" : "cache"
* "partition" : "customerA", "partition" : "customerB"
* "track" : "daily", "track" : "weekly"

### Label selectors

通过标签选择器，客户端或用户可以选中一组有相应标签组合的对象。API 支持两种选择方式：equality-based、set-based。

#### Equality-based

这种方式有三种操作：`=`、`==`、`!=`。`=` 和 `==` 相同都表示相等，`!=` 表示不相等。

```
// 选中所有 environment 为 production 的对象
environment = production
// 选中所有 tier 不是 frontend 的对象
tier != frontend
```

#### Set-based

这种方式也有三种操作：`in`、`notin` 和 `exists`。

```
// 选中所有 environment 为 production 或 qa 的对象
environment in (production, qa)
// 选中所有 tier 不是 frontend 或 backend 的对象
tier notin (frontend, backend)
// exists，选中所有带有 key 包含 partition 的对象（不限制 value）
partition
// exists，选中所有带有 key 不包含 partition 的对象（不限制 value）
!partition
```

## Annotations

用户可以使用 Kubernetes annotations 将任意的非标识性的元数据附加到对象上。使用工具或库可以检索到这些元数据。

Annotations 可长可短，并且可以使用 labels 中非法的字符。
