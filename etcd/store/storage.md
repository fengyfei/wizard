# Storage

## Store Overview

![Store Overview](../images/store_overview.svg)

Store 接口定义：

```go
type Store interface {
	Version() int
	Index() uint64

	Get(nodePath string, recursive, sorted bool) (*Event, error)
	Set(nodePath string, dir bool, value string, expireOpts TTLOptionSet) (*Event, error)
	Update(nodePath string, newValue string, expireOpts TTLOptionSet) (*Event, error)
	Create(nodePath string, dir bool, value string, unique bool,
		expireOpts TTLOptionSet) (*Event, error)
	CompareAndSwap(nodePath string, prevValue string, prevIndex uint64,
		value string, expireOpts TTLOptionSet) (*Event, error)
	Delete(nodePath string, dir, recursive bool) (*Event, error)
	CompareAndDelete(nodePath string, prevValue string, prevIndex uint64) (*Event, error)

	Watch(prefix string, recursive, stream bool, sinceIndex uint64) (Watcher, error)

	Save() ([]byte, error)
	Recovery(state []byte) error

	Clone() Store
	SaveNoCopy() ([]byte, error)

	JsonStats() []byte
	DeleteExpiredKeys(cutoff time.Time)

	HasTTLKeys() bool
}
```

## Node

node 数据结构定义为：

```go
type node struct {
	Path string

	CreatedIndex  uint64
	ModifiedIndex uint64

	Parent *node `json:"-"` // should not encode this field! avoid circular dependency.

	ExpireTime time.Time
	Value      string           // for key-value pair
	Children   map[string]*node // for directory

	// A reference to the store this node is attached to.
	store *store
}
```

如果一个 node 的 Children 不为 nil，那么这个节点是一个 Dir；否则为一个 Key-Value 节点。Dir 节点的 Value 域没有具体值。

store 创建时，构建如下结构：

![Root Node Overview](../images/new_store_node_sketch.svg)

### 转换为 NodeExtern 结构

![Convert To External Node](../images/node_to_external.svg)

## ttlKeyHeap

![ttlKeyHeap Overview](../images/ttl_key_heap.svg)

array 只要负责 append 存储元素即可；排序操作通过 keyMap 操作。使用最小堆的特性，将超时时间最早的元素作为根节点。
