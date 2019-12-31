---
title: "Practice with Sock Shop"
permalink: /studies/docker/practice/
excerpt: "Practice"
layout: single
last_modified_at: 2019-08-20T21:36:11-04:00
redirect_from:
  - /theme-setup/
toc: true
sidebar:
  nav: "studies"
---
Docker가 무엇이고, 어떻게 사용하는지 파악하는 가장 좋은 방법은 아무래도 직접 사용해보는 것일 것이다. 이를 위해서 microservice로 이루어진 [Sock Shop](https://microservices-demo.github.io/)을 container 단위로 분해해서 공부를 해보자.

# Sock Shop
Sock Shop은 Weaveworks에서 만든 **microservice demo application**으로, microservice의 demostration과 testing, 그리고 cloud native technology를 위해서 만들었다.

[Weaveworks](https://www.weave.works/)와 [Container Solution](https://www.container-solutions.com/)에 의해서 유지되고 관리받고 있다.

[Please see the official document for more information](https://microservices-demo.github.io/)

## Basic Architecture
기본적인 Architecture는 다음과 같다.
![Architecture](https://github.com/microservices-demo/microservices-demo.github.io/blob/HEAD/assets/Architecture.png)
그림과 같이 front-end에서 여러 API를 통해서 다른 Microservice와 통신을 하는 구조이다.

## Load Test
load test는 container에 있는 test script를 packaging 한다.

# Todo
microservice 형태로 여러 개로 구축
sock shop
https://microservices-demo.github.io/
microservice로 이루어짐 - 구축 해보기
*구축할 때, server에다가
나중에 영진이형이 접근할 수 있게

service discovery
 - 다 클라우드에 있음
 - 이를 edge 단으로 땡겨야 할까?
사용자 요청 - API gate
얘를 찾아내서 

Netflix - Eureka

구조