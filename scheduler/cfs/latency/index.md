[Home](https://hipiphock.github.io/)

# Overview
Scheduling latency는 모든 task들을 적어도 한번씩 돌리는데 소요되는 시간을 의미한다.

### latency and time slice
CFS에서는 **target latency**라는게 있다. 모든 runnable task가 적어도 processor에서 one turn을 가지게 하는 최소한의 시간을 의미한다.

모든 runnable task는 target latency의 1/N의 slice를 가지게 된다. 이때 N은 number of tasks이다.

### minimum granularity
OS에서는 context switch를 할 때 overhead가 발생하고, 당연히 자주 일어날 수록 더 overhead가 커질 것이다.

이를 방지하기 위해서 모든 task가 돌아야 하는 최소한의 시간이 있다. 이를 **minimum granularity**라고 부르며, 이는 위에서 나온 time slice(1/N)보다 클 수도 있다.

만약 minimum granularity가 1/N보다 더 크다면, system은 overload된다.

# code


## sched_proc_update_handler()
kernel/sched/fair.c에 위치해 있다.

``` c
/**************************************************************
 * Scheduling class statistics methods:
 */

int sched_proc_update_handler(struct ctl_table *table, int write,
		void __user *buffer, size_t *lenp,
		loff_t *ppos)
{
	int ret = proc_dointvec_minmax(table, write, buffer, lenp, ppos);
	unsigned int factor = get_update_sysctl_factor();

	if (ret || !write)
		return ret;

	sched_nr_latency = DIV_ROUND_UP(sysctl_sched_latency,
					sysctl_sched_min_granularity);

#define WRT_SYSCTL(name) \
	(normalized_sysctl_##name = sysctl_##name / (factor))
	WRT_SYSCTL(sched_min_granularity);
	WRT_SYSCTL(sched_latency);
	WRT_SYSCTL(sched_wakeup_granularity);
#undef WRT_SYSCTL

	return 0;
}
#endif
```
얘는 뭐하는 함수지?

## update_sysctl()
kernel/sched/fair.c에 위치해있다.

``` c
static void update_sysctl(void)
{
	unsigned int factor = get_update_sysctl_factor();

#define SET_SYSCTL(name) \
	(sysctl_##name = (factor) * normalized_sysctl_##name)
	SET_SYSCTL(sched_min_granularity);
	SET_SYSCTL(sched_latency);
	SET_SYSCTL(sched_wakeup_granularity);
#undef SET_SYSCTL
}
```
