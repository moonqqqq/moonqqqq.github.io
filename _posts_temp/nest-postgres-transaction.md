---
title: 백엔드 동시성 - 1. Postgres 가시성
description: 백엔드 동시성 관리
header: 백엔드 동시성 - 1. Postgres 가시성
tags:
 - node.js
 - 백엔드
 - 동시성
 - database
 - transaction
---
`Hit Bits`, `MVCC`, `pg_xact`

동시성 확보 관련해서 프로그래머가 직접 코드단에서 짜야하는 것을 알아보기전에 데이터베이스에서 기본적으로 제공해주는 동시성 확보 방식을 알아본다.

### 목차
```
Database단
    1. DB자체적으로 도와주는 부분
        a. MVCC - Isolation  => 한계가 있다.
    2. DB의 기능을 이용하여 프로그래머가 설정해야하는 부분
        a. 낙관적 락/ 비관적 락
        b. Avoid deadlock
Application단
    1. 분산 락
    2. 메시지 큐
```

# Database단
데이터베이스단에서 동시성 확보는 디비 자체적으로 해주는 것과 프로그래머가 직접 코드로 짜야하는 걸로 나뉜다. 
"1. 기본적인 "잠금(shared/exclusive lock)"과 "2. Isolation level 구현(MVCC)은 데이터베이스에서 이미 완성시켜뒀다. 그 나머지 부분은 프로그래머가 코드로 구현해야만한다.

## 1. 기본 잠금 - Shared Lock, Exclusive Lock
모든 데이터베이스에서는 update, delete문이 실행되는 동안 해당 row를 잠궈버린다. 다른 쿼리중에서 update, delete문이 같은 row에 대해 실행하려한다면 이전 쿼리의 트랜잭션이 끝날때까지 기다려야한다. (INSERT, UPDATE, DELETE문은 트랜잭션을 명시적으로 설정하지않아도 데이터베이스가 알아서 트랜잭션을 실행하고 작업이 완료되면 트랜잭션을 끝낸다. 반대로 기본적으로 아무것도 설정하지않은(트랜잭션 X, for update 설정 X) select문은 트랜잭션도, Lock도 이용하지않는다.)

여기서 쓰이는 잠금은 "Shared Lock"과 "Exclusive Lock"이다.

Shared Lock: 내가 이 데이터 보고 있으니까 데이터 바꾸지마. 보기만 해. 근데 너도 같이 잠글수 있어(shared lock만)<br>
Exclusive Lock: 내가 데이터 수정할거니까 보기만하고 수정하지마..

두가지 잠금 모두 다른 트랜잭션에서 **데이터 수정을 불가능**하게 만든다. **둘의 차이점은 잠금을 걸었을 때 다른 잠금이 동시에 잠금을 걸수 있는지 없는지 여부다.** Exclusive Lock은 이름처럼 어떠한 락도 공존할수 없다. Shared Lock은 이름처럼 공존할수 있다. 다만 공존할수 있는 Lock은 Shared Lock뿐이다.
만약 트랜잭션A에서 row1에 Shared Lock을 건 상태라면 다른 트랜잭션에서도 row1에 Shared Lock을 걸수 있다. Shared Lock이 겹쳤을 때는 마지막 Shared Lock을 건 트랜잭션이 Lock을 풀면 해당 row의 shared Lock이 풀린다. Exclusive Lock은 동시에 다른 잠금을 걸수 없다.

## 2. MVCC (Multi Version Concurrency Control)
MVCC는 대부분의 관계형 데이터베이스에서 통용되는 규칙인 "Isolation Level"을 구현한 것이다.

### MVCC의 논리적 기반
MVCC는 데이터베이스의 기본 저장방식인 **COMMIT**을 이용한다. 데이터베이스는 데이터를 저장, 수정할 때 1차로 메모리에 저장한뒤에 트랜젝션이 COMMIT되면 DISK에 저장한다. (매커니즘에 따라 여러 커밋이 뭉쳐서 disk에 저장되기도 한다.)
커밋한 데이터만 읽어들일건지, 커밋하지않았지만 메모리에 저장된 데이터도 읽어들일건지에 따라 Isolation Level이 나뉜다.

**Read Committed**는 커밋된 데이터만 읽는다.<br>
**Repeatable Read**는 커밋되지 않은 데이터도 읽는다. 하지만 트랜잭션이 시작된 시점에도 존재했던 데이터만 읽는다.

더 자세히 알아보자.

### MVCC의 물리적 기반
Postgres MVCC는 Hint bit, commit log, snapshot, xmin, xmax 데이터를 이용하여 유효한 데이터를 판단한다. 

#### xmin, xmax
먼저 xmin, xmax부터 알야아한다.(hint bit에서도 쓰이고 snapshot에서도 쓰이기 때문에) xmin, xmax는 모든 row가 가지고 있는 컬럼이다. 일반 컬럼이 아니고 시스템 컬럼이라서 명시적으로 검색하지않는 이상 기본적으로는 보이지않는다. INSERT문이 실행되면서 row가 생성될 때 해당 트랜잭션의 ID를 해당 row의 xmin에 설정한다. xmin은 직관적으로 이해가 가지만 xmax는 생소할수 있다. postgres는 기본적으로 update 작업을 할 때 기존 데이터를 변경하지 않는다. 변경하는 대신에 기존 데이터의 xmax에 현재 트랜잭션 ID를 설정한다. 그리고 업데이드된 데이터를 가진 새로운 row를 생성한다. INSERT와 마찬가지로 xmin은 현재 트랜잭션 ID가 설정된다. DELETE는 기존 row의 xmax에 현재 트랜잭션 ID를 설정하고 끝이다. 특정 row에 xmax값이 존재한다면 그 값은 가장 최신의 값이 아니라고 생각할수 있다. (최신의 값이 아닐 뿐이지 유효한 값이 아니라고 생각하면 안된다. Repeatable Read에서는 최신의 값이 중요한게 아니기 때문이다. 아래에서 더 자세히 설명.)
여기서 짚고 넘어가야할 하나는 xmin 값이 있다는게 커밋됐다는 의미는 아니다. row들이 커밋되기전에 임시로 저장되어있듯이 row들의 모든 컬럼들도(시스템컬럼 또한) 임시로 저장된 상태이다.

xmin, xmax를 대략 알았다면 다시 처음으로 돌아가자.

먼저 SELECT문을 WHERE절과 같이 실행하면 

#### 1. WHERE 조건 확인
WHERE절의 조건대로 데이터를 필터링한다.

#### 2.1 커밋 여부 확인 - Hint bit
WHERE 필터링된 row들의 header를 확인하여 hint bit를 확인하여 커밋 여부를 파악한다.

모든 개별 row들은 개별 http 요청들이 헤더를 가지듯이 header를 가지고 있다. header안에 hint bit가 있는데 아래 4가지를 boolean값으로 가지고 있다.
```
XMIN_COMMITTED -- creating transaction is known committed
XMIN_ABORTED -- creating transaction is known aborted
XMAX_COMMITTED -- same, for the deleting transaction
XMAX_ABORTED -- ditto
```

이름만 봐도 알수 있듯이 XMIN_COMMITTED가 true이면 커밋된 것을 의미하고 XMIN_ABORTED가 true라면 롤백된 트랜잭션을 의미한다.

#### 2.2 커밋 여부 확인 - commit log
사실 hint bit는 commit log의 캐시버전이다. 그래서 hint bit부터 확인하고 hint bit가 없다면 커밋 로그를 확인하게 된다.

#### 3. snapshot
mvcc에서 제일 중요한 게 snapshot이다. 커밋된 상태는 어디든 디스크에 바로 저장해두면 되는거라서 특별하다고 할수 없다. snapshot만이 postgres의 특별한 방법이다.(다만 그리 완벽한 매커니즘은 아니라 보완하는 기능이 추가돼있다.) 









먼저 SELECT문에 있는 WHERE절을 실행하여 WHERE조건에 맞는 데이터들만 필터링한다.

#### Hint bit
데이터베이스의 모든 row들은 그들만의 header를 가지고 있는데 그 header안에 hint bit가 존재한다.
```
XMIN_COMMITTED -- creating transaction is known committed
XMIN_ABORTED -- creating transaction is known aborted
XMAX_COMMITTED -- same, for the deleting transaction
XMAX_ABORTED -- ditto
```
위 4가지를 이용하여 where필터링을 거친 데이터들을 한번더 필터링한다. XMIN_COMMITTED가 true이고 XMAX_COMMITED가 false라면 

가져온 row들의 Header에는 hint bit가 존재한다.


Postgres MVCC는 "snapshot"이라는 내부 데이터를 활용한다. (데이터베이스 복구할 때 이용하는 snapshot과는 이름만 같다.)

먼저 MVCC의 스냅샷이 어떤 형태인지 훑어보자.

```c
typedef struct SerializedSnapshotData
{
	TransactionId xmin;     // !
	TransactionId xmax;     // !
	uint32		xcnt;
	int32		subxcnt;
	bool		suboverflowed;
	bool		takenDuringRecovery;
	CommandId	curcid;
	TimestampTz whenTaken;
	XLogRecPtr	lsn;
} SerializedSnapshotData;

// https://github.com/postgres/postgres/blob/06c418e163e913966e17cb2d3fb1c5f8a8d58308/src/backend/utils/time/snapmgr.c
```

일차적으로는 xmin, xmax 이 두가지만 필요하다.

xmin: 현재 활성화된(실행되고있는) 트랜잭션 중 가장 작은 트랜잭션의 ID<br>
xmax: 현재 활성화된 트랜잭션중 가장 큰 트랜잭션의 ID

모든 row들은 system column으로 xmin, xmax를 가진다. (`SELECT xmin, xmax FROM ANY_TABLE` 을 한번 실행해보면 볼수 있다.) 어떤 흐름으로 xmin, xmax가 설정되는지는 아래와 같다.
- 데이터(row)가 생성될 때 xmin으로 현재 트랜잭션 ID가 설정되고 xmax로 0이 설정된다.
- 데이터를 삭제하면 xmax에 현재 트랜잭션 ID가 설정된다.
- 데이터를 수정하면 xmax에 현재 트랜잭션 ID를 설정한다. 그리고 새로운 데이터(row)를 생성하면서 xmin은 현재 트랜잭션 ID, xmax는 0으로 설정된다.

**위 3가지 규칙을 이용하여 where문으로 가져온 row들을 다시 한번 필터링**하게 된다. 필터링 조건은 Isolation level마다 다르다.

#### Hint bits
그리고 하나 더, 힌트 비트(hint bits)를 알아보자.

모든 row는 헤더안에 힌트 비트를 가지고 있는데 4가지를 가지고 있다.
```md
XMIN_COMMITTED -- creating transaction is known committed
XMIN_ABORTED -- creating transaction is known aborted
XMAX_COMMITTED -- same, for the deleting transaction
XMAX_ABORTED -- ditto

// https://wiki.postgresql.org/wiki/Hint_Bits
```

**XMIN_COMMITTED 혹은 XMAX_COMMITTED가 true라면 커밋된 데이터를 뜻한다.** where문으로 가져온 rows들에 붙어있는(헤더에 저장돼있는) hint bit를 확인하여 커밋된 데이터인지 아닌지 파악하여 또 다시 필터링한다.

#### pg_xact(pg_clog) 커밋로그
hint bits의 값이 없다면 해당 row에 대해서 "pg_xact"를 확인한다.

pg_xact는 우리가 흔히 말하는 트랜잭션 로그이다. disk에 따로 저장돼있는 파일로써 트랜잭션에 대한 로그를 저장해둔다. 이 파일을 이용하여 트랜잭션들의 상태를 파악할 수 있다. pg_xact는 기본적으로 disk에 저장돼있어 느리지만 최신 트랜잭션 로그에 대해서는 메모리에 캐싱해두어 성능을 높인다.

#### Read Committed에서의 snapshot 이용.
Read Committed는 커밋된 데이터만 보여주는 것이기 규칙이기때문에 XID를 거의 이용하지않는다.
트랜잭션안에서 지금 실행되는 쿼리 시점에서 COMMIT된 데이터만 읽어와서 보여줄 뿐이다. XMIN, XMAX가 이용되는 건 특정 row에 여러번의 수정 작업이 있었다거나 삭제된 기록을 XMAX를 이용하여 필터링한다. XMAX가 0이 아니라면 수정이 발생해서 이전 버전의 데이터거나, 삭제된 데이터를 의미하기때문이다.

트랜잭션안에서 매 statement 시작전에 생성되는 xmin, xmax 정보를 이용하여 "1. where 조건문으로 검색한 데이터", 그리고 "2. 이 데이터를 Read Commiited로 필터링한 데이터"에 다시한번 필터링을 추가한다.

검색된 데이터의 xmin값이 스냅샷의 xmin보다 작거나 같아야만 접근가능한 데이터이다. 크다면 현재 트랜잭션 보다 늦게 시작된 트랜잭션이기때문에 


==============
==============


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


===============
==============

https://squarelab.co/blog/prisma-transactions/
postgres lock - https://medium.com/@hnasr/postgres-locks-a-deep-dive-9fc158a5641c
isolation level - https://mangkyu.tistory.com/299
낙관적/비관적 락 
- https://cookie-dev.tistory.com/30
- https://sabarada.tistory.com/175

- postbres mvcc https://blog.ex-em.com/1663
https://devcenter.heroku.com/articles/postgresql-concurrency
- postgres snapshot https://postgrespro.com/blog/pgsql/5967899
https://postgrespro.com/blog/pgsql/5967899
- postgres snapshot data https://jnidzwetzki.github.io/2024/04/03/postgres-and-snapshots.html


MVCC는 스냅샷을 통해 

일반적으로 공유 락은 읽기 락이고, 베타 락은 쓰기 락이다. 따라서 베타 락에서 읽기는 가능하고, 쓰기만 불가능하다. 이렇게 널리 퍼져있지만 이는 틀린 말입니다(트랜잭션 격리 수준에 따라 베타 락에서도 읽기가 불가능 할 수 있음).

공유 락과 베타 락의 핵심은 읽기 쓰기가 아니라 락 사이 호환 여부입니다.

공유 락은 같은 공유 락을 허용하기 때문에 읽기가 가능하고, 쓰기에 필요한 베타 락을 불허하기 때문에 쓰기가 불가능 한 것이며, 베타 락은 락을 사용하는 읽기(Locking Reads)인 경우에는 불가능하며 락을 사용하지 않는 읽기(Consistent Nonlocking Reads)인 경우에는 가능합니다.

https://javamix.tistory.com/entry/PostgreSQL-Concurrency-With-MVCC


데이터베이스 동시성 관리
데이터베이스는 제각각 Isolation level을 자기만의 방법으로 구현한다. postgres를 파악해보자~

1. MVCC가 뭔지
2. MVCC를 무엇을 이용해서 구현하나. 
    1. 스냅샷
        - 스냅샷의 내용물 설명
        - 내용물이 어떻게 이용되는지.
3. default인 read committed
    - 한계가 있지만 이용해도 되는 이유.
    - 극복할 방법 
        - 낙관적락(특정테이블 변화확인)
        - select에 비관적락 설정

4. Repeatable read는 뭐가 다른가?
    - non-repeatable read를 해결해준다.
        - 이거 해결해서 뭐가 좋은데? 어떤 실제 문제가 발생하는데?
    - repeatable read도 read committed와 거의 같은 문제가 발생함.

5. 둘다 문제가 있으면 어떡하라고? serialzae쓰라고?
    1. 아까 말한 낙관적락, 비관적락이 중요하다.

6. plus alpha
    4. avoid deadlock
        •	교착 상태 탐지 및 복구: 교착 상태를 탐지하고 복구하는 메커니즘을 구현하여 시스템이 멈추는 것을 방지합니다.
        •	교착 상태 예방: 자원을 정렬된 순서로 획득하게 하거나, 타임아웃을 설정하여 교착 상태를 사전에 예방합니다.
        - 데드락 타임아웃 짧게 설정 innodb_lock_wait_timeout
5. 애플리케이션 락, 메시지 큐
