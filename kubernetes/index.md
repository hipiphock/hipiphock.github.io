# Overview
kubernetes는 Google에서 시작된 project로 docker를 이용해 더 쉽게 작업들을 관리하는? 그런걸 한다.

## scheduling in kubernetes
kubernetes에서는 다음 과정을 거쳐서 scheduling을 한다.

1. predicate - 각 pod를 위한 node를 선별하고
2. priority function - node를 rank한다.
3. 가장 높은 priority를 가진 node가 선택된다.

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
