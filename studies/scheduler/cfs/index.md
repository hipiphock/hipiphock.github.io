---
title: "Completely Fair Scheduler"
permalink: /studies/scheduler/cfs/
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
Completely Fair Scheduler는 linux kernel의 기본 scheduler로, **모든 task가 똑같은 시간동안 작동할 수 있도록 하는 scheduler이다.**

Red-black tree 자료구조를 통해서 task들을 관리하고, 각각의 task의 virtual runtime을 통해서 정렬을 한다.

## Basic Concepts
CFS는 `sched_entity->vruntime`를 key로 red-black tree 내부에서 task들을 정렬한다.

`sched_entity`는 CFS에서의 task 하나하나를 의미하며, 
`vruntime`은 각각의 task의 실행시간을 누적시킨 값을 run queue에 배치하기 위해 load weight을 반대로 적용한 값이다.

누적되는 virtual runtime slice는 다음과 같이 산출된다.

> virtual runtime slice = time slice * weight<sub>0</sub> / weight

Time slice는 task가 실제로 동작하게 되는 시간이다. Scheduling lateny에서 전체 weight 중 자신의 weight 만큼을 할당 받는다.

> time slice = scheduling latency * weight / total weight

Scheduling latency는 모든 runnable task가 적어도 한 번씩 실행되기 위한 interval을 의미한다.

이때, 한 latency period에서 active한 process들의 동작시간 분포는 다음과 같이 처리된다.

``` c
slice * se->load.weight / cfs_rq->load.weight
```

CFS는 주어진 nice 값에 따라서 weight 값을 부여한다. 
nice 값 0을 기준으로 1024를 부여하고, nice의 값이 1 줄어들 때마다 10%의 time slice를 더 받을 수 있도록 weight를 25% 증가시킨다.

`nice->weight`의 빠른 산출을 위해서 다음과 같은 table이 /kernel/sched/core.c에 이미 구현되어있다.
``` c
/*
 * Nice levels are multiplicative, with a gentle 10% change for every
 * nice level changed. I.e. when a CPU-bound task goes from nice 0 to
 * nice 1, it will get ~10% less CPU time than another CPU-bound task
 * that remained on nice 0.
 *
 * The "10% effect" is relative and cumulative: from _any_ nice level,
 * if you go up 1 level, it's -10% CPU usage, if you go down 1 level
 * it's +10% CPU usage. (to achieve that we use a multiplier of 1.25.
 * If a task goes up by ~10% and another task goes down by ~10% then
 * the relative distance between them is ~25%.)
 */
const int sched_prio_to_weight[40] = {
 /* -20 */     88761,     71755,     56483,     46273,     36291,
 /* -15 */     29154,     23254,     18705,     14949,     11916,
 /* -10 */      9548,      7620,      6100,      4904,      3906,
 /*  -5 */      3121,      2501,      1991,      1586,      1277,
 /*   0 */      1024,       820,       655,       526,       423,
 /*   5 */       335,       272,       215,       172,       137,
 /*  10 */       110,        87,        70,        56,        45,
 /*  15 */        36,        29,        23,        18,        15,
};
```

실제로 time slice를 계산할 때, 나눗셈 대신 곱셈을 사용하기 위해서 weight의 역수를 계산해놓은 값을 사용한다.

이를 wmult라 하는데, wmult는 2<sup>32</sup>를 weight값으로 나눈 값이다. 이 값은 나중에 `>> 32`하는 식으로 처리한다.
``` c
/*
 * Inverse (2^32/x) values of the sched_prio_to_weight[] array, precalculated.
 *
 * In cases where the weight does not change often, we can use the
 * precalculated inverse to speed up arithmetics by turning divisions
 * into multiplications:
 */
const u32 sched_prio_to_wmult[40] = {
 /* -20 */     48388,     59856,     76040,     92818,    118348,
 /* -15 */    147320,    184698,    229616,    287308,    360437,
 /* -10 */    449829,    563644,    704093,    875809,   1099582,
 /*  -5 */   1376151,   1717300,   2157191,   2708050,   3363326,
 /*   0 */   4194304,   5237765,   6557202,   8165337,  10153587,
 /*   5 */  12820798,  15790321,  19976592,  24970740,  31350126,
 /*  10 */  39045157,  49367440,  61356676,  76695844,  95443717,
 /*  15 */ 119304647, 148102320, 186737708, 238609294, 286331153,
};
```

# Code - time slice

## sched_slice()
kernel/sched/fair.c에 위치해있다.
``` c
/*
 * We calculate the wall-time slice from the period by taking a part
 * proportional to the weight.
 *
 * s = p*P[w/rw]
 */
static u64 sched_slice(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	u64 slice = __sched_period(cfs_rq->nr_running + !se->on_rq);

	for_each_sched_entity(se) {
		struct load_weight *load;
		struct load_weight lw;

		cfs_rq = cfs_rq_of(se);
		load = &cfs_rq->load;

		if (unlikely(!se->on_rq)) {
			lw = cfs_rq->load;

			update_load_add(&lw, se->load.weight);
			load = &lw;
		}
		slice = __calc_delta(slice, se->load.weight, load);
	}
	return slice;
}
```
주어진 schedule entity가 해당 run queue에 있지 않은 경우, se를 cfs_rq에 추가하고, 그에 따른 slice를 반환한다.
run queue에 schedule entity를 올린 후, 해당 schedule entity에 해당되는 slice를 계산하고, 이를 반환한다.

## \_\_calc_delta()
kernel/sched/fair.c에 위치해있다.
``` c
/*
 * delta_exec * weight / lw.weight
 *   OR
 * (delta_exec * (weight * lw->inv_weight)) >> WMULT_SHIFT
 *
 * Either weight := NICE_0_LOAD and lw \e sched_prio_to_wmult[], in which case
 * we're guaranteed shift stays positive because inv_weight is guaranteed to
 * fit 32 bits, and NICE_0_LOAD gives another 10 bits; therefore shift >= 22.
 *
 * Or, weight =< lw.weight (because lw.weight is the runqueue weight), thus
 * weight/lw.weight <= 1, and therefore our shift will also be positive.
 */
static u64 __calc_delta(u64 delta_exec, unsigned long weight, struct load_weight *lw)
{
	u64 fact = scale_load_down(weight);
	int shift = WMULT_SHIFT;

	__update_inv_weight(lw);

	if (unlikely(fact >> 32)) {
		while (fact >> 32) {
			fact >>= 1;
			shift--;
		}
	}

	/* hint to use a 32x32->64 mul */
	fact = (u64)(u32)fact * lw->inv_weight;

	while (fact >> 32) {
		fact >>= 1;
		shift--;
	}

	return mul_u64_u32_shr(delta_exec, fact, shift);
}
...
// include/linux/math64.h
#ifndef mul_u64_u32_shr
static inline u64 mul_u64_u32_shr(u64 a, u32 mul, unsigned int shift)
{
	return (u64)(((unsigned __int128)a * mul) >> shift);
}
```
스케줄링 기간을 산출하여 반환한다.

처음에 weight의 inverse를 update하고, 수의 범위가 2<sup>32</sup>을 넘어가는 경우에는 scale down하는 작업을 한다.

수의 범위를 잘 잡아줬다면, `mul_u64_u32_shr(delta_exec, fact, shift)`를 불러서 주어진 weight에 따른 slice를 계산한다.

## \_\_update_inv_weight()
kernel/sched/fair.c에 위치해있다.
``` c
#define WMULT_CONST	(~0U)
#define WMULT_SHIFT	32

static void __update_inv_weight(struct load_weight *lw)
{
	unsigned long w;

	if (likely(lw->inv_weight))
		return;

	w = scale_load_down(lw->weight);

	if (BITS_PER_LONG > 32 && unlikely(w >= WMULT_CONST))
		lw->inv_weight = 1;
	else if (unlikely(!w))
		lw->inv_weight = WMULT_CONST;
	else
		lw->inv_weight = WMULT_CONST / w;
}
```
inv_weight을 update하는 함수이다.

# Code - vruntime의 산출
## sched_vslice()
kernel/sched/fair.c에 있는 함수이다.
``` c
/*
 * We calculate the vruntime slice of a to-be-inserted task.
 *
 * vs = s/w
 */
static u64 sched_vslice(struct cfs_rq *cfs_rq, struct sched_entity *se)
{
	return calc_delta_fair(sched_slice(cfs_rq, se), se);
}
```
sched_entity의 load에 해당하는 time slice값을 `sched_slice`함수를 통해 산출하고, 이를 통해서 vruntime을 구한다.

## calc_delta_fair()
kernel/sched/fair.c에 있는 함수이다.
``` c
/*
 * delta /= w
 */
static inline u64 calc_delta_fair(u64 delta, struct sched_entity *se)
{
	if (unlikely(se->load.weight != NICE_0_LOAD))
		delta = __calc_delta(delta, NICE_0_LOAD, &se->load);

	return delta;
}
```
time slice값에 해당하는 vruntime값을 산출한다.

> vruntime = time slice * weight<sub>0</sub> / weight

