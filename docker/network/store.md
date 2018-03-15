# Store

在 Driver 注册过程中，有如下代码：

```go
if err := c.initStores(); err != nil {
	return nil, err
}
```

那么所谓 Store 的作用是什么呢？initStores 第一行是：

```go
registerKVStores()
```

进入 registerKVStores 发现：

```go
consul.Register()
zookeeper.Register()
etcd.Register()
boltdb.Register()
```

注册了四种支持 Key/Value 存储格式的服务。我们以 boltdb 为例，继续看：

```go
func Register() {
	libkv.AddStore(store.BOLTDB, New)
}
```

只是添加了 boltdb 的构建方法，在需要时，通过 New 方法创建一个 Store 对象。再看下 Store 的定义：

```go
// Store represents the backend K/V storage
// Each store should support every call listed
// here. Or it couldn't be implemented as a K/V
// backend for libkv
type Store interface {
	// Put a value at the specified key
	Put(key string, value []byte, options *WriteOptions) error

	// Get a value given its key
	Get(key string) (*KVPair, error)

	// Delete the value at the specified key
	Delete(key string) error

	// Verify if a Key exists in the store
	Exists(key string) (bool, error)

	// Watch for changes on a key
	Watch(key string, stopCh <-chan struct{}) (<-chan *KVPair, error)

	// WatchTree watches for changes on child nodes under
	// a given directory
	WatchTree(directory string, stopCh <-chan struct{}) (<-chan []*KVPair, error)

	// NewLock creates a lock for a given key.
	// The returned Locker is not held and must be acquired
	// with `.Lock`. The Value is optional.
	NewLock(key string, options *LockOptions) (Locker, error)

	// List the content of a given prefix
	List(directory string) ([]*KVPair, error)

	// DeleteTree deletes a range of keys under a given directory
	DeleteTree(directory string) error

	// Atomic CAS operation on a single value.
	// Pass previous = nil to create a new key.
	AtomicPut(key string, value []byte, previous *KVPair, options *WriteOptions) (bool, *KVPair, error)

	// Atomic delete of a single value
	AtomicDelete(key string, previous *KVPair) (bool, error)

	// Close the store connection
	Close()
}
```

可以看到，Store 的作用应该就是存储一些 Key/Value 值，同时，可以监控一个 Key 的变化情况。

那么创建一个 Store 的方法，就很容易理解了：

```go
func NewStore(backend store.Backend, addrs []string, options *store.Config) (store.Store, error) {
	// AddStore 会将支持的 Store 放入 initializers
	if init, exists := initializers[backend]; exists {
		return init(addrs, options)
	}

	return nil, fmt.Errorf("%s %s", store.ErrBackendNotSupported.Error(), supportedBackend)
}
```