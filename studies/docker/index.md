---
title: "Docker Overview"
permalink: /studies/docker/
excerpt: "Docker Overview"
layout: single
last_modified_at: 2019-08-20T21:36:11-04:00
redirect_from:
  - /theme-setup/
toc: true
sidebar:
  nav: "studies"
---
흔히 Docker라고 하면 다음 세 가지를 떠올릴 수 있다.

1. Docker, Inc.
2. Container runtime and orchestration technology.
3. Open-source project(Moby).

이 셋의 관계를 정리하면 다음과 같다.

**2번은 3번에 의해서 개발되었으며, 1번은 3번의 overall maintainer이다.**

일반적으로 Technicall하게 다루는 docker는 당연하 2번이며, 앞으로도 계속 2번에 대해서 이야기를 할 것이다.

# The Docker Platform
Docker는 container라는 isolated environment 내에서 application을 run하고 package할 능력을 제공한다. 주어진 host에서 많은 container들을 돌릴 수 있도록 한다.

Container들은 lightweight이다. 이는 hypervisor가 필요하지 않기 때문이다. host machine의 kernel에서 direct하게 돌아가기 때문이다. 따라서 VM보다도 더 많은 container를 돌릴 수 있다.

Docker는 tooling과 platform을 제공한다. 이를 통해서 container의 lifecycle을 manage할 수 있다.
 * Develop your application and its supporting components using container
 * The container becomes the unit for distributing and testing your application.
 * When you’re ready, deploy your application into your production environment, as a container or an orchestrated service. This works the same whether your production environment is a local data center, a cloud provider, or a hybrid of the two.

# Docker Engine
**Docker Engine**은 client-server application으로, 다음 세 major component가 있다.
 * A server which is a type of long-running program called a daemon process (the dockerd command).
 * A REST API which specifies interfaces that programs can use to talk to the daemon and instruct it what to do.
 * A command line interface (CLI) client (the docker command).

The **CLI** uses the Docker REST API to control or interact with the Docker daemon through scripting or direct CLI commands. Many other Docker applications use the underlying API and CLI.

The **daemon** creates and manages Docker objects, such as images, containers, networks, and volumes.

> **Note**: Docker is licensed under the open source Apache 2.0 license.

# Docker Architecture
Docker는 client-server architecture를 사용한다.
Docker의 *client*가 Docker의 *daemon*에게 메세지를 보내면 Docker daemon이 이를 처리하는 식으로 일들이 이루어진다.

## The Docker Daemon
Daemon(`dockerd`)은 Docker API request를 받아들이고 Docker object를 관리한다.
Daemon은 다른 daemon들과 소통하기도 한다.

## The Docker Client
Docker client(`docker`)는 유저들이 Docker를 사용하기 위해 쓰는 가장 주된 방법이다. `dockerd`에게 메세지를 보내는 역할을 하며, `docker` command는 Docker API를 쓴다.

## Docker Registries
Docker *registry*는 Docker image를 저장한다. Docker Hub은 누구나 쓸 수 있는 public registry이며, Docker는 default로 docker hub에서 먼저 image를 검색한다.
`docker pull`이나 `docker run` command를 쓰면, 요구된 image는 너의 configured registry로부터 pull된다. `docker push` command를 쓰면, 너의 configured registry로 image가 push된다.

## Docker Objects
Docker를 쓸 때, 너는 image, container, network, volume, plugin, 그리고 다른 object를 사용한다.

### Images
Image는 read-only template이다. 여기에는 Docker container를 만드는데 필요한 instruction들이 있다.
Image는 추가적인 customization과 함께 다른 image를 참조할 때가 종종 있다. 예를 들자면, ubuntu image를 build했지만, Apache web server와 너의 app을 설치하는 경우가 있을 것이다. 이때는 물론 configuration detail이 필요할 것이다.

너는 네 image를 직접 만들거나 registry에 있는 다른 image를 쓸 수도 있다. 직접 image를 만들기 위해서는 간단한 instruction과 함께 Dockerfile을 만들어야 한다. Dockerfile에 있는 각각의 instruction은 image에 layer를 만든다. Dockerfile을 바꾸고 image를 다시 빌드한다면, 바뀐 부분만이 rebuild될 것이다. 이게 image가 엄청 가벼운 이유 중 하나다.

### Container
Container는 image의 runnable instance이다. Docker API나 CLI를 통해서 container를 만들거나 시작하거나 멈추거나 움직이거나 지울 수 있다. Container끼리 연결을 할 수도 있고, storage를 부착할 수도 있고, 현재의 state에서 새로운 image를 만들 수도 있다.
Container는 host machine과 다른 container들과 잘 분리되어 있다. host machine에서 container의 network, storage, 그 외의 다른 subsystem의 isolation 정도를 control할 수 있다.

### Services
Services allow you to scale containers across multiple Docker daemons, which all work together as a swarm with multiple managers and workers. Each member of a swarm is a Docker daemon, and the daemons all communicate using the Docker API. A service allows you to define the desired state, such as the number of replicas of the service that must be available at any given time. By default, the service is load-balanced across all worker nodes. To the consumer, the Docker service appears to be a single application. Docker Engine supports swarm mode in Docker 1.12 and higher.