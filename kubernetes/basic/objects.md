# Kubernetes Objects

## 什么是 Kubernetes Objects

在 Kubernetes 系统中，Kubernetes Objects 是描述了 Kubernetes 集群期望状态的持久化条目。一旦创建对象，Kubernetes 系统将持续工作以确保对象存在。通过创建对象，可以有效地告知 Kubernetes 系统，所需要的集群工作负载看起来是什么样子的。它们描述了如下信息：

* 什么容器化应用在运行（以及在哪个 Node 上）
* 可以被应用使用的资源
* 关于应用如何表现的策略，比如重启策略、升级策略，以及容错策略

## Objects 的规格和状态

每个 Kubernetes 对象包含两个嵌套的对象字段，它们负责管理对象的配置：对象 spec 和 对象 status。spec 必须提供，它描述了对象的期望状态 -- 希望对象所具有的特征。status 描述了对象的实际状态，它是由 Kubernetes 系统提供和更新。在任何时刻，Kubernetes Control Plane 一直处于活跃状态，管理着对象的实际状态以与我们所期望的状态相匹配。

## Objects 的描述

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

使用 `kubectl create -f nginx-deployment.yaml --record` 创建对象

### 必需字段

创建的 Kubernetes 对象需要配置的字段如下：

* apiVersion: 创建该对象所使用的 Kubernetes API 的版本
* kind: 想要创建的对象的类型
* metadata: 帮助识别对象唯一性的数据，包括一个 name 字符串、UID 和可选的 namespace

也需要提供对象的 spec 字段。对象 spec 的精确格式对每个 Kubernetes 对象来说是不同的，包含了特定于该对象的嵌套字段。具体可以参考：[Kubernetes API Reference](https://kubernetes.io/docs/reference)