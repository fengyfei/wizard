# Driver

## 概览

![Driver](./images/driver.png)

## Driver 接口

```go
type Driver interface {
	discoverapi.Discover

	// NetworkAllocate invokes the driver method to allocate network
	// specific resources passing network id and network specific config.
	// It returns a key,value pair of network specific driver allocations
	// to the caller.
	NetworkAllocate(nid string, options map[string]string, ipV4Data, ipV6Data []IPAMData) (map[string]string, error)

	// NetworkFree invokes the driver method to free network specific resources
	// associated with a given network id.
	NetworkFree(nid string) error

	// CreateNetwork invokes the driver method to create a network
	// passing the network id and network specific config. The
	// config mechanism will eventually be replaced with labels
	// which are yet to be introduced. The driver can return a
	// list of table names for which it is interested in receiving
	// notification when a CRUD operation is performed on any
	// entry in that table. This will be ignored for local scope
	// drivers.
	CreateNetwork(nid string, options map[string]interface{}, nInfo NetworkInfo, ipV4Data, ipV6Data []IPAMData) error

	// DeleteNetwork invokes the driver method to delete network passing
	// the network id.
	DeleteNetwork(nid string) error

	// CreateEndpoint invokes the driver method to create an endpoint
	// passing the network id, endpoint id endpoint information and driver
	// specific config. The endpoint information can be either consumed by
	// the driver or populated by the driver. The config mechanism will
	// eventually be replaced with labels which are yet to be introduced.
	CreateEndpoint(nid, eid string, ifInfo InterfaceInfo, options map[string]interface{}) error

	// DeleteEndpoint invokes the driver method to delete an endpoint
	// passing the network id and endpoint id.
	DeleteEndpoint(nid, eid string) error

	// EndpointOperInfo retrieves from the driver the operational data related to the specified endpoint
	EndpointOperInfo(nid, eid string) (map[string]interface{}, error)

	// Join method is invoked when a Sandbox is attached to an endpoint.
	Join(nid, eid string, sboxKey string, jinfo JoinInfo, options map[string]interface{}) error

	// Leave method is invoked when a Sandbox detaches from an endpoint.
	Leave(nid, eid string) error

	// ProgramExternalConnectivity invokes the driver method which does the necessary
	// programming to allow the external connectivity dictated by the passed options
	ProgramExternalConnectivity(nid, eid string, options map[string]interface{}) error

	// RevokeExternalConnectivity asks the driver to remove any external connectivity
	// programming that was done so far
	RevokeExternalConnectivity(nid, eid string) error

	// EventNotify notifies the driver when a CRUD operation has
	// happened on a table of its interest as soon as this node
	// receives such an event in the gossip layer. This method is
	// only invoked for the global scope driver.
	EventNotify(event EventType, nid string, tableName string, key string, value []byte)

	// DecodeTableEntry passes the driver a key, value pair from table it registered
	// with libnetwork. Driver should return {object ID, map[string]string} tuple.
	// If DecodeTableEntry is called for a table associated with NetworkObject or
	// EndpointObject the return object ID should be the network id or endppoint id
	// associated with that entry. map should have information about the object that
	// can be presented to the user.
	// For exampe: overlay driver returns the VTEP IP of the host that has the endpoint
	// which is shown in 'network inspect --verbose'
	DecodeTableEntry(tablename string, key string, value []byte) (string, map[string]string)

	// Type returns the type of this driver, the network type this driver manages
	Type() string

	// IsBuiltIn returns true if it is a built-in driver
	IsBuiltIn() bool
}
```

## 驱动注册

驱动注册是在创建 NetworkController 时完成的：

```go
c := &controller{
	id:               stringid.GenerateRandomID(),
	cfg:              config.ParseConfigOptions(cfgOptions...),
	sandboxes:        sandboxTable{},
	svcRecords:       make(map[string]svcInfo),
	serviceBindings:  make(map[serviceKey]*service),
	agentInitDone:    make(chan struct{}),
	networkLocker:    locker.New(),
	DiagnosticServer: diagnostic.New(),
}

// 初始化 Store， 支持 zookeeper, etcd, consul, boltdb
if err := c.initStores(); err != nil {
	return nil, err
}

// 前两个传入参数，并没有在 drvRegistry 中保留
drvRegistry, err := drvregistry.New(c.getStore(datastore.LocalScope), c.getStore(datastore.GlobalScope), c.RegisterDriver, nil, c.cfg.PluginGetter)

// 遍历内置的驱动
for _, i := range getInitializers(c.cfg.Daemon.Experimental) {
	var dcfg map[string]interface{}

	// 外部驱动不需要用户提供配置
	if i.ntype != "remote" {
		dcfg = c.makeDriverConfig(i.ntype)
	}

	// 加载驱动
	if err := drvRegistry.AddDriver(i.ntype, i.fn, dcfg); err != nil {
		return nil, err
	}
}
```

以 Linux 为例，getInitializers 返回的内建驱动列表为：

```go
in := []initializer{
	{bridge.Init, "bridge"},
	{host.Init, "host"},
	{macvlan.Init, "macvlan"},
	{null.Init, "null"},
	{remote.Init, "remote"},
	{overlay.Init, "overlay"},
}
```

需要注意，在执行 drvRegistry.AddDriver 时，Driver 的初始化函数已经调用完成:

```go
func (r *DrvRegistry) AddDriver(ntype string, fn InitFunc, config map[string]interface{}) error {
	return fn(r, config)
}
```

然后初始化全局 IPAM：

```go
if err = initIPAMDrivers(drvRegistry, nil, c.getStore(datastore.GlobalScope)); err != nil {
	return nil, err
}

c.drvRegistry = drvRegistry
```

具体来说，initIPAMDrivers 执行以下代码：

```go
// lDs: Local Data Store
// gDs: Global Data Store
func initIPAMDrivers(r *drvregistry.DrvRegistry, lDs, gDs interface{}) error {
	for _, fn := range [](func(ipamapi.Callback, interface{}, interface{}) error){
		builtinIpam.Init,	// 内建
		remoteIpam.Init,	// 外部驱动
		nullIpam.Init,		// null 驱动
	} {
		if err := fn(r, lDs, gDs); err != nil {
			return err
		}
	}

	return nil
}
```

## References

- [consul](https://github.com/hashicorp/consul)
- [zookeeper](https://github.com/apache/zookeeper)
- [etcd](https://github.com/coreos/etcd)
- [boltdb](https://github.com/coreos/bbolt)
