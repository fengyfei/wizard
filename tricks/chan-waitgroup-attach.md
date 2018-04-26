# Channel Waitgroup Attach Mode

## 详解

### 数据结构定义

```go
type EtcdServer struct {
	stopping chan struct{}
	wgMu sync.RWMutex
	wg sync.WaitGroup
}
```

### Attach 方法

```go
func (s *EtcdServer) goAttach(f func()) {
	s.wgMu.RLock()
	defer s.wgMu.RUnlock()
	select {
	case <-s.stopping:
		plog.Warning("server has stopped (skipping goAttach)")
		return
	default:
	}

	// now safe to add since waitgroup wait has not started yet
	s.wg.Add(1)
	go func() {
		defer s.wg.Done()
		f()
	}()
}
```

只需要在某处执行：

```go
s.wg.Wait()
```
