---
title: API 서버가 데이터를 받기까지 - 서버가 작동하는 흐름 1
description: 클라이언트의 요청이 서버도착하기 전까지의 흐름.
header: API 서버가 데이터를 받기까지 - 서버가 작동하는 흐름 1
tags:
 - node.js
 - TCP/IP
 - OS
---

#### 시리즈 목차
1. [서버는 데이터를 받아서 처리하고 응답값을 반환하는 곳이니 데이터를 받는 과정부터 정리한다.](https://moonqqqq.github.io/before-arriving-server-server-series-1)
2. <U>[[현재글]서버가 받은 데이터가 어떻게 처리되는지. Node.js만의 방식이 어떤지 정리한다.](https://moonqqqq.github.io/from-the-libuv-server-series-1)</U>
3. Node.js만의 방식을 하나하나 조금 더 디테일하게 - Event loop, Libuv, V8, JS


일단 Node.js 서버가 가지고 있는 receive socket buffer는 하나가 아니다. 여러 클라이언트와 연결되기때문에 연결되는 클라이언트 수만큼 소켓이 생기게 된다. 이렇게 되면 관리해야하는 socket이 꽤나 많아지는데 

