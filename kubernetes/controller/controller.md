# Controller

## 注册配置

![New Config](images/configuration-preparation.svg)

## 

![Run](images/run-kube-controller-manager.svg)

## 启动 HTTP Server

![Start HTTP Server](images/start-http-server.svg)

Secure Serve

TLSConfig: vendor/k8s.io/apiserver/pkg/server/serve.go Serve

## 创建 

![create controller context](images/create-controller-context.svg)

## 创建 informer

这里以 serviceAccountInformer 为例

![service account informer](images/service-account-informer.svg)

重要代码

```go
func NewSharedIndexInformer(lw ListerWatcher, objType runtime.Object, defaultEventHandlerResyncPeriod time.Duration, indexers Indexers) SharedIndexInformer {
	realClock := &clock.RealClock{}
	sharedIndexInformer := &sharedIndexInformer{
		processor:                       &sharedProcessor{clock: realClock},
		indexer:                         NewIndexer(DeletionHandlingMetaNamespaceKeyFunc, indexers),
		listerWatcher:                   lw,
		objectType:                      objType,
		resyncCheckPeriod:               defaultEventHandlerResyncPeriod,
		defaultEventHandlerResyncPeriod: defaultEventHandlerResyncPeriod,
		cacheMutationDetector:           NewCacheMutationDetector(fmt.Sprintf("%T", objType)),
		clock: realClock,
	}
	return sharedIndexInformer
}
```

## References

* [informer factory](../client-go/informer_factory.md)
