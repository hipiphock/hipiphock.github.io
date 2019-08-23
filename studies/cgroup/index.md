[test1](https://hipiphock.github.io/cgroup/test1)

[test2](https://hipiphock.github.io/cgroup/test2)

# cgroup
**cgroup**은 linux에서 자원할당을 효율적으로 하기 위해서 존재하는 기능 중 하나이다.
cgroup은 여러 task를 여러 group으로 나누고, 그 나눠진 group 내에서 resource를 관리한다.
이 기능의 중요한 특징 중 하나는 **file system**을 이용해서 작업을 관리한다는 것이다.

### Definition

**cgroup**: associates a set of tasks with a set of parameters for one or more subsystems
  
**subsystem**:  **resource controller**. cgroup에 있는 process들의 자원들을 관리하는 놈
  
**hierarchy**:  set of cgroups arranged in a tree
  
**cgroup은 subsystem의 집합이며, hierarchy는 cgroup의 집합이다.**

### Why do we need this?

자원의 효율적인 관리에는 process aggregation과 process partition이 수반된다. 예를 들자면 cpusets, CKRM/ResGroups, UserBeanCounters, virtual server namespace 등이 있다. 이것들은 기본적으로 grouping/partitioning을 요구한다.
  
kernel cgroup patch는 이러한 group을 구현하는데 필요한 최소한의 mehanism을 제공한다.

### Implementation

각 cgroup은 cgroup file system에 있는 directory로 대표된다. 그리고 여러 파일들과 subdirectory를 가지고 있다.

*  tasks: list of tasks (by PID)

*  cgroups.procs: list of thread group IDs in the cgroup

*  notify_on_release flag: 만약 flag가 1로 설정되었으면(enabled) 마지막에 cgroup에 남아있던 task가 cgroup을 떠나고 지워지면, kernel은 "root에 있던 release_agent를 실행한다.

*  release_agent: root에만 존재하며, notify_on_release flag에 따라서 실행이 된다.

추가적으로 다른 file들이 subsystem들에 의해서 추가될 수도 있다.


## How to use
cpuset subsystem을 사용하며 새로운 작업을 한다고 하자.

1. mount -t tmpfs cgroup_root /sys/fs/cgroup
2. mkdir /sys/fs/cgroup/cpuset
3. mount -t cgroup -ocpuset cpuset /sys/fs/cgroup/cpuset
4. mkdir과 write(또는 echo)를 통해서 새로운 cgroup을 만든다.
5. 새로운 작업을 실행할 작업을 시작한다. 만약 test를 한다면 shell이 되지 않을까?
6. 그 task를 새로운 cgroup에다가 붙인다. 이때 tasks file에다가 PID를 쓰기만 하면 된다.
7. fork,exec, clone 등을 통해서 원하는 작업을 수행한다.

example)

``` bash
mount -t tmpfs cgroup_root /sys/fs/cgroup
mkdir /sys/fs/cgroup/cpuset
mount -t cgroup cpuset -ocpuset /sys/fs/cgroup/cpuset
cd /sys/fs/cgroup/cpuset
mkdir Charlie
cd Charlie
/bin/echo 2-3 > cpuset.cpus
/bin/echo 1 > cpuset.mems
/bin/echo $$ > tasks
sh
# The subshell 'sh' is now running in cgroup Charlie
# The next line should display '/Charlie'
cat /proc/self/cgroup
```

## Test 1
2-core CPU에서 실험을 직접 해보았다.
heavyProgram.out이라는 무한루프 프로그램을 만들었으며, 이 프로그램을 통해서 실제로 자원분배가 이루어지는지 확인해보았다.

``` bash
mount -t tmpfs cgroup_root /sys/fs/cgroup
mkdir /sys/fs/cgroup/cpuset
mount -t cgroup cpuset -ocpuset /sys/fs/cgroup/cpuset
cd /sys/fs/cgroup/cpuset
mkdir Charlie
cd Charlie
/bin/echo 1 > cpuset.cpus
/bin/echo 0 > cpuset.mems
/bin/echo 0 > tasks
cd ~/practice
./heavyProgram.out
```
이때 `top` 기능을 통해 다른 terminal에서 cpu1의 사용률이 100%까지 올라가는 것을 확인했다. 또한, .../Charlie/tasks가 해당 program의 pid를 가지고 있는 것도 확인했다.

반대로 `/bin/echo 0 > cpuset.cpus`를 했을때, cpu0의 사용률이 100%까지 올라가는 것을 확인했다.

## Test 2
CPU에서 자원을 분할해보았다.
