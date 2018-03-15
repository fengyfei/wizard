# Remote Driver Management

## Remote Drivers 架构

让我们来看一下 remote driver 结构定义

```go
type driver struct {
	endpoint    *plugins.Client
	networkType string
}
```

根据 [Plugin](../plugin.md) 部分的讲解，不难看出，Remote Driver 也是通过 HTTP 方式，与外部驱动通信。

结构中的 networkType 也不难看出，是 driver 的类型。

按照 [Bridge Network](./bridge.md) 的经验，我们也照样找到 Init 方法。

在 Init 中，首先是定义了创建新 network plugin 的方法：

```go
newPluginHandler := func(name string, client *plugins.Client) {
	// 创建 driver 结构
	d := newDriver(name, client)
	// 与 remote driver provider 协商，获取 driver 的能力
	c, err := d.(*driver).getCapabilities()
	if err != nil {
		logrus.Errorf("error getting capability for %s due to %v", name, err)
		return
	}
	// 注册驱动
	if err = dc.RegisterDriver(name, d, *c); err != nil {
		logrus.Errorf("error registering driver for %s due to %v", name, err)
	}
}
```

然后，再获取全部 network plugin，并通过 newPluginHandler 注册进 DrvRegistry。

```go
// 获取 plugin 系统内建 Handle
handleFunc := plugins.Handle

// 获取 plugin 插件管理
if pg := dc.GetPluginGetter(); pg != nil {
	// 使用 PluginGetter 的 Handle 方法
	handleFunc = pg.Handle

	// 获取全部 network plugin
	activePlugins := pg.GetAllManagedPluginsByCap(driverapi.NetworkPluginEndpointType)
	for _, ap := range activePlugins {
		// 注册驱动
		newPluginHandler(ap.Name(), ap.Client())
	}
}
// 使用 newPluginHandler 处理 network plugin
handleFunc(driverapi.NetworkPluginEndpointType, newPluginHandler)
```

## 创建网络

创建网络时，首先构建 CreateNetworkRequest

```go
create := &api.CreateNetworkRequest{
	NetworkID: id,
	Options:   options,
	IPv4Data:  ipV4Data,
	IPv6Data:  ipV6Data,
}
```

然后，通过 driver.endpoint 发送请求给外部服务即可。

```go
return d.call("CreateNetwork", create, &api.CreateNetworkResponse{})
```

## References

- [Remote Drivers](https://github.com/docker/libnetwork/blob/master/docs/remote.md)
