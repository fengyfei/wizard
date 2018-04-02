# Event

## Overview

![Events Overview](./images/events_overview.svg)

主要接口定义如下：

- EventSink

存储事件相关操作

```go
type EventSink interface {
	Create(event *v1.Event) (*v1.Event, error)
	Update(event *v1.Event) (*v1.Event, error)
	Patch(oldEvent *v1.Event, data []byte) (*v1.Event, error)
}
```

- Interface

停止观察，及获取 Event

```go
type Interface interface {
	Stop()

	ResultChan() <-chan Event
}
```

- EventRecorder

生成事件

```go
type EventRecorder interface {
	Event(object runtime.Object, eventtype, reason, message string)

	Eventf(object runtime.Object, eventtype, reason, messageFmt string, args ...interface{})

	PastEventf(object runtime.Object, timestamp metav1.Time, eventtype, reason, messageFmt string, args ...interface{})
}
```

## EventBroadcaster

来看一下 EventBroadcaster 实例关系图：

![Event Broadcaster Instance](./images/events_broadcaster.svg)
