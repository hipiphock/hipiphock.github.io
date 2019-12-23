---
title: Kubernetes
permalink: /studies/kubernetes/
layout: single
last_modified_at: 2019-08-20T21:36:11-04:00
redirect_from:
  - /theme-setup/
toc: true
sidebar:
  nav: "studies"
---

**Kubernetes (K8s) is an open-source system for automating deployment, scaling, and management of containerized applications.**

# Basic Concepts
Kubernetes에서는 굉장히 많은 추상적인 개념이 나오며, 이를 확실히 정리하는 것은 이해하는데 매우 도움이 된다.

## Hardware
### Node
기본적으로 machine 하나하나는 **node**라고 불리우며, 이는 Kubernetes에서 가장 작은 단위의 computing hardware이다.

### Cluster
이러한 **node**는 모여서 하나의 **cluster**를 이루게 된다. 이렇게 모인 cluster는 나중에 program을 할당했을 때, 매우 지능적인 방식으로 그 program에 node를 할당하게 된다. 새로운 node가 생기거나 지워지면 cluster는 거기에 맞춰서 다시 할당을 할 것이다.

### Persistent Volume
**cluster**에서 도는 program은 특정 node에서 작동하도록 설계되지 않기 때문에, data를 관리하는데 특별한 방법이 필요하다: 기존의 node의 storage는 temporary cache로 사용을 하고, cluster에 따로 **persistent volume**을 부착한다. 이는 어떠한 node에서 program이 작동하는지와 관계없이 작동을 하며, 실제 storage의 역할을 한다.

## Software
### Container
Kubernetes에서 돌아가는 program들은 모두 **Linux Container**이다.

### Pod
Kubernetes는 다른 system과 다르게 직접적으로 **container**를 돌리지 않는다.

Kubernetes는 container(s)를 **pod**라는 high-level structure로 포장을 하며, 같은 pod에 있는 container들은 같은 resource와 local network를 공유할 것이다. 같은 pod에 있는 container들은 소통이 매우 자유롭다.

### Deployment
**Pod**가 Kubernetes에서의 basic unit이지만, pod가 직접적으로 cluster에서 실행이 되지는 않는다. **Deployment**는 pod의 replica가 얼마나 있는지를 보여주는 역할을 하며, 이는 나중에 cluster가 notify된 수만큼의 pod를 돌릴 수 있도록 한다. pod가 죽으면 deployment는 자동으로 다시 만든다.

### Ingress
Kubernetes는 기본적으로 pod와 외부 환경 사이를 격리한다. Pod가 외부 환경과 통신을 하기 위해서는 **ingress**가 필요하다.

Ingress는 통신을 위한 channel이고, 이를 cluster에 추가하기 위해서는 ingress controller나 LoadBalancer를 추가해야한다.
