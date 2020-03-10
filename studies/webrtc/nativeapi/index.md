---
title: "Native API for WebRTC"
permalink: /studies/webrtc/
excerpt: "Native API for WebRTC"
layout: single
last_modified_at: 2019-08-20T21:36:11-04:00
redirect_from:
  - /theme-setup/
toc: true
sidebar:
  nav: "webrtc"
---
WebRTC's motivation was to make peer-to-peer streaming in browsers. **However**, WebRTC is no longer limited to browsers; in fact, it is available in Android, IOS, and even in desktops by building native C++ APIs.

In this document, we will discuss about the native C++ APIs of WebRTC.

# Basics of Native API
Javascript에서 API를 호출하면, 브라우저 내부에서는 Native API를 call해서 이를 처리한다. 