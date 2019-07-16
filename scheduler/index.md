[Home](https://hipiphock.github.io/)

[context switch](https://hipiphock.github.io/scheduler/context-switch)
[completely fair scheduler](https://hipiphock.github.io/scheduler/completely-fair-scheduler)

# Overview
기본적인 linux kernel의 scheduler에 대한 설명

현재 가장 stable한 버전인 v5.1.15를 보면서 작성하였다.

## Task
기본적으로 linux kernel은 task가 scheduling 되는 최소 단위가 된다.
kernel thread와 user process와 user thread가 task가 된다.

## Priority
linux kernel의 scheduler는 priority에 따라서 어떤 scheduler를 사용할지, weight을 어떻게 설정할지를 결정한다.

priority에는 0 ~ 139까지 총 140단계가 있으며,
0 ~ 99까지는 RT(Real Time) scheduler가,
100 ~ 139까지는 CFS(Completely Fair Scheduler)가 사용한다.

### Nice
여기에서 CFS가 쓰는 priority는 user가 지정한 nice값을 변환한 것이며,
nice는 -20~+19까지 총 40단계가 있다.
여기에서 -20이 가장 높은 우선순위를 가지며, time slice를 많이 받는다.
user가 process나 thread를 생성할 때의 nice 값은 0부터 시작한다.

# Scheduler
linux에서는 기본적으로 5개의 scheduler가 있다.
*  Stop
*  Deadline
*  RT (Real Time)
*  CFS (Completely Fair Scheduler)
*  Idle Task

여기에서 user가 쓸 수 있는 scheduler는 Deadline, RT, CFS 총 세 가지가 있다.

## Real Time Scheduler
RT Scheduler는 **어떠한 작업이든 똑같은 시간 내에 끝날 수 있도록 하기 위해 고안된 scheduler이다.**
따라서 RT scheduler는 한 시점에 단 하나의 작업이 실행되도록 하고, 동시에 높은 priority부터 순차적으로 실행이 될 수 있도록 한다.
Linux kernel의 RT scheduler에는 `SCHED_FIFO`와 `SCHED_RR`이라는 두 개의 scheduling policy가 있다.

### SCHED_FIFO
말 그대로 처음 들어온 작업을 먼저 한 다음, 다음에 들어온 작업을 하는 정책이다.
SCHED_FIFO로 작업을 할 때, yield나 preemption이 없는 한 계속 작업을 한다.
Timeslice는 없으며, 현재 작업중인 task보다 낮은 priority를 가지는 다른 작업들은 현재 작업이 CPU를 포기할 때까지 scheduling 되지 않는다.
같은 priority를 가진 작업의 경우, preemption을 하지 않는다.

### SCHED_RR
Round Robin을 쓰는 정책으로, 기본적으로 priority의 순서대로 작업을 한다는 점에서는 SCHED_FIFO와 별 차이가 없다.
SCHED_FIFO와 차이가 있다면, priority에 따라 timeslice가 배정이 된다는 점이다.
배정받은 timeslice를 다 쓰게 되면, 해당 작업은 CPU를 놓아주어야 한다.

## Completely Fair Scheduler
Completely Fair Scheduler의 main idea는 **모든 task에 완벽하게 똑같은 시간을 주자**는데 있다.

모든 task에게 똑같은 시간을 주며 밸런스를 유지하기 위해서 CFS는 task에게 주어졌던 시간을 **virtual runtime**으로 저장한다. 이 virtual runtime이 적을수록, 그 task에게 processor가 주어질 가능성이 높아진다.

이를 위해서 CFS는 red-black tree를 사용하고 있으며, red-black tree에서 `vruntime`을 키로 설정해서 sorting이 된다.

CFS에서의 weight은 사용하는 nice 우선순위에 따라서 달라진다.
nice가 0인 task에 대한 weight값은 1024가 되며, nice값이 1 감소할 때마다 10%의 time slice를 더 받을 수 있도록 weight값은 25%씩 증가한다.
vruntime은 다음과 같이 계산한다.
```
vruntime slice = time slice * weight0 / weight
```
단, 여기에서 `weight0`는 1024가 된다.

Linux kernel의 CFS에는 `SCHED_NORMAL`, `SCHED_BATCH`, `SCHED_IDLE` 총 세 가지의 정책이 있다.

### SCHED_NORMAL
task를 CFS에서 동작하게 한다.

### SCHED_BATCH
task를 CFS에서 동작하게 한다. 단, yield를 회피하여 현재 task가 최대한 처리를 할 수 있게 한다.

### SCHED_IDLE
task를 CFS에서 가장 낮은 우선순위로 동작하게 한다. (nice = 20)
