# Caddy

## Overview

![Overview](./images/overview.svg)

Instance 创建过程：

![Instance Creation](./images/instance_creation.svg)

### 启动服务器

发送 StartupEvent，详细内容请参照 [Events](events.md)

```go
// Executes Startup events
caddy.EmitEvent(caddy.StartupEvent, nil)
```

读取配置文件：

```go
caddyfileinput, err := caddy.LoadCaddyfile(serverType)
```

启动：

```go
instance, err := caddy.Start(caddyfileinput)
```

发送 InstanceStartupEvent：

```go
caddy.EmitEvent(caddy.InstanceStartupEvent, instance)
```

caddy.Start 流程图如下：

![Start Server](./images/start_server.svg)
