---
title: "Memos"
permalink: /memos/
excerpt: "Ideas and thoughts that suddenly come to my mind."
layout: single
last_modified_at: 2019-08-20T21:36:11-04:00
redirect_from:
  - /theme-setup/
toc: true
sidebar:
  nav: "memos"
---
# Edge에서의 Microservice의 구동
기존의 대부분의 Microservice App은 Cloud를 대상으로 설계되었다.
이때, **user와 cloud 사이의 latency**가 크다는 것이 문제가 될 수 있다.
User와 cloud 사이의 latency를 줄이기 위해서 **Microservice의 일부를 edge node로 옮긴다**고 생각해보자.
이 아이디어에서 파생될 수 있는 문제점은 다음과 같다.
 * DB에서의 Write Problem
 * Microservice가 제공할 수 있는 service의 대상과 범위
 * (Mobility를 고려한다면) handover problem

단순하게 생각해봐도 read-only app의 경우에는 latency를 크게 줄일 수 있다는 장점이 있을 것이다. 하지만 write의 경우, 결국 cloud에 있는 DB에 write를 해야하기 때문에, 적용할 수 없다는 문제점이 있다.
 
# More Ideas
Smart City?

## Microservice의 caching 사례
 * CDN으로 streaming
   * cache
 * block 긁고 옴 - 이미 나옴

# 5G edge gaming
mobility?
Streaming?

edge - entrypoint가 여러개 - offloading