# genericScheduler

实现算法调度器通用功能。

## 数据结构

```go
type genericScheduler struct {
	cache                    schedulercache.Cache
	equivalenceCache         *EquivalenceCache
	schedulingQueue          SchedulingQueue
	predicates               map[string]algorithm.FitPredicate
	priorityMetaProducer     algorithm.PriorityMetadataProducer
	predicateMetaProducer    algorithm.PredicateMetadataProducer
	prioritizers             []algorithm.PriorityConfig
	extenders                []algorithm.SchedulerExtender
	lastNodeIndexLock        sync.Mutex
	lastNodeIndex            uint64
	alwaysCheckAllPredicates bool
	cachedNodeInfoMap        map[string]*schedulercache.NodeInfo
	volumeBinder             *volumebinder.VolumeBinder
	pvcLister                corelisters.PersistentVolumeClaimLister
}
```

在 [Scheduler](scheduler.md) 中，我们遗留了 Algorithm.Schedule 方法。其中的 Algorithm 就是 genericScheduler。

## 调度方法详解

### Schedule

```go
func (g *genericScheduler) Schedule(pod *v1.Pod, nodeLister algorithm.NodeLister) (string, error) {
	trace := utiltrace.New(fmt.Sprintf("Scheduling %s/%s", pod.Namespace, pod.Name))
	defer trace.LogIfLong(100 * time.Millisecond)

	// 检查基本设置是否成立，主要检查 Persistent Volume Claim 状态
	if err := podPassesBasicChecks(pod, g.pvcLister); err != nil {
		return "", err
	}

	// 获取全部节点
	nodes, err := nodeLister.List()
	if err != nil {
		return "", err
	}
	if len(nodes) == 0 {
		return "", ErrNoNodesAvailable
	}

	// Used for all fit and priority funcs.
	err = g.cache.UpdateNodeNameToInfoMap(g.cachedNodeInfoMap)
	if err != nil {
		return "", err
	}

	trace.Step("Computing predicates")
	startPredicateEvalTime := time.Now()

	// 找到最适合调度的 Node
	filteredNodes, failedPredicateMap, err := findNodesThatFit(pod, g.cachedNodeInfoMap, nodes, g.predicates, g.extenders, g.predicateMetaProducer, g.equivalenceCache, g.schedulingQueue, g.alwaysCheckAllPredicates)
	if err != nil {
		return "", err
	}

	if len(filteredNodes) == 0 {
		return "", &FitError{
			Pod:              pod,
			NumAllNodes:      len(nodes),
			FailedPredicates: failedPredicateMap,
		}
	}
	metrics.SchedulingAlgorithmPredicateEvaluationDuration.Observe(metrics.SinceInMicroseconds(startPredicateEvalTime))

	trace.Step("Prioritizing")
	startPriorityEvalTime := time.Now()

	// 过滤后，只剩余一个可用 Node
	if len(filteredNodes) == 1 {
		metrics.SchedulingAlgorithmPriorityEvaluationDuration.Observe(metrics.SinceInMicroseconds(startPriorityEvalTime))
		return filteredNodes[0].Name, nil
	}

	metaPrioritiesInterface := g.priorityMetaProducer(pod, g.cachedNodeInfoMap)
	// 获取 Node 优先级列表
	priorityList, err := PrioritizeNodes(pod, g.cachedNodeInfoMap, metaPrioritiesInterface, g.prioritizers, filteredNodes, g.extenders)
	if err != nil {
		return "", err
	}
	metrics.SchedulingAlgorithmPriorityEvaluationDuration.Observe(metrics.SinceInMicroseconds(startPriorityEvalTime))

	trace.Step("Selecting host")

	// 返回选择的 Node
	return g.selectHost(priorityList)
}
```
