# 辅助结构

## Wait

![Wait](../images/wait.svg)


## WaitTime

![Wait Time](../images/wait_time.svg)

## Scheduler

![Scheduler](../images/scheduler.svg)

- Schedule

```go
func (f *fifo) Schedule(j Job) {
	f.mu.Lock()
	defer f.mu.Unlock()

	if f.cancel == nil {
		panic("schedule: schedule to stopped scheduler")
	}

	// pendings 队列为空
	if len(f.pendings) == 0 {
		// 锁的存在，已经可以保证发送一定成功；使用 default 只是为了更加保险
		select {
		case f.resume <- struct{}{}:
		default:
		}
	}
	f.pendings = append(f.pendings, j)
}
```
