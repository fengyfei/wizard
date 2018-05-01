# Client Request Handling

## Server Context

![Server Context Overview](../images/server_ctx_overview.svg)

创建 serverCtx：

```go
func startClientListeners(cfg *Config) (sctxs map[string]*serveCtx, err error) {
	sctxs = make(map[string]*serveCtx)
	for _, u := range cfg.LCUrls {
		sctx := newServeCtx()

		// ...

		if sctx.l, err = net.Listen(proto, addr); err != nil {
			return nil, err
		}
		sctx.addr = addr

		// ...

		for k := range cfg.UserHandlers {
			sctx.userHandlers[k] = cfg.UserHandlers[k]
		}
		sctx.serviceRegister = cfg.ServiceRegister

		// ...

		sctxs[addr] = sctx
	}
}
```

## gRPC Server (V3)

代码位置 etcdserver/api/v3rpc/grpc.go 中，使用 gRPC 不错的范例。

### QuotaKVServer

![QuotaKVServer Overview](../images/quota_kv_server_overview.svg)

#### Put 流程

![KV Put Overview](../images/kv_put_process.svg)

### WatchServer

![WatchServer Overview](../images/watch_server_overview.svg)

详细讲解请看 [Watch](watch.md)

### QuotaLeaseServer

![QuotaLeaseServer Overview](../images/quota_lease_server_overview.svg)

### ClusterServer

![ClusterServer Overview](../images/cluster_server_overview.svg)

### AuthServer

![AuthServer Overview](../images/auth_server_overview.svg)

### MaintenanceServer

![MaintenanceServer Overview](../images/maintenance_server_overview.svg)

## HTTP Server (V3)

通过 [grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway) 及 [grpc-websocket-proxy](https://github.com/tmc/grpc-websocket-proxy) 转化为 gRPC 调用。

