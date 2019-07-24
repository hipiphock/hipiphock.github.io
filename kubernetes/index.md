# Overview
Kubernetes는 Google에서 시작된 project로 docker를 이용해 더 쉽게 작업들을 관리하는? 그런걸 한다.

# code
``` go
type genericScheduler struct {
	predicates   map[string]algorithm.FitPredicate
	prioritizers []algorithm.PriorityConfig
	pods         algorithm.PodLister
	random       *rand.Rand
	randomLock   sync.Mutex
}
```
