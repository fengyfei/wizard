# Storage

## Storage Factory

接口定义如下：

```go
type StorageFactory interface {
	// New finds the storage destination for the given group and resource. It will
	// return an error if the group has no storage destination configured.
	NewConfig(groupResource schema.GroupResource) (*storagebackend.Config, error)

	// ResourcePrefix returns the overridden resource prefix for the GroupResource
	// This allows for cohabitation of resources with different native types and provides
	// centralized control over the shape of etcd directories
	ResourcePrefix(groupResource schema.GroupResource) string

	// Backends gets all backends for all registered storage destinations.
	// Used for getting all instances for health validations.
	Backends() []Backend
}
```

### Default Storage Factory

![Storage Factory Overview](./images/apiserver_default_factory.svg)

### Group Resource Overrides

![Group Resource Overrides](./images/group_resource_overrides_overview.svg)

在创建 StorageFactory 时，已经注册了以下 GroupResource:

![Group Resource Registered](./images/group_resource_registered.svg)

### Transformer

![Transformer Overview](./images/value_transformer.svg)

### 引用关系

![Storage Factory References](./images/storage_factory_references.svg)
