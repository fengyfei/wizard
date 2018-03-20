# Master-Node Communication

本⽂对 Master 节点（确切说是 apiserver）和 Kubernetes 集群之间的通信路径进⾏了分类。⽬的是为了让⽤户能够⾃定义他们的安装，对⽹络配置进⾏加固，使得集群能够在不可信的⽹络上（或者在⼀个云服务商完全公共的 IP 上）运⾏。

## Cluster -> Master

所有从集群到 master 的通信路径都终止于 apiserver（其它 master 组件没有被设计为可暴露远程服务）。在一个典型的部署中，apiserver 被配置为在一个安全的 HTTPS 端口（443）上监听远程连接并启用客户端[身份认证](https://kubernetes.io/docs/admin/authentication/)。应该启用一种或多种[授权](https://kubernetes.io/docs/admin/authorization/)形式，特别是在允许使用 [匿名请求](https://kubernetes.io/docs/admin/authentication/#anonymous-requests) 或 [服务账户令牌](https://kubernetes.io/docs/admin/authentication/#service-account-tokens) 的时候。

## Master -> Cluster

从 master (apiserver) 到集群有两种主要的通信路径。第⼀种是从 apiserver 到集群中每个节点上运⾏的 kubelet 进程。第⼆种是从 apiserver 通过它的代理功能到任何 node、pod 或者 service。

* apiserver -> kubelet

  从apiserver到kubelet的连接用于:

  * 获取 pods 的日志
  * 连接（通过 kubectl）运行中的 pods
  * 使用 kubelet 的端口转发功能

  默认的，apiserver 不会验证 kubelet 的服务证书，这会导致连接遭到中间人攻击，因而在不可信或公共网络上是不安全的。

  为了对这个连接进行认证，请使用 --kubelet-certificate-authority 标记给 apiserver 提供一个根证书捆绑，用于 kubelet 的服务证书。

  如果这样不可能，又要求避免在不可信的或公共的网络上进行连接，请在 apiserver 和 kubelet 之间使用 SSH 隧道。

  最后，应该启用 [kubelet 用户认证或权限认证](https://kubernetes.io/docs/admin/kubelet-authentication-authorization/) 来保护 kubelet API。

* apiserver -> nodes, pods, and services
  从 apiserver 到 node、pod或者service 的连接默认为纯 HTTP 方式，因此既没有认证，也没有加密。他们能够通过给 API URL 中的 node、pod 或 service 名称添加前缀 https: 来运行在安全的 HTTPS 连接上。但他们即不会认证 HTTPS endpoint 提供的证书，也不会提供客户端证书。这样虽然连接是加密的，但它不会提供任何完整性保证。这些连接目前还不能安全的在不可信的或公共的网络上运行。

* SSH Tunnels

  [Google Container Engine](https://cloud.google.com/kubernetes-engine/) 使用 SSH 隧道保护 Master -> Cluster 通信路径。在这种配置下，apiserver 发起一个到集群中每个节点的 SSH 隧道（连接到在 22 端口监听的 ssh 服务）并通过这个隧道传输所有到 kubelet、node、pod 或者 service 的流量。这个隧道保证流量不会在集群运行的私有 GCE 网络之外暴露。