# Bridge Network

## driver 结构

```go
type driver struct {
	config         *configuration
	network        *bridgeNetwork
	natChain       *iptables.ChainInfo
	filterChain    *iptables.ChainInfo
	isolationChain *iptables.ChainInfo
	networks       map[string]*bridgeNetwork
	store          datastore.DataStore
	nlh            *netlink.Handle
	configNetwork  sync.Mutex
	sync.Mutex
}
```

## 初始化方法

```go
d := newDriver()
if err := d.configure(config); err != nil {
	return err
}

c := driverapi.Capability{
	DataScope:         datastore.LocalScope, // LocalScope，分配信息，仅限本地
	ConnectivityScope: datastore.LocalScope, // LocalScope，仅限本地连接
}

// dc 为 DrvRegistry
return dc.RegisterDriver(networkType, d, c)
```

## 场景分析

### 创建网络

根据 Driver 接口，创建网络使用的方法为 CreateNetwork。首先，进行参数验证：

```go
// IP 配置是否正确
if len(ipV4Data) == 0 || ipV4Data[0].Pool.String() == "0.0.0.0/0" {
	return types.BadRequestErrorf("ipv4 pool is empty")
}

// 网络是否已存在
d.Lock()
if _, ok := d.networks[id]; ok {
	d.Unlock()
	return types.ForbiddenErrorf("network %s exists", id)
}
d.Unlock()
```

再执行 parseNetworkOptions 解析配置文件，这部分代码主要注意一下检查网络接口是否存在这一行：

```go
exists, err := bridgeInterfaceExists(config.BridgeName)
```

看一下 bridgeInterfaceExists 的 Linux 实现：

```go
func bridgeInterfaceExists(name string) (bool, error) {
	// 获取 netlink handler
	nlh := ns.NlHandle()

	// 查找 name 的链接是否存在
	link, err := nlh.LinkByName(name)
	if err != nil {
		if strings.Contains(err.Error(), "Link not found") {
			return false, nil
		}
		return false, fmt.Errorf("failed to check bridge interface existence: %v", err)
	}

	// 类型是否为 bridge
	if link.Type() == "bridge" {
		return true, nil
	}
	return false, fmt.Errorf("existing interface %s is not a bridge", name)
}
```

并根据接口是否存在，保存不同的接口创建方法标识：

```go
if !exists {
	config.BridgeIfaceCreator = ifaceCreatedByLibnetwork
} else {
	config.BridgeIfaceCreator = ifaceCreatedByUser
}
```

然后，检查 IP 是否冲突：

```go
if err = d.checkConflict(config); err != nil
```

一切前置检查完毕后，开始创建网络。

### createNetwork

初始化操作系统相关的上下文：

```go
defer osl.InitOSContext()()
```

看一下 Linux 版本实现：

```go
func InitOSContext() func() {
	runtime.LockOSThread()
	// 设置网络 Namespace
	// 并执行 return os.Readlink(fmt.Sprintf("/proc/%d/task/%d/ns/net", os.Getpid(), syscall.Gettid())) 读取信息
	if err := ns.SetNamespace(); err != nil {
		logrus.Error(err)
	}
	return runtime.UnlockOSThread
}
```

然后，获取当前全部 bridge 网络：

```go
networkList := d.getNetworks()
```

创建或获取 bridge 设备对象；创建对象时，并没有创建 bridge 设备。

```go
bridgeIface, err := newInterface(d.nlh, config)
```

创建 network 对象：

```go
network := &bridgeNetwork{
	id:         config.ID,
	endpoints:  make(map[string]*bridgeEndpoint),
	config:     config,
	portMapper: portmapper.New(d.config.UserlandProxyPath),
	bridge:     bridgeIface,
	driver:     d,
}
```

记录新的 network:

```go
d.Lock()
d.networks[config.ID] = network
d.Unlock()
```

生成 bridge 设备创建配置:

```go
bridgeSetup := newBridgeSetup(config, bridgeIface)

// 如果 bridge 设备不存在，添加生成设备方法
bridgeAlreadyExists := bridgeIface.exists()
if !bridgeAlreadyExists {
	bridgeSetup.queueStep(setupDevice)
}

// 添加 bridge 设备设置 IPv4 地址方法
bridgeSetup.queueStep(setupBridgeIPv4)

// 根据配置文件，设置对应的方法
for _, step := range []struct {
	Condition bool
	Fn        setupStep
}{
	// Enable IPv6 on the bridge if required. We do this even for a
	// previously  existing bridge, as it may be here from a previous
	// installation where IPv6 wasn't supported yet and needs to be
	// assigned an IPv6 link-local address.
	{config.EnableIPv6, setupBridgeIPv6},

	// We ensure that the bridge has the expectedIPv4 and IPv6 addresses in
	// the case of a previously existing device.
	{bridgeAlreadyExists, setupVerifyAndReconcile},

	// Enable IPv6 Forwarding
	{enableIPv6Forwarding, setupIPv6Forwarding},

	// Setup Loopback Adresses Routing
	{!d.config.EnableUserlandProxy, setupLoopbackAdressesRouting},

	// Setup IPTables.
	{d.config.EnableIPTables, network.setupIPTables},

	//We want to track firewalld configuration so that
	//if it is started/reloaded, the rules can be applied correctly
	{d.config.EnableIPTables, network.setupFirewalld},

	// Setup DefaultGatewayIPv4
	{config.DefaultGatewayIPv4 != nil, setupGatewayIPv4},

	// Setup DefaultGatewayIPv6
	{config.DefaultGatewayIPv6 != nil, setupGatewayIPv6},

	// Add inter-network communication rules.
	{d.config.EnableIPTables, setupNetworkIsolationRules},

	//Configure bridge networking filtering if ICC is off and IP tables are enabled
	{!config.EnableICC && d.config.EnableIPTables, setupBridgeNetFiltering},
} {
	if step.Condition {
		bridgeSetup.queueStep(step.Fn)
	}
}

// 添加启动方法
bridgeSetup.queueStep(setupDeviceUp)
```

最后，执行 bridgeSetup 生成网络：

```go
bridgeSetup.apply()
```

## References

- [Understanding And Programming With Netlink Sockets](https://people.redhat.com/nhorman/papers/netlink.pdf)
- [netns](https://github.com/vishvananda/netns)
- [netlink](https://github.com/vishvananda/netlink)
