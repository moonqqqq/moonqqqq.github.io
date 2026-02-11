---
title: 장시간 비동기 작업의 Race Condition
description: 장시간동안 실행되는 job들이 네트워크 문제로 복수개 실행될때 epoch 처리로 동시성 처리하기.
header: 장시간 비동기 작업의 Race Condition
tags:
 - Nest.js
 - Bullmq
 - Fencing token
---

딥리서치와 같이 오래걸리는 작업을 관리할 때 네트워크 문제로 동시에 복수개의 워커에서 하나의 job이 실행될수가 있다. 이럴 때 같은 목적지의 데이터 저장소에 stream을 쏘게 되면 데이터가 이상해진다.
같은 요청이기때문에 LLM이 비슷한 응답을 한다고 하더라도 아래와 같이.
"I am a boy", "I am a young man" 

두 job 워커가 같은 스트림으로 데이터를 잘라서 쏘기 때문에. 쌓인 데이터의 최종 결과물은 이상해질수 있다.

아래 처럼 데이터가 넘어와서 이상한 결과를 프론트에 전달하게 된다.
```
I             (worker 1)
am a          (worker 1)
I             (worker 2)
boy           (worker 1)
am a          (worker 2)
young man     (worker 2)
```



//// 목차

# 문제 정의
:결과물 난잡해짐.

# 해결 방식 간단 설명
:
1. bullmq 를 이용하여 순차적으로 처리할수 있게 한다.
  bullmq 관련 기능들 필수적인거 다 파악하기.
  - rock
  - rock 연장
  - 좀비워커라는 단어.
2. fenching token or epoch(시대) 관리 프로세스로 이전 시대의 프로세스, 데이터는 무시하는 프로세스를 만든다.
  시대 값 관리는 bullmq에서 주는 몇번째시도인지 값으로 이용.
  시대 값을 전체 서버에서 어떻게 이용할지.
  abortController를 이용해야 좀비프로세스를 빨리 끌수 있음.
  프론트엔드에서도 필터링 혹은 리프래시가 필요하다.
  

펜싱토큰을 확인하는 프로세스는 어디서 할건지. 빈번하면 코드가 난잡해지고 cpu사용량이 늘어날수 있음.