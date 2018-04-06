# Scheduler

## 全景图

![Scheduler Overview](./images/scheduler_overview.svg)

- Binder 来源

```go
func (c *configFactory) getBinderFunc(extenders []algorithm.SchedulerExtender) func(pod *v1.Pod) scheduler.Binder {
	var extenderBinder algorithm.SchedulerExtender

	// 优先从扩展中查找
	for i := range extenders {
		if extenders[i].IsBinder() {
			extenderBinder = extenders[i]
			break
		}
	}
	defaultBinder := &binder{c.client}

	return func(pod *v1.Pod) scheduler.Binder {
		// 优先使用扩展
		if extenderBinder != nil && extenderBinder.IsInterested(pod) {
			return extenderBinder
		}

		// 使用默认
		return defaultBinder
	}
}
```