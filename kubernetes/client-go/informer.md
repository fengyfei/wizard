# Informer

## 概览

![Informer Overview](./images/informer_overview.svg)

## sharedInformerFactory

- 启动分发机制

```go
func (f *sharedInformerFactory) Start(stopCh <-chan struct{}) {
	f.lock.Lock()
	defer f.lock.Unlock()

	// 遍历全部已注册的 informer
	for informerType, informer := range f.informers {
		// 如果没有启动，启动
		if !f.startedInformers[informerType] {
			go informer.Run(stopCh)
			f.startedInformers[informerType] = true
		}
	}
}
```
