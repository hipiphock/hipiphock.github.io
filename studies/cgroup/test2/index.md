# Overview
저번에 cgroup을 통해서 CPU의 core 자원을 제한하는 실험을 했었다.
  
이번 실험에서는 특정 cgroup에 대해서 CPU usage를 제한하고, 이것과 관련하여 scheduler에서는 어떠한 작업을 하는지, scheduler에서 priority에 어떠한 영향을 주는지 알아보고자 한다.

## Design
다음과 같은 과정으로 실험을 진행한다.

1. 서로 다른 cgroup A와 B를 만든다.
2. 각각의 group에서의 cpu.shares의 값을 다르게 설정한다. 이때의 cpu.shares의 비율을 적당히 조절해서 설정한다.
3. A group과 B group에서 동시에 stress를 주는 program을 돌린다.
4. top command를 통해서 의도한 대로 흘러가는지 확인한다.
5. scheduler와 관련해서 어떠한 작업을 하는지는 어떻게 확인하지?

## Experiment
편의를 위해서 cgroup-tools를 설치했다.

``` bash
sudo apt install cgroup-tools
```

우선 서로 다른 cgroup을 /cgroup/cpu 내부에 만들었다.

``` bash
cgcreate -g cpu:agroup
cgcreate -g cpu:bgroup
```

그 다음, agroup의 cpu.shares를 256으로 설정했다. bgroup의 cpu.shares는 deafault인 1024인 상태이다.

이후에 이전에 사용했던 ./heavyProgram.out을 실행했고, 이를 각각 agroup과 bgroup에 추가했다.

``` bash
cd /sys/fs/cgroup/bgroup
/bin/echo 2178 > tasks

cd /sys/fs/cgroup/agroup
/bin/echo 2177 > tasks
```

차이점은 `top` command를 통해서 확인했다.

``` bash
...
PID     USER        PR  NI  VIRT    RES     SHR     S   %CPU    %MEM    TIME+       COMMAND
2178    hingook     20  0   4372    812     748     R   99.7    0.0     0:45.39     heavyProgram.out
2177    hingook     20  0   4372    796     732     R   30.8    0.0     0:37.78     heavyProgram.out
...
```