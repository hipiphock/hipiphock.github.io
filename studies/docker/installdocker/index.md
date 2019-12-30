---
title: "Installing Docker"
permalink: /studies/docker/installdocker/
excerpt: "Installing Docker"
layout: single
last_modified_at: 2019-08-20T21:36:11-04:00
redirect_from:
  - /theme-setup/
toc: true
sidebar:
  nav: "studies"
---
설치할 수 있는 Docker의 종류는 여러가지가 있다.
 * Desktop
   * Windows
   * Mac
 * Server
   * Linux
   * Windows Server

# Dockers for Windows
Windows 10에 Docker를 설치하기 위해서는 우선 Hyper-V와 Container 기능을 켜야 한다. 그 이후에 Docker를 설치하면 깔-끔

# 간단한 명령어
다음을 입력하면 docker의 version을 확인할 수 있다.
``` bash
$ docker version
```

# Images
## Viewing Images
Docker image에 대해서 이해하는 가장 좋은 방법은 OS filesystem과 application을 가지고 있는 object라고 생각하는 것이다. Virtual machine template이라고 생각해도 편하다.

다음을 통해 현재 Docker의 이미지들을 확인할 수 있다.
``` bash
$ docker image ls
```

## Pulling Images
Docker host로 image를 가져오는 것을 **pulling**이라고 한다.
다음을 통해서 Image를 가져올 수 있다.
``` bash
$ docker image pull ubuntu:latest
```

# Containers
Docker host에 image를 pull했다면, `docker container run` command를 통해서 container를 돌릴 수 있다.
