# Overview
**Kubernetes (K8s) is an open-source system for automating deployment, scaling, and management of containerized applications.**

## scheduling in kubernetes
Kubernetes에서의 scheduling이란 pod에다가 node를 할당하는 것을 의미한다.

kubernetes에서는 다음 과정을 거쳐서 scheduling을 한다.

1. predication - 각 pod를 위한 node를 선별한다.
2. prioritize - 선별된 node들을 rank한다.

여기에서 가장 높은 priority를 가진 node가 선택된다.

# code


### Run()
``` go
// Run begins watching and scheduling.
// It waits for cache to be synced, then starts a goroutine and returns immediately.
func (sched *Scheduler) Run() {
	if !sched.config.WaitForCacheSync() {
		return
	}

	go wait.Until(sched.scheduleOne, 0, sched.config.StopEverything)
}
```
sched.scheduleOne()을 무한루프 돌자.

### scheduleOne()
``` go
// scheduleOne does the entire scheduling workflow for a single pod.  It is serialized on the scheduling algorithm's host fitting.
func (sched *Scheduler) scheduleOne() {
	fwk := sched.config.Framework


	pod := sched.config.NextPod()
	// pod could be nil when schedulerQueue is closed
	if pod == nil {
		return
	}
	if pod.DeletionTimestamp != nil {
		sched.config.Recorder.Eventf(pod, nil, v1.EventTypeWarning, "FailedScheduling", "Scheduling", "skip schedule deleting pod: %v/%v", pod.Namespace, pod.Name)
		klog.V(3).Infof("Skip schedule deleting pod: %v/%v", pod.Namespace, pod.Name)
		return
	}


	klog.V(3).Infof("Attempting to schedule pod: %v/%v", pod.Namespace, pod.Name)


	// Synchronously attempt to find a fit for the pod.
	start := time.Now()
	pluginContext := framework.NewPluginContext()
	scheduleResult, err := sched.schedule(pod, pluginContext)
	if err != nil {
		// schedule() may have failed because the pod would not fit on any host, so we try to
		// preempt, with the expectation that the next time the pod is tried for scheduling it
		// will fit due to the preemption. It is also possible that a different pod will schedule
		// into the resources that were preempted, but this is harmless.
		if fitError, ok := err.(*core.FitError); ok {
			if sched.config.DisablePreemption {
				klog.V(3).Infof("Pod priority feature is not enabled or preemption is disabled by scheduler configuration." +
					" No preemption is performed.")
			} else {
				preemptionStartTime := time.Now()
				sched.preempt(pod, fitError)
				metrics.PreemptionAttempts.Inc()
				metrics.SchedulingAlgorithmPremptionEvaluationDuration.Observe(metrics.SinceInSeconds(preemptionStartTime))
				metrics.DeprecatedSchedulingAlgorithmPremptionEvaluationDuration.Observe(metrics.SinceInMicroseconds(preemptionStartTime))
				metrics.SchedulingLatency.WithLabelValues(metrics.PreemptionEvaluation).Observe(metrics.SinceInSeconds(preemptionStartTime))
				metrics.DeprecatedSchedulingLatency.WithLabelValues(metrics.PreemptionEvaluation).Observe(metrics.SinceInSeconds(preemptionStartTime))
			}
			// Pod did not fit anywhere, so it is counted as a failure. If preemption
			// succeeds, the pod should get counted as a success the next time we try to
			// schedule it. (hopefully)
			metrics.PodScheduleFailures.Inc()
		} else {
			klog.Errorf("error selecting node for pod: %v", err)
			metrics.PodScheduleErrors.Inc()
		}
		return
	}
	metrics.SchedulingAlgorithmLatency.Observe(metrics.SinceInSeconds(start))
	metrics.DeprecatedSchedulingAlgorithmLatency.Observe(metrics.SinceInMicroseconds(start))
	// Tell the cache to assume that a pod now is running on a given node, even though it hasn't been bound yet.
	// This allows us to keep scheduling without waiting on binding to occur.
	assumedPod := pod.DeepCopy()


	// Assume volumes first before assuming the pod.
	//
	// If all volumes are completely bound, then allBound is true and binding will be skipped.
	//
	// Otherwise, binding of volumes is started after the pod is assumed, but before pod binding.
	//
	// This function modifies 'assumedPod' if volume binding is required.
	allBound, err := sched.assumeVolumes(assumedPod, scheduleResult.SuggestedHost)
	if err != nil {
		klog.Errorf("error assuming volumes: %v", err)
		metrics.PodScheduleErrors.Inc()
		return
	}


	// Run "reserve" plugins.
	if sts := fwk.RunReservePlugins(pluginContext, assumedPod, scheduleResult.SuggestedHost); !sts.IsSuccess() {
		sched.recordSchedulingFailure(assumedPod, sts.AsError(), SchedulerError, sts.Message())
		metrics.PodScheduleErrors.Inc()
		return
	}


	// assume modifies `assumedPod` by setting NodeName=scheduleResult.SuggestedHost
	err = sched.assume(assumedPod, scheduleResult.SuggestedHost)
	if err != nil {
		klog.Errorf("error assuming pod: %v", err)
		metrics.PodScheduleErrors.Inc()
		// trigger un-reserve plugins to clean up state associated with the reserved Pod
		fwk.RunUnreservePlugins(pluginContext, assumedPod, scheduleResult.SuggestedHost)
		return
	}
	// bind the pod to its host asynchronously (we can do this b/c of the assumption step above).
	go func() {
		// Bind volumes first before Pod
		if !allBound {
			err := sched.bindVolumes(assumedPod)
			if err != nil {
				klog.Errorf("error binding volumes: %v", err)
				metrics.PodScheduleErrors.Inc()
				// trigger un-reserve plugins to clean up state associated with the reserved Pod
				fwk.RunUnreservePlugins(pluginContext, assumedPod, scheduleResult.SuggestedHost)
				return
			}
		}


		// Run "permit" plugins.
		permitStatus := fwk.RunPermitPlugins(pluginContext, assumedPod, scheduleResult.SuggestedHost)
		if !permitStatus.IsSuccess() {
			var reason string
			if permitStatus.Code() == framework.Unschedulable {
				reason = v1.PodReasonUnschedulable
			} else {
				metrics.PodScheduleErrors.Inc()
				reason = SchedulerError
			}
			if forgetErr := sched.Cache().ForgetPod(assumedPod); forgetErr != nil {
				klog.Errorf("scheduler cache ForgetPod failed: %v", forgetErr)
			}
			// trigger un-reserve plugins to clean up state associated with the reserved Pod
			fwk.RunUnreservePlugins(pluginContext, assumedPod, scheduleResult.SuggestedHost)
			sched.recordSchedulingFailure(assumedPod, permitStatus.AsError(), reason, permitStatus.Message())
			return
		}


		// Run "prebind" plugins.
		prebindStatus := fwk.RunPrebindPlugins(pluginContext, assumedPod, scheduleResult.SuggestedHost)
		if !prebindStatus.IsSuccess() {
			var reason string
			if prebindStatus.Code() == framework.Unschedulable {
				reason = v1.PodReasonUnschedulable
			} else {
				metrics.PodScheduleErrors.Inc()
				reason = SchedulerError
			}
			if forgetErr := sched.Cache().ForgetPod(assumedPod); forgetErr != nil {
				klog.Errorf("scheduler cache ForgetPod failed: %v", forgetErr)
			}
			// trigger un-reserve plugins to clean up state associated with the reserved Pod
			fwk.RunUnreservePlugins(pluginContext, assumedPod, scheduleResult.SuggestedHost)
			sched.recordSchedulingFailure(assumedPod, prebindStatus.AsError(), reason, prebindStatus.Message())
			return
		}


		err := sched.bind(assumedPod, scheduleResult.SuggestedHost, pluginContext)
		metrics.E2eSchedulingLatency.Observe(metrics.SinceInSeconds(start))
		metrics.DeprecatedE2eSchedulingLatency.Observe(metrics.SinceInMicroseconds(start))
		if err != nil {
			klog.Errorf("error binding pod: %v", err)
			metrics.PodScheduleErrors.Inc()
			// trigger un-reserve plugins to clean up state associated with the reserved Pod
			fwk.RunUnreservePlugins(pluginContext, assumedPod, scheduleResult.SuggestedHost)
			sched.recordSchedulingFailure(assumedPod, err, SchedulerError, fmt.Sprintf("Binding rejected: %v", err))
		} else {
			klog.V(2).Infof("pod %v/%v is bound successfully on node %v, %d nodes evaluated, %d nodes were found feasible", assumedPod.Namespace, assumedPod.Name, scheduleResult.SuggestedHost, scheduleResult.EvaluatedNodes, scheduleResult.FeasibleNodes)
			metrics.PodScheduleSuccesses.Inc()


			// Run "postbind" plugins.
			fwk.RunPostbindPlugins(pluginContext, assumedPod, scheduleResult.SuggestedHost)
		}
	}()
}
```

### schedule()
``` go
// schedule implements the scheduling algorithm and returns the suggested host.
func (sched *Scheduler) schedule(pod *v1.Pod) (string, error) {
	host, err := sched.config.Algorithm.Schedule(pod, sched.config.NodeLister)
	if err != nil {
		glog.V(1).Infof("Failed to schedule pod: %v/%v", pod.Namespace, pod.Name)
		copied, cerr := api.Scheme.Copy(pod)
		if cerr != nil {
			runtime.HandleError(err)
			return "", err
		}
		pod = copied.(*v1.Pod)
		sched.config.Error(pod, err)
		sched.config.Recorder.Eventf(pod, v1.EventTypeWarning, "FailedScheduling", "%v", err)
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


## Filtering
Filtering is done by this function

### findNodesThatFit()
``` go
// Filters the nodes to find the ones that fit based on the given predicate functions
// Each node is passed through the predicate functions to determine if it is a fit
func (g *genericScheduler) findNodesThatFit(pluginContext *framework.PluginContext, pod *v1.Pod, nodes []*v1.Node) ([]*v1.Node, FailedPredicateMap, error) {
	var filtered []*v1.Node
	failedPredicateMap := FailedPredicateMap{}


	if len(g.predicates) == 0 {
		filtered = nodes
	} else {
		allNodes := int32(g.cache.NodeTree().NumNodes())
		numNodesToFind := g.numFeasibleNodesToFind(allNodes)


		// Create filtered list with enough space to avoid growing it
		// and allow assigning.
		filtered = make([]*v1.Node, numNodesToFind)
		errs := errors.MessageCountMap{}
		var (
			predicateResultLock sync.Mutex
			filteredLen         int32
		)


		ctx, cancel := context.WithCancel(context.Background())


		// We can use the same metadata producer for all nodes.
		meta := g.predicateMetaProducer(pod, g.nodeInfoSnapshot.NodeInfoMap)


		checkNode := func(i int) {
			nodeName := g.cache.NodeTree().Next()


			fits, failedPredicates, err := podFitsOnNode(
				pod,
				meta,
				g.nodeInfoSnapshot.NodeInfoMap[nodeName],
				g.predicates,
				g.schedulingQueue,
				g.alwaysCheckAllPredicates,
			)
			if err != nil {
				predicateResultLock.Lock()
				errs[err.Error()]++
				predicateResultLock.Unlock()
				return
			}
			if fits {
				// Iterate each plugin to verify current node
				status := g.framework.RunFilterPlugins(pluginContext, pod, nodeName)
				if !status.IsSuccess() {
					predicateResultLock.Lock()
					failedPredicateMap[nodeName] = append(failedPredicateMap[nodeName],
						predicates.NewFailureReason(status.Message()))
					if status.Code() != framework.Unschedulable {
						errs[status.Message()]++
					}
					predicateResultLock.Unlock()
					return
				}


				length := atomic.AddInt32(&filteredLen, 1)
				if length > numNodesToFind {
					cancel()
					atomic.AddInt32(&filteredLen, -1)
				} else {
					filtered[length-1] = g.nodeInfoSnapshot.NodeInfoMap[nodeName].Node()
				}
			} else {
				predicateResultLock.Lock()
				failedPredicateMap[nodeName] = failedPredicates
				predicateResultLock.Unlock()
			}
		}


		// Stops searching for more nodes once the configured number of feasible nodes
		// are found.
		workqueue.ParallelizeUntil(ctx, 16, int(allNodes), checkNode)


		filtered = filtered[:filteredLen]
		if len(errs) > 0 {
			return []*v1.Node{}, FailedPredicateMap{}, errors.CreateAggregateFromMessageCountMap(errs)
		}
	}


	if len(filtered) > 0 && len(g.extenders) != 0 {
		for _, extender := range g.extenders {
			if !extender.IsInterested(pod) {
				continue
			}
			filteredList, failedMap, err := extender.Filter(pod, filtered, g.nodeInfoSnapshot.NodeInfoMap)
			if err != nil {
				if extender.IsIgnorable() {
					klog.Warningf("Skipping extender %v as it returned error %v and has ignorable flag set",
						extender, err)
					continue
				} else {
					return []*v1.Node{}, FailedPredicateMap{}, err
				}
			}


			for failedNodeName, failedMsg := range failedMap {
				if _, found := failedPredicateMap[failedNodeName]; !found {
					failedPredicateMap[failedNodeName] = []predicates.PredicateFailureReason{}
				}
				failedPredicateMap[failedNodeName] = append(failedPredicateMap[failedNodeName], predicates.NewFailureReason(failedMsg))
			}
			filtered = filteredList
			if len(filtered) == 0 {
				break
			}
		}
	}
	return filtered, failedPredicateMap, nil
}
```

## prioritizing
`PrioritizeNodes` 함수에서 점수를 준다. 0-10점 사이로, 0은 제일 낮은 priority, 10은 제일 높은 priority다.
``` go
// PrioritizeNodes prioritizes the nodes by running the individual priority functions in parallel.
// Each priority function is expected to set a score of 0-10
// 0 is the lowest priority score (least preferred node) and 10 is the highest
// Each priority function can also have its own weight
// The node scores returned by the priority function are multiplied by the weights to get weighted scores
// All scores are finally combined (added) to get the total weighted scores of all nodes
func PrioritizeNodes(
	pod *v1.Pod,
	nodeNameToInfo map[string]*schedulernodeinfo.NodeInfo,
	meta interface{},
	priorityConfigs []priorities.PriorityConfig,
	nodes []*v1.Node,
	extenders []algorithm.SchedulerExtender,
	framework framework.Framework,
	pluginContext *framework.PluginContext) (schedulerapi.HostPriorityList, error) {
	// If no priority configs are provided, then the EqualPriority function is applied
	// This is required to generate the priority list in the required format
	if len(priorityConfigs) == 0 && len(extenders) == 0 {
		result := make(schedulerapi.HostPriorityList, 0, len(nodes))
		for i := range nodes {
			hostPriority, err := EqualPriorityMap(pod, meta, nodeNameToInfo[nodes[i].Name])
			if err != nil {
				return nil, err
			}
			result = append(result, hostPriority)
		}
		return result, nil
	}


	var (
		mu   = sync.Mutex{}
		wg   = sync.WaitGroup{}
		errs []error
	)
	appendError := func(err error) {
		mu.Lock()
		defer mu.Unlock()
		errs = append(errs, err)
	}


	results := make([]schedulerapi.HostPriorityList, len(priorityConfigs), len(priorityConfigs))


	// DEPRECATED: we can remove this when all priorityConfigs implement the
	// Map-Reduce pattern.
	for i := range priorityConfigs {
		if priorityConfigs[i].Function != nil {
			wg.Add(1)
			go func(index int) {
				defer wg.Done()
				var err error
				results[index], err = priorityConfigs[index].Function(pod, nodeNameToInfo, nodes)
				if err != nil {
					appendError(err)
				}
			}(i)
		} else {
			results[i] = make(schedulerapi.HostPriorityList, len(nodes))
		}
	}


	workqueue.ParallelizeUntil(context.TODO(), 16, len(nodes), func(index int) {
		nodeInfo := nodeNameToInfo[nodes[index].Name]
		for i := range priorityConfigs {
			if priorityConfigs[i].Function != nil {
				continue
			}


			var err error
			results[i][index], err = priorityConfigs[i].Map(pod, meta, nodeInfo)
			if err != nil {
				appendError(err)
				results[i][index].Host = nodes[index].Name
			}
		}
	})


	for i := range priorityConfigs {
		if priorityConfigs[i].Reduce == nil {
			continue
		}
		wg.Add(1)
		go func(index int) {
			defer wg.Done()
			if err := priorityConfigs[index].Reduce(pod, meta, nodeNameToInfo, results[index]); err != nil {
				appendError(err)
			}
			if klog.V(10) {
				for _, hostPriority := range results[index] {
					klog.Infof("%v -> %v: %v, Score: (%d)", util.GetPodFullName(pod), hostPriority.Host, priorityConfigs[index].Name, hostPriority.Score)
				}
			}
		}(i)
	}
	// Wait for all computations to be finished.
	wg.Wait()
	if len(errs) != 0 {
		return schedulerapi.HostPriorityList{}, errors.NewAggregate(errs)
	}


	// Run the Score plugins.
	scoresMap, scoreStatus := framework.RunScorePlugins(pluginContext, pod, nodes)
	if !scoreStatus.IsSuccess() {
		return schedulerapi.HostPriorityList{}, scoreStatus.AsError()
	}


	// Summarize all scores.
	result := make(schedulerapi.HostPriorityList, 0, len(nodes))


	for i := range nodes {
		result = append(result, schedulerapi.HostPriority{Host: nodes[i].Name, Score: 0})
		for j := range priorityConfigs {
			result[i].Score += results[j][i].Score * priorityConfigs[j].Weight
		}
	}


	for _, scoreList := range scoresMap {
		for i := range nodes {
			result[i].Score += scoreList[i]
		}
	}


	if len(extenders) != 0 && nodes != nil {
		combinedScores := make(map[string]int, len(nodeNameToInfo))
		for i := range extenders {
			if !extenders[i].IsInterested(pod) {
				continue
			}
			wg.Add(1)
			go func(extIndex int) {
				defer wg.Done()
				prioritizedList, weight, err := extenders[extIndex].Prioritize(pod, nodes)
				if err != nil {
					// Prioritization errors from extender can be ignored, let k8s/other extenders determine the priorities
					return
				}
				mu.Lock()
				for i := range *prioritizedList {
					host, score := (*prioritizedList)[i].Host, (*prioritizedList)[i].Score
					if klog.V(10) {
						klog.Infof("%v -> %v: %v, Score: (%d)", util.GetPodFullName(pod), host, extenders[extIndex].Name(), score)
					}
					combinedScores[host] += score * weight
				}
				mu.Unlock()
			}(i)
		}
		// wait for all go routines to finish
		wg.Wait()
		for i := range result {
			result[i].Score += combinedScores[result[i].Host]
		}
	}


	if klog.V(10) {
		for i := range result {
			klog.Infof("Host %s => Score %d", result[i].Host, result[i].Score)
		}
	}
	return result, nil
}
```
