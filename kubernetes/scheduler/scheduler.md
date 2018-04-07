# Scheduler

## 数据结构全景图

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

- EquivalenceCache

```go
if c.enableEquivalenceClassCache && getEquivalencePodFuncFactory != nil {
	pluginArgs, err := c.getPluginArgs()
	if err != nil {
		return nil, err
	}
	c.equivalencePodCache = core.NewEquivalenceCache(
		getEquivalencePodFuncFactory(*pluginArgs),
	)
	glog.Info("Created equivalence class cache")
}
```

从上面的代码看， EquivalenceCache 开启是有条件的。从 [Feature Gates](https://kubernetes.io/docs/reference/feature-gates/)，可以看到 EnableEquivalenceClassCache 默认是 false 状态。所以，图中 Ecache 指向 nil。

- PodConditionUpdater, PodPreemptor

通过封装 Clientset 完成，最终是通过 HTTP 完成操作。

## Run 方法

```go
func (sched *Scheduler) Run() {
	// 同步 cache，失败后只能退出
	if !sched.config.WaitForCacheSync() {
		return
	}

	if utilfeature.DefaultFeatureGate.Enabled(features.VolumeScheduling) {
		go sched.config.VolumeBinder.Run(sched.bindVolumesWorker, sched.config.StopEverything)
	}

	// 调度算法启动
	go wait.Until(sched.scheduleOne, 0, sched.config.StopEverything)
}
```

### 调度过程详解

![Scheduler Run Overview](./images/scheduler_run.svg)

- Scheduler.schedule()

```go
func (sched *Scheduler) schedule(pod *v1.Pod) (string, error) {
	// 使用算法进行调度
	host, err := sched.config.Algorithm.Schedule(pod, sched.config.NodeLister)

	// 调度失败
	if err != nil {
		glog.V(1).Infof("Failed to schedule pod: %v/%v", pod.Namespace, pod.Name)
		pod = pod.DeepCopy()
		sched.config.Error(pod, err)

		// 记录失败事件，通过 Broadcaster 发送
		sched.config.Recorder.Eventf(pod, v1.EventTypeWarning, "FailedScheduling", "%v", err)

		// 更新 Pod 状态为失败，并记录失败原因
		sched.config.PodConditionUpdater.Update(pod, &v1.PodCondition{
			Type:    v1.PodScheduled,
			Status:  v1.ConditionFalse,
			Reason:  v1.PodReasonUnschedulable,
			Message: err.Error(),
		})
		return "", err
	}
	return host, err
}
```

- Scheduler.preempt

调度失败时，尝试抢占。

```go
func (sched *Scheduler) preempt(preemptor *v1.Pod, scheduleErr error) (string, error) {
	// 如果没有开启抢占式调度，直接返回
	if !util.PodPriorityEnabled() {
		glog.V(3).Infof("Pod priority feature is not enabled. No preemption is performed.")
		return "", nil
	}
	preemptor, err := sched.config.PodPreemptor.GetUpdatedPod(preemptor)
	if err != nil {
		glog.Errorf("Error getting the updated preemptor pod object: %v", err)
		return "", err
	}

	// 抢占
	node, victims, nominatedPodsToClear, err := sched.config.Algorithm.Preempt(preemptor, sched.config.NodeLister, scheduleErr)
	metrics.PreemptionVictims.Set(float64(len(victims)))

	// 抢占失败
	if err != nil {
		glog.Errorf("Error preempting victims to make room for %v/%v.", preemptor.Namespace, preemptor.Name)
		return "", err
	}

	// 处理抢占到的 Node 及要 Kill 的 Pods
	var nodeName = ""
	if node != nil {
		nodeName = node.Name
		err = sched.config.PodPreemptor.SetNominatedNodeName(preemptor, nodeName)
		if err != nil {
			glog.Errorf("Error in preemption process. Cannot update pod %v annotations: %v", preemptor.Name, err)
			return "", err
		}

		// 删除被抢占的 Pods
		for _, victim := range victims {
			if err := sched.config.PodPreemptor.DeletePod(victim); err != nil {
				glog.Errorf("Error preempting pod %v/%v: %v", victim.Namespace, victim.Name, err)
				return "", err
			}
			sched.config.Recorder.Eventf(victim, v1.EventTypeNormal, "Preempted", "by %v/%v on node %v", preemptor.Namespace, preemptor.Name, nodeName)
		}
	}

	for _, p := range nominatedPodsToClear {
		rErr := sched.config.PodPreemptor.RemoveNominatedNodeName(p)
		if rErr != nil {
			glog.Errorf("Cannot remove nominated node annotation of pod: %v", rErr)
		}
	}
	return nodeName, err
}
```

- Scheduler.assume

```go
func (sched *Scheduler) assume(assumed *v1.Pod, host string) error {
	assumed.Spec.NodeName = host
	if err := sched.config.SchedulerCache.AssumePod(assumed); err != nil {
		glog.Errorf("scheduler cache AssumePod failed: %v", err)

		// 记录错误
		sched.config.Error(assumed, err)
		// 发送事件
		sched.config.Recorder.Eventf(assumed, v1.EventTypeWarning, "FailedScheduling", "AssumePod failed: %v", err)
		// 更改状态为失败
		sched.config.PodConditionUpdater.Update(assumed, &v1.PodCondition{
			Type:    v1.PodScheduled,
			Status:  v1.ConditionFalse,
			Reason:  "SchedulerError",
			Message: err.Error(),
		})
		return err
	}

	// 如果开启 EnableEquivalenceClassCache，使无效
	if sched.config.Ecache != nil {
		sched.config.Ecache.InvalidateCachedPredicateItemForPodAdd(assumed, host)
	}
	return nil
}
```

最后，异步的将 Pod 与 Host 做绑定。

```go
go func() {
		err := sched.bind(assumedPod, &v1.Binding{
			ObjectMeta: metav1.ObjectMeta{Namespace: assumedPod.Namespace, Name: assumedPod.Name, UID: assumedPod.UID},
			Target: v1.ObjectReference{
				Kind: "Node",
				Name: suggestedHost,
			},
		})
		// ...
	}()
```

关于算法调度，将在 [Algorithm](algorithm.md) 中说明。

## References

- [Feature Gates](https://kubernetes.io/docs/reference/feature-gates/)
