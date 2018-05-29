# Events

## 实现

相关定义：

```go
eventHooks = &sync.Map{}

type EventHook func(eventType EventName, eventInfo interface{}) error
```

注册 EventHook：

```go
func RegisterEventHook(name string, hook EventHook) {
	if name == "" {
		panic("event hook must have a name")
	}
	_, dup := eventHooks.LoadOrStore(name, hook)
	if dup {
		panic("hook named " + name + " already registered")
	}
}
```

分发事件

```go
func EmitEvent(event EventName, info interface{}) {
	eventHooks.Range(func(k, v interface{}) bool {
		err := v.(EventHook)(event, info)
		if err != nil {
			log.Printf("error on '%s' hook: %v", k.(string), err)
		}
		return true  // 注意返回 true
	})
}
```
