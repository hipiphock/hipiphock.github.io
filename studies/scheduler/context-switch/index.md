---
title: "Context Switch"
permalink: /studies/scheduler/context-switch/
excerpt: "Linux Scheduler"
layout: single
last_modified_at: 2019-08-20T21:36:11-04:00
redirect_from:
  - /theme-setup/
toc: true
sidebar:
  nav: "studies"
---
# Overview
`schedule()`함수는 context switch를 관장하는 함수이다.

# Code
기본적으로 scheduling은 interrupt가 들어오거나, 또는 현재 task가 CPU를 양보할 때 생기게 된다.

| File | Function | Line |
| ---- | -------- | ---- |
| core.c | <global> | 3514 EXPORT_SYMBOL(schedule); |
| core.c | schedule | 3502 asmlinkage __visible void __sched schedule(void ) |
| core.c | schedule_user | 3555 schedule(); |
| core.c | schedule_preempt_disabled | 3568 schedule(); |
| core.c | do_sched_yield | 4941 schedule(); |
| core.c | yield_to | 5088 schedule(); |
| core.c | io_schedule | 5131 schedule(); |
| wait.c | do_wait_intr | 315 schedule(); |
| wait.c | do_wait_intr_irq | 332 schedule(); |
| wait_bit.c | bit_wait | 198 schedule(); |

## schedule()
/kernel/sched/core.c에 위치해 있다.
``` c
asmlinkage __visible void __sched schedule(void)
{
	struct task_struct *tsk = current;

	sched_submit_work(tsk);
	do {
		preempt_disable();
		__schedule(false);
		sched_preempt_enable_no_resched();
	} while (need_resched());
	sched_update_worker(tsk);
}
EXPORT_SYMBOL(schedule);
```
현재 작업중이던 task를 저장하고 scheduling을 하는 함수이다.

`sched_submit_work(tsk)`를 통해서 현재 작업중인 task를 저장한 뒤, preemption을 disable 한 상태에서 scheduling을 한다. 이때, `__schedule()` 함수 내부가 `false`가 되어야 함에 유의하자.

`__schedule(false)`가 끝나면 다시 preemption을 enable한다.

그런 상태로 scheduling이 필요할 때까지 while loop를 돌린다.

while loop를 빠져나가면 `sched_update_worker(tsk);`를 부른다.

## sched_submit_work(struct task_struct \*tsk)
/kernel/sched/core.c에 위치해 있다.
``` c
static inline void sched_submit_work(struct task_struct *tsk)
{
	if (!tsk->state || tsk_is_pi_blocked(tsk))
		return;

	/*
	 * If a worker went to sleep, notify and ask workqueue whether
	 * it wants to wake up a task to maintain concurrency.
	 * As this function is called inside the schedule() context,
	 * we disable preemption to avoid it calling schedule() again
	 * in the possible wakeup of a kworker.
	 */
	if (tsk->flags & PF_WQ_WORKER) {
		preempt_disable();
		wq_worker_sleeping(tsk);
		preempt_enable_no_resched();
	}

	/*
	 * If we are going to sleep and we have plugged IO queued,
	 * make sure to submit it to avoid deadlocks.
	 */
	if (blk_needs_flush_plug(tsk))
		blk_schedule_flush_plug(tsk);
}
```
parameter로 주어진 task를 저장하는 함수이다.

## preemption
/inculde/linux/preempt.h에 매크로 함수로 정의되어있다.
``` c
// preempt_disable
#define preempt_disable() \
do { \
	preempt_count_inc(); \
	barrier(); \
} while (0)

// preempt_count_inc
#define preempt_count_inc() preempt_count_add(1)

// preempt_enable
#define sched_preempt_enable_no_resched() \
do { \
	barrier(); \
	preempt_count_dec(); \
} while (0)

// preempt_count_dec
#define preempt_count_dec() preempt_count_sub(1)
```
Preemption은 기존에 작업이 되고 있던 task에게서 자원을 빼앗는 것을 의미한다. 다른 말로는 특정 thread(또는 process)의 수행을 중단하고, 다른 thread(또는 process)를 실행하는 것을 의미한다. 이는 특정 thread가 자원을 무한정 사용하거나 독점하는 것을 방지할 수 있다.

Preemption은 꼭 필요한 기능 중 하나지만, 반드시 막아야 하는 경우도 존재한다. 특정 thread가 critical section에 있는 작업들을 수행한다고 하자. 이때 preemption이 일어나게 되면 concurrency 측면에서 문제가 생길 수 있다. 이러한 일을 방지하기 위해서 preemption disable은 꼭 필요하다.

위에 있는 것 처럼 `preempt_disable`은 매크로 함수로 구현이 되어있다. `preempt_count_inc()`는 `preempt_count`를 1만큼 증가시키는 일을 하며, `barrier`는 compiler의 optimization 과정에서 주어진 함수가 작성된 순서대로(barrier 위와 아래가 섞이지 않도록) 실행시켜주는 기능을 한다.

`sched_preempt_enable_no_resched`함수 또한 매크로 함수로 구현이 되어있으며, 이는 아까와는 반대로 `preempt_count`를 1만큼 줄여주는 일을 한다.

## \_\_schedule(bool preempt)
/kernel/sched/core.c에 위치해 있으며, main scheduler function이다.
``` c
/*
 * __schedule() is the main scheduler function.
 *
 * The main means of driving the scheduler and thus entering this function are:
 *
 *   1. Explicit blocking: mutex, semaphore, waitqueue, etc.
 *
 *   2. TIF_NEED_RESCHED flag is checked on interrupt and userspace return
 *      paths. For example, see arch/x86/entry_64.S.
 *
 *      To drive preemption between tasks, the scheduler sets the flag in timer
 *      interrupt handler scheduler_tick().
 *
 *   3. Wakeups don't really cause entry into schedule(). They add a
 *      task to the run-queue and that's it.
 *
 *      Now, if the new task added to the run-queue preempts the current
 *      task, then the wakeup sets TIF_NEED_RESCHED and schedule() gets
 *      called on the nearest possible occasion:
 *
 *       - If the kernel is preemptible (CONFIG_PREEMPT=y):
 *
 *         - in syscall or exception context, at the next outmost
 *           preempt_enable(). (this might be as soon as the wake_up()'s
 *           spin_unlock()!)
 *
 *         - in IRQ context, return from interrupt-handler to
 *           preemptible context
 *
 *       - If the kernel is not preemptible (CONFIG_PREEMPT is not set)
 *         then at the next:
 *
 *          - cond_resched() call
 *          - explicit schedule() call
 *          - return from syscall or exception to user-space
 *          - return from interrupt-handler to user-space
 *
 * WARNING: must be called with preemption disabled!
 */
 ```
주석은 중요한 정보가 많다. 주석을 잘 읽어보자.
 
`__schedule()`함수에 진입하게 되는 경우는 세 가지가 있다.
 
1. Explicit blocking - mutex, semaphore, waitqueue 등이 있거나

2. TIF_NEED_RESCHED flag가 interrupt와 userspace return path에서 체크되어있을 경우이다.

	task 사이에 preemption을 허용하기 위해서 scheduelr는 `scheduler_tick()`이란 timer interrupt handler 안에서 flag를 설정한다.
	

3. 새로 run queue에 들어온 task가 현재 task를 preempt한다면, 
wakeup은 `TIF_NEED_RESCHED`를 set하고 `schedule()`함수가 최대한 가까운 곳에서 호출된다.
여기에서 wakeup은 직접 `schedule()`을 부르지 않고 단순하게 run queue에다가 새로운 task를 넣는다.

	run queue에 들어온 새로운 task가 current task를 preempt하게 된다면,
	wakeup은 `TIF_NEED_RESCHED`를 설정하고 `schedule()`함수는 가능한 최대한 가까운 위치에서 호출된다.

``` c
static void __sched notrace __schedule(bool preempt)
{
	struct task_struct *prev, *next;
	unsigned long *switch_count;
	struct rq_flags rf;
	struct rq *rq;
	int cpu;

	cpu = smp_processor_id();
	rq = cpu_rq(cpu);
	prev = rq->curr;

	schedule_debug(prev);

	if (sched_feat(HRTICK))
		hrtick_clear(rq);

	local_irq_disable();
	rcu_note_context_switch(preempt);

	/*
	 * Make sure that signal_pending_state()->signal_pending() below
	 * can't be reordered with __set_current_state(TASK_INTERRUPTIBLE)
	 * done by the caller to avoid the race with signal_wake_up().
	 *
	 * The membarrier system call requires a full memory barrier
	 * after coming from user-space, before storing to rq->curr.
	 */
	rq_lock(rq, &rf);
	smp_mb__after_spinlock();

	/* Promote REQ to ACT */
	rq->clock_update_flags <<= 1;
	update_rq_clock(rq);

	switch_count = &prev->nivcsw;
	if (!preempt && prev->state) {
		if (signal_pending_state(prev->state, prev)) {
			prev->state = TASK_RUNNING;
		} else {
			deactivate_task(rq, prev, DEQUEUE_SLEEP | DEQUEUE_NOCLOCK);

			if (prev->in_iowait) {
				atomic_inc(&rq->nr_iowait);
				delayacct_blkio_start();
			}
		}
		switch_count = &prev->nvcsw;
	}

	next = pick_next_task(rq, prev, &rf);
	clear_tsk_need_resched(prev);
	clear_preempt_need_resched();

	if (likely(prev != next)) {
		rq->nr_switches++;
		rq->curr = next;
		/*
		 * The membarrier system call requires each architecture
		 * to have a full memory barrier after updating
		 * rq->curr, before returning to user-space.
		 *
		 * Here are the schemes providing that barrier on the
		 * various architectures:
		 * - mm ? switch_mm() : mmdrop() for x86, s390, sparc, PowerPC.
		 *   switch_mm() rely on membarrier_arch_switch_mm() on PowerPC.
		 * - finish_lock_switch() for weakly-ordered
		 *   architectures where spin_unlock is a full barrier,
		 * - switch_to() for arm64 (weakly-ordered, spin_unlock
		 *   is a RELEASE barrier),
		 */
		++*switch_count;

		trace_sched_switch(preempt, prev, next);

		/* Also unlocks the rq: */
		rq = context_switch(rq, prev, next, &rf);
	} else {
		rq->clock_update_flags &= ~(RQCF_ACT_SKIP|RQCF_REQ_SKIP);
		rq_unlock_irq(rq, &rf);
	}

	balance_callback(rq);
}
```
처음에 run queue를 CPU에서 가져온 후, 현재 작업을 prev라고 하자.

`sched_feat()`은 scheduler에서 특정 기능이 작동하는지 안하는지 시험해보는 함수이다.
이 경우에는 `sched_feat(HRTICK)`을 call하고, 이는 hrtimer의 기능에 대해서 시험해본다는 의미이다.
`__schedule()`을 부른 경우, scheduler를 또 hrtimer에서 부를 필요가 없기 때문에 이를 disable해준다

local interrupt를 disable해주고 `rcu_note_context_switch(preempt);`를 통해서 context switch를 ㅁㅁ한다. 
여기에서 RCU는 Read-Copy-Update라는 synchronization mechanism으로, 쉽게 이해하기 위해 reader-writer lock을 생각해보자.

본격적인 scheduling에 들어가기에 앞서서 run queue에 lock을 잡아준다.

run queue clock을 update하고, switch_count를 기존 task의 switch횟수로 바꿔준다.

만약 기존 task가 RUNNING이 아니면서 `preempt`가 `false`가 아니라면,
그러면서 signal을 처리해줘야 한다면 기존 task의 state을 `TASK_RUNNING`으로 바꿔준다.

그렇지 않다면 task를 deactivate한다.

이후에 task가 io를 wait한다면 뭐 어찌저찌하고 이거 왜 안나오냐

여기까지 기존의 task에 대한 처리를 한 후에 새로운 task를 고르게 된다.
새롭게 선택한 task가 이전의 task와 다르다면, 선택된 task를 다음에 실행할 task로 설정한다.

이후에 `context_switch`를 수행한다.

이후에는 lock을 풀어준다.

그리고 `balance_callback`을 호출해서 밸런스를 맞춰주겠지.
