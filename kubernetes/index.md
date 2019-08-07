[Home](https://hipiphock.github.io)

# Overview
**Kubernetes (K8s) is an open-source system for automating deployment, scaling, and management of containerized applications.**

[Kubernetes Scheduler](https://hipiphock.github.io/kubernetes/scheduler)

# Basic Concepts
kubernetes에서는 수많은 abstraction이 나오며, 이를 확실히 정리하는 것이 이해하는데 매우 도움이 된다.

기본적으로 Kubernetes는 Master와 Node로 구성이 된다.
 * Master: 
 * Node: 실제로 일을 하는 기본적인 hardware 구성단위.

이 Node는 나중에 Pod가 동작하는데 쓰이게 된다.
 * Pod: Kubernetes에서 실행되는 app의 가장 기본적인 구성단위.
Pod는 application의 container(s), storage resource, unique IP, 그 외에 option들을 가지고 있다.

