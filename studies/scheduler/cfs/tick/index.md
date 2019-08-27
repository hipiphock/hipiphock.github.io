---
title: "Scheduler Tick"
permalink: /studies/scheduler/cfs/tick/
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
세밀한 단위의 시간을 제어하기 위해 linux kernel은 **high resolution timer**를 사용하고 있다.

High resoolution timer, 줄여서 hrtimer는 ns단위로 시간을 처리한다.

scheduler_tick()함수에서 task_tick()을 부르게 되면, CFS는 task_tick_fair()를 부르게 된다.

# Code

## task_tick_fair()
kernel/sched/fair.c에 있다. run queue에 있는 모든 sched_entity에 대해서 할 일을 수행한다.
``` c
/*
 * scheduler tick hitting a task of our scheduling class.
 *
 * NOTE: This function can be called remotely by the tick offload that
 * goes along full dynticks. Therefore no local assumption can be made
 * and everything must be accessed through the @rq and @curr passed in
 * parameters.
 */
static void task_tick_fair(struct rq *rq, struct task_struct *curr, int queued)
{
	struct cfs_rq *cfs_rq;
	struct sched_entity *se = &curr->se;

	for_each_sched_entity(se) {
		cfs_rq = cfs_rq_of(se);
		entity_tick(cfs_rq, se, queued);
	}

	if (static_branch_unlikely(&sched_numa_balancing))
		task_tick_numa(rq, curr);

	update_misfit_status(curr, rq);
	update_overutilized_status(task_rq(curr));
}
```
현재 task의 sched_entity를 가져온 후, 그 se가 호함된 run queue에서 다음 일들을 수행한다.
 * 실행시간 갱신, 만료시 reschedule 요청
 * entity load의 평균 갱신
 * blocked load 갱신
 * shares 갱신
 * preemption이 필요한 경우, flag 설정

## entity_tick()
kernel/sched/fair.c에 위치한 함수로, 각 sched_entity에 대해서 작업을 해주는 함수이다.
``` c
static void
entity_tick(struct cfs_rq *cfs_rq, struct sched_entity *curr, int queued)
{
	/*
	 * Update run-time statistics of the 'current'.
	 */
	update_curr(cfs_rq);

	/*
	 * Ensure that runnable average is periodically updated.
	 */
	update_load_avg(cfs_rq, curr, UPDATE_TG);
	update_cfs_group(curr);

#ifdef CONFIG_SCHED_HRTICK
	/*
	 * queued ticks are scheduled to match the slice, so don't bother
	 * validating it and just reschedule.
	 */
	if (queued) {
		resched_curr(rq_of(cfs_rq));
		return;
	}
	/*
	 * don't let the period tick interfere with the hrtick preemption
	 */
	if (!sched_feat(DOUBLE_TICK) &&
			hrtimer_active(&rq_of(cfs_rq)->hrtick_timer))
		return;
#endif

	if (cfs_rq->nr_running > 1)
		check_preempt_tick(cfs_rq, curr);
}
```
먼저 `curr`에 대해서 