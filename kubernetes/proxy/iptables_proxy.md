# Iptables Mode

## 概览

![Main Data Structure Overview](./images/iptables_proxier_overview.svg)

## 运行

注册 informer 的 EventHandler，通过 APIServer 监控 Etcd 状态变化。

```go
serviceConfig := config.NewServiceConfig(informerFactory.Core().InternalVersion().Services(), s.ConfigSyncPeriod)
serviceConfig.RegisterEventHandler(s.ServiceEventHandler)
go serviceConfig.Run(wait.NeverStop)

endpointsConfig := config.NewEndpointsConfig(informerFactory.Core().InternalVersion().Endpoints(), s.ConfigSyncPeriod)
endpointsConfig.RegisterEventHandler(s.EndpointsEventHandler)
go endpointsConfig.Run(wait.NeverStop)
```

两个 go 分别执行 iptables.Proxier 的 OnServiceSynced 及 OnEndpointsSynced 方法。

最终的执行为：

```go
s.Proxier.SyncLoop()
```

![Iptables Proxier Sync Loop](./images/iptables_proxier_syncloop.svg)

## References

- [IP Sysctl Options](https://www.kernel.org/doc/Documentation/networking/ip-sysctl.txt)
- [Martian Packet](https://en.wikipedia.org/wiki/Martian_packet)
