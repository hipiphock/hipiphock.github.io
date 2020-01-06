---
title: "Graduation Project"
permalink: /projects/gradproj/
excerpt: "How to quickly install and setup Minimal Mistakes for use with GitHub Pages."
layout: single
last_modified_at: 2020-01-06T14:02:11-02:22
redirect_from:
  - /theme-setup/
toc: true
sidebar:
  nav: "projects"
---
# IoT 장치의 동작 무결성 검증을 위한 펌웨어 검증 시스템 개발
이 과제의 목표는 사용자 단말에서 발생시키는 장치 제어 명령에 대해 IoT 장치의 펌웨어가 동작하는 상황을 모니터링하여 정상 동작여부를 판단하는 시스템을 개발하는 것이다.

## Todo:
 * 사용자 단말과 IoT 장치는 serial로 연결
 * 사용자 단말에서 장치 제어 명령을 자동으로 생성하여 IoT 장치로 전달하는 SW 개발
 * 장치 제어 명령에 따른 IoT 장치의 펌웨어 동작 상황을 계측기를 통해 모니터링
 * 장치 제어 명령과 IoT 장치의 동작결과를 모두 저장하는 DBMS 구축
 * DBMS에서 특정 명령과 동작결과를 검색할 수 있는 UI 개발
 * 간단한 report 생성 기능 개발

# 평가 방법
 * 장치 제어 명령 자동 생성 및 전송 여부
 * 펌웨어 동작 상황 모니터링 여부
 * 제어 명령과 펌웨어 동작 상황을 DBMS에 저장 및 검색하는 기능 개발 여부

# Basic Architecture
크게 네 장치들의 상호작용으로 나타낼 수 있다.
## User
사용자 단말은 IoT와 serial하게 연결되어 있으며, 이는 IoT 장치에 장치 제어 명령을 자동으로 생성하고 전달하는 역할을 한다.
## IoT
IoT는 User에서 온 명령에 대해서 동작하고, 이에 따른 결과를 계측기에 보내준다.
## 계측기
장치 제어 명령에 따른 IoT의 동작 상황을 모니터링한다.
## Database
장치 제어 명령에 따른 IoT의 동작 결과를 모두 저장한다.
 + 특정 명령과 동작 결과를 검색할 수 있어야 한다.
 + 간단한 report를 생성할 수 있어야 한다.