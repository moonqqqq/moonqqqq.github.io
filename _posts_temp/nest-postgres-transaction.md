---
title: 백엔드 동시성 관리 - 1. Database
description: 백엔드 동시성 관리
header: 백엔드 동시성 관리
tags:
 - node.js
 - 백엔드
 - 동시성
 - database
 - transaction
---

Database단
    1. Isolation
        postgres는 read committed
            - 한줄 설명
            - 한계/단점 파악
            - 단점 예방 방법
                => MVCC가 도와준다.
    2. MVCC
        - 간단 설명
        - XID(XMIN,XMAX)를 이용하여 visibility를 정한다.
        - 이것마저 한계라면 3번의 도움을 받는다.
        isolation만 있을 때의 데이터 보호와 mvcc가 합채됐을 때 데이터 보호
        한계를 도와주는 낙관적 락/비관적 락
    3. 낙관적 락/ 비관적 락
        설명
        어떤점을 도와주는지.
        어플리케이션 단이라 귀찮은지?
        실제로 쓰는 데가 있는지.
    4. Avoid deadlock
        •	교착 상태 탐지 및 복구: 교착 상태를 탐지하고 복구하는 메커니즘을 구현하여 시스템이 멈추는 것을 방지합니다.
        •	교착 상태 예방: 자원을 정렬된 순서로 획득하게 하거나, 타임아웃을 설정하여 교착 상태를 사전에 예방합니다.
        - 데드락 타임아웃 짧게 설정 innodb_lock_wait_timeout


백엔드에서 필요한 동시성 관리가 어떻게 진행되는지 정리해보자. 흔히 이용하는 인프라 자체에서 해주는 것과 프로그래머가 직접해야하는 것으로 나뉜다.

목차는 아래와 같다. 특이점은 OS단에서의 동시성 문제는 포함하지않았다. node.js(싱글스레드) 환경에서는 흔히 발생하지않기때문이다. 그리고 메시지큐에 대해서도 다룰건데 메시지큐 방식을 이용하면 싱글스레드에서 발생할 수 있는 대부분의 OS 동시성 문제를 예방가능하다.

### 목차
Database단
    1. Isolation
    2. MVCC
    3. 낙관적 락/ 비관적 락
    4. Avoid deadlock

Application단
    1. 분산 락
    2. 메시지 큐

# Database단
데이터베이스단에서 동시성 확보는 디비 자체적으로 해주는 것과 프로그래머가 직접 해줘야하는게 반반이다. Isolation, MVCC는 데이터베이스에서 알아서 해주는 것들이다. 반면에 "낙관적/비관적 락"과 "Avoid dead lock"은 프로그래머가 직접 해야할 일이다.

## 1. Isolation

###postgres는 read committed###

(이 글에서는 PostgresSQL을 기준으로 한다. Isolation level에 대해 모르면 먼저 공부하고 오길 바란다.)
postgresSQL에서는 "Read Committed" level이 기본 설정이다. 서비스의 기능이 Read Committed가 맞지않다면 굳이 바꿀 필요는 없다.

<U>커밋된 데이터만 읽어올 수 있는 "Read Commiited"</U>의 한계는 대표적으로 두가지다. **"Non-repeatable read"**와 **"Phantom read"**. 둘이 비슷하면서도 조금 다르다.

```md
# Non-repeatable Read
하나의 트랜잭션에서 돌일한 row를 검색했음에도 다른 결과가 나온다. 트랜잭션A가 이 id=1인 row를 검색한뒤에 트랜잭션B가 id=1인 row를 변경했다(commit까지 완료). 그 뒤 다시 트랜잭션A가 같은 row를 조회하면 다른 데이터가 나오게 된다.

# Phantom read
하나의 트랜잭션에서 언제는 데이터가 있다가 언제는 없다. 트랜잭션A가 id=1인 row를 검색했을 때는 데이터가 존재했다. 그런데 트랜잭션A가 끝나기 전에 트랜잭션B가 id=1인 데이터를 삭제한다(커밋까지 완료). 그리고 다시 트랜잭션A가 id=1인 데이터를 검색했을 때는 데이터가 존재하지 않는다.
```
이 두가지를 해결하기 위해서 postgres는 MVCC(Multi Version Concurrrency Control)을 제공한다.

## MVCC
Postgres의 MVCC의 제일 중요한 것은 **XID(트랜잭션 ID)**이다. 매 트랜잭션마다 XID라는 값이 생긴다. 그리고 이 값을 XMIN, XMAX 시스템 필드에 저장하여 데이터 검색에 이용한다. 

### XMIN, XMAX
Postgres는 모든 테이블의 시스템 컬럼으로 XMIN, XMAX가 존재한다. 따로 볼수는 없지만 SELECT 쿼리를 실행하면 볼수 있다. (`SELECT xmin, xmax from ANY_TABLE` 을 한번 실행해보면 볼수 있다.)

Postgres는 매 트랜잭션마다 생기는 XID를 XMIN와 XMAX에 저장함으로 시점에 맞는 데이터 검색을 가능하게 한다.
XID가 적으면 먼저 실행된 트랜잭션이고 XID가 크면 나중에 실행된 트랜잭션이다

INSERT문이 실행된다면 XID가 XMIN 값에 저장되고 XMAX는 0이 저장된다.
UPDATE문에서는 기존에 존재하는 row의 XMAX에 현재 XID를 저장한다. 그리고 새로운 row를 만들어서 XMIN에 지금 XID를 저장한다.
DELETE문에서는 기존에 존재하는 row의 XMAX에 현재 XID를 저장한다.


일단 기본적으로 INSERT문이 실행되면 트랜잭션이 실행되기때문에 XID가 생성된다. 그리고 생성하는 row데이터의 XMIN 필드에 
XID를 XMIN, XMAX에 저장함으로써 버전 관리하듯이 트랜잭션 상황에 맞는 특정 버전의 데이터를 가져올수 있다. 

### What is XMIN, XMAX?
XMIN, XMAX는 [System Column](https://www.postgresql.org/docs/current/ddl-system-columns.html)이다. 모든 테이블에 존재하며 데이터 검색시 DBMS에 의해 자동으로 조건이 셋팅된다. 

**어떻게 설정되고 어떻게 이용되는지**




//////////////////////////////////////////
//////////////////////////////////////////



Database단
    1. Isolation
        postgres는 read committed
            - 한줄 설명
            - 한계/단점 파악
            - 단점 예방 방법
                => MVCC가 도와준다.
    2. MVCC
        - 간단 설명
        - XID를 이용하여 visibility를 정한다.
        - 이것마저 한계라면 3번의 도움을 받는다.
        isolation만 있을 때의 데이터 보호와 mvcc가 합채됐을 때 데이터 보호
        한계를 도와주는 낙관적 락/비관적 락
    3. 낙관적 락/ 비관적 락
        설명
        어떤점을 도와주는지.
        어플리케이션 단이라 귀찮은지?
        실제로 쓰는 데가 있는지.
    4. Avoid deadlock
        •	교착 상태 탐지 및 복구: 교착 상태를 탐지하고 복구하는 메커니즘을 구현하여 시스템이 멈추는 것을 방지합니다.
        •	교착 상태 예방: 자원을 정렬된 순서로 획득하게 하거나, 타임아웃을 설정하여 교착 상태를 사전에 예방합니다.
        - 데드락 타임아웃 짧게 설정 innodb_lock_wait_timeout

Application단
    1. 분산 락
    2. 메시지 큐

Database
- isolation
- transaction
- 명시적 락
- 낙관적 락 / 비관적 락
- MVCC
- 데드락 피하기
	•	교착 상태 탐지 및 복구: 교착 상태를 탐지하고 복구하는 메커니즘을 구현하여 시스템이 멈추는 것을 방지합니다.
	•	교착 상태 예방: 자원을 정렬된 순서로 획득하게 하거나, 타임아웃을 설정하여 교착 상태를 사전에 예방합니다.
    - 데드락 타임아웃 짧게 설정 innodb_lock_wait_timeout

Application
- 스레드 동기화 => node.js는 필요 없음. event loop
- 분산 락
- 메시지큐



보통 


https://choi-geonu.medium.com/%EB%B0%B1%EC%97%94%EB%93%9C-%EA%B0%9C%EB%B0%9C%EC%9E%90%EB%93%A4%EC%9D%B4-%EC%95%8C%EC%95%84%EC%95%BC%ED%95%A0-%EB%8F%99%EC%8B%9C%EC%84%B1-5-continuation-passing-style-5058ab5cb781

//// 아래는 postgres로 글쓰려고 했을 때 적었던 것들.

포스트그래스는 READ_COMMIT이 기본이다. 대부분의 애플리케이션에서 충분한 동시성과 일관성을 제공하기 때문에 많이 사용됩니다.
이 대부분의 경우가 뭔지

read committed인데 이 격리수준에는 한계가 있다.
phantom read, non-repeatable read두가지.
- 코드로 따라해보기

비관락: 수정 비율이 높은 앱
- exclusive lock
- shared lock - 여러사용자가 읽기 가능, update, delete 작업 금지.
낙관락: 읽기 비중이 높은 앱

포스트그래스는 팬텀 리드를 못막는다. 그래서 명시적 잠금을 사용해야함.

 일반적으로, 트랜잭션 범위를 최소화하고, 필요한 최소한의 락을 사용하는 것이 바람직합니다.

page lock, row lock 모든 디비는 기본이 row lock인가? page lock도 있을듯

데드락을 피하기 위해 트랜잭션 순서를 고정시켜야한다.

shared lock과 exclusive lock
 - select for update
 - select for share

prisma.js와 postgresSQL을 이용할 때의 트랜잭션이 어떻게 처리되는지 알아보자.

prisma.js에서는 아래와 같은 모양으로 트랜잭션을 처리한다.
```ts
this.prisma.$transaction(async (tx) => {
    await tx.user.findFirst(~)
    await tx.order.update(~)
})
```

나도 처음에는 대충 이렇게하면 안정성이 보장되겠지 했지만 알고보니 이걸로는 많이 부족했다.

Shared Lock에서 문제가 발생할 수 있는 상황

- 하지만, 만약 두 개 이상의 트랜잭션이 데이터를 읽은 후 수정하려고 할 때 문제가 발생할 수 있습니다.

이유는
prisma.someEntity.findFirst() 는 단순 "SELECT"문이라서 lock(shared)을 걸지않는다.

자동화 서버에서는 여러 job process 서버에서 같은 테이블을 검색한뒤 job을 실행하는 일이 많다.

이때 kafka consumer면 되는구나.

kafka prodcer는 하나만 있나?


"for update" 를 쓰면 잠기나보네?

SELECT문은 잠기지않는다.



https://squarelab.co/blog/prisma-transactions/
postgres lock - https://medium.com/@hnasr/postgres-locks-a-deep-dive-9fc158a5641c
isolation level - https://mangkyu.tistory.com/299
낙관적/비관적 락 
- https://cookie-dev.tistory.com/30
- https://sabarada.tistory.com/175

- postbres mvcc https://blog.ex-em.com/1663
https://devcenter.heroku.com/articles/postgresql-concurrency