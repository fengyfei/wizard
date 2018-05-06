# Server

## Start

代码如下：

```go
func (s *EtcdServer) Start() {
	s.start()
	s.goAttach(func() { s.adjustTicks() })
	s.goAttach(func() { s.publish(s.Cfg.ReqTimeout()) })
	s.goAttach(s.purgeFile)
	s.goAttach(func() { monitorFileDescriptor(s.stopping) })
	s.goAttach(s.monitorVersions)
	s.goAttach(s.linearizableReadLoop)
	s.goAttach(s.monitorKVHash)
}
```

### start 方法

创建 Wait 结构，详细结构请看 [Widget](widget.md)

```go
s.w = wait.New()
```

创建 WaitTime 结构，详细结构请看 [Widget](widget.md)

```go
s.applyWait = wait.NewTimeList()
```

创建控制 chan：

```go
s.done = make(chan struct{})
s.stop = make(chan struct{})
s.stopping = make(chan struct{})
s.ctx, s.cancel = context.WithCancel(context.Background())
s.readwaitc = make(chan struct{}, 1)
```

创建 notifier，详细结构请看 [Widget](widget.md):

```go
s.readNotifier = newNotifier()
```

最后，执行：

```go
go s.run()
```

### run

获取快照：

```go
sn, err := s.r.raftStorage.Snapshot()
```

创建调度器：

```go
sched := schedule.NewFIFOScheduler()
```

创建 raftReadyHandler：

```go
rh := &raftReadyHandler{
	updateLeadership: func(newLeader bool) {
		if !s.isLeader() {
			if s.lessor != nil {
				s.lessor.Demote()
			}
			if s.compactor != nil {
				s.compactor.Pause()
			}
			setSyncC(nil)
		} else {
			if newLeader {
				t := time.Now()
				s.leadTimeMu.Lock()
				s.leadElectedTime = t
				s.leadTimeMu.Unlock()
			}
			setSyncC(s.SyncTicker.C)
			if s.compactor != nil {
				s.compactor.Resume()
			}
		}

		// TODO: remove the nil checking
		// current test utility does not provide the stats
		if s.stats != nil {
			s.stats.BecomeLeader()
		}
	},
	updateCommittedIndex: func(ci uint64) {
		cci := s.getCommittedIndex()
		if ci > cci {
			s.setCommittedIndex(ci)
		}
	},
}
```

启动 raftNode:

```go
s.r.start(rh)
```

获取 lessor 信息：

```go
if s.lessor != nil {
	expiredLeaseC = s.lessor.ExpiredLeasesC()
}
```

进入事件循环
