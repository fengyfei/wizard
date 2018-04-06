# FIFO

## 数据结构

```go
type FIFO struct {
	// cond 需要 lock 配合使用
	lock sync.RWMutex
	cond sync.Cond
	// slice 决定顺序；
	// items 快速查找；
	// queue 中元素必须唯一
	items map[string]interface{}
	queue []string

	// ...

	// object -> string
	keyFunc KeyFunc

	// queue 是否已关闭
	closed     bool
	closedLock sync.Mutex
}
```

## 方法

- Add

```go
func (f *FIFO) Add(obj interface{}) error {
	// 计算 key
	id, err := f.keyFunc(obj)
	if err != nil {
		return KeyError{obj, err}
	}

	// 加写锁
	f.lock.Lock()
	defer f.lock.Unlock()
	f.populated = true

	// key 不存在，添加至 queue 尾部
	if _, exists := f.items[id]; !exists {
		f.queue = append(f.queue, id)
	}

	// 创建或修改 key 对应值
	f.items[id] = obj

	// 添加完毕，广播
	f.cond.Broadcast()
	return nil
}
```

- Delete

```go
func (f *FIFO) Delete(obj interface{}) error {
	// 计算 key 值
	id, err := f.keyFunc(obj)
	if err != nil {
		return KeyError{obj, err}
	}
	f.lock.Lock()
	defer f.lock.Unlock()
	f.populated = true

	// 直接清除，不处理 queue
	delete(f.items, id)
	return err
}
```

- Pop

```go
func (f *FIFO) Pop(process PopProcessFunc) (interface{}, error) {
	f.lock.Lock()
	defer f.lock.Unlock()

	// 防止多线程下，读取失败
	for {
		// 队列为空
		for len(f.queue) == 0 {
			// 队列已关闭，直接退出
			if f.IsClosed() {
				return nil, FIFOClosedError
			}

			// 等待至少有一个元素，但不能保证一定被当前 Pop 读取到
			f.cond.Wait()
		}

		// 读取第一个 key
		id := f.queue[0]

		// 保存剩余 key
		f.queue = f.queue[1:]
		if f.initialPopulationCount > 0 {
			f.initialPopulationCount--
		}

		// 元素如果不存在，继续
		item, ok := f.items[id]
		if !ok {
			continue
		}

		// 移除当前 key
		delete(f.items, id)

		// 处理当前元素
		err := process(item)

		// 如果需要重新进队
		if e, ok := err.(ErrRequeue); ok {
			f.addIfNotPresent(id, item)
			err = e.Err
		}
		return item, err
	}
}
```
