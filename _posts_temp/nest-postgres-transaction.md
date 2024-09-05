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
"1. 기본적인 잠"금(shared/exclusive lock)"과 "2. Isolation level 구현(MVCC)은 데이터베이스에서 이미 완성시켜뒀다. 그 나머지 부분은 프로그래머가 코드로 구현해야만한다.

## 1. 기본 잠금 - Shared Lock, Exclusive Lock
모든 데이터베이스에서는 update, delete문이 실행되는 동안 해당 row를 잠궈버린다. 다른 update, delete문이 같은 row에대해 실행하려한다면 이전 쿼리의 트랜잭션이 끝날때까지 기다려야한다. (INSERT, UPDATE, DELETE문은 트랜잭션을 명시적으로 설정하지않아도 데이터베이스가 알아서 트랜잭션을 실행하고 작업이 완료되면 트랜잭션을 끝낸다. 반대로 기본적으로 아무것도 설정하지않은(트랜잭션 X, for update 설정 X) select문은 트랜잭션도, Lock도 이용하지않는다.)

여기서 쓰이는 잠금은 "Shared Lock"과 "Exclusive Lock"이다.

Shared Lock: 내가 이 데이터 보고 있으니까 데이터 바꾸지마. 보기만 해.
Exclusive Lock: 내가 데이터 수정할거니까 보기만하고 수정하지마..

두가지 잠금 모두 다른 트랜잭션에서 **데이터 수정을 불가능**하게 만든다. **둘의 차이점은 잠금을 걸었을 때 다른 잠금이 동시에 잠금을 걸수 있는지 없는지 여부다.** Exclusive Lock은 이름처럼 어떠한 락도 공존할수 없다. Shared Lock은 이름처럼 공존할수 있다. 다만 공존할수 있는 Lock은 Shared Lock뿐이다.
만약 트랜잭션 A에서 row1에 Shared Lock을 건 상태라면 다른 트랜잭션에서도 row1에 Shared Lock을 걸수 있다. Shared Lock이 겹쳤을 때는 마지막 Shared Lock을 건 트랜잭션이 Lock을 풀면 해당 row의 shared Lock이 풀린다. Exclusive Lock은 동시에 다른 잠금을 걸수 없다.

## 2. MVCC (Multi Version concurrency control)
MVCC는 모든 RDB 데이터베이스에서 통용되는 규칙인 "Isolation Level"을 구현한 것이다.

### MVCC의 논리적 기반
MVCC는 데이터베이스의 기본 저장방식인 **COMMIT**을 이용한다. 데이터베이스는 데이터를 저장, 수정할 때 1차로 메모리에 저장한뒤에 트랜젝션이 커밋되면 DISK에 저장하는 방식이다.
커밋한 데이터만 읽어들일건지, 커밋하지않았지만 메모리에 저장된 데이터도 읽어들일건지에 따라 Isolation Level이 나뉜다.

**Read Committed**는 메모리에만 존재하는 데이터는 읽지않는다.(커밋된 데이터만 읽는다.)
**Repeatable Read**는 메모리에만 존재하는 데이터도 읽는다. 하지만 트랜잭션이 시작된 시점에 존재했던 데이터만 가능하다.

더 자세히 알아보자.

### MVCC의 물리적 기반
MVCC는 snapshot이란 내부 데이터를 활용한다. (데이터베이스 복구할때 이용하는 snapshot과는 이름만 같다.)

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
xmin: 현재 활성화된(실행되고있는) 트랜잭션 중 가장 작은 트랜잭션의 ID
xmax: 현재 활성화된 트랜잭션중 가장 큰 트랜잭션의 ID

모든 row들은 system column으로 xmin, xmax를 가진다. (`SELECT xmin, xmax FROM ANY_TABLE` 을 한번 실행해보면 볼수 있다.)
- 데이터(row)가 생성될 때 xmin으로 현재 트랜잭션 ID가 설정되고 xmax로 0이 설정된다.
- 데이터를 삭제하면 xmax에 현재 트랜잭션 ID가 설정된다.
- 데이터를 수정하면 xmax에 현재 트랜잭션 ID를 설정한다. 그리고 새로운 데이터(row)를 생성하면서 xmin은 현재 트랜잭션 ID, xmax는 0으로 설정된다.

위 3가지 규칙을 이용하여 where문으로 가져온 row들을 다시한번 필터링하게 된다. 필터링 조건은 Isolation level마다 다르다.

#### Hit bits
그리고 하나더 힌트 비트(hint bits)를 보고 넘어가자.

개별 row의 헤더에 힌트 비트를 가지고 있는데 4가지를 가지고 있다.
```md
XMIN_COMMITTED -- creating transaction is known committed
XMIN_ABORTED -- creating transaction is known aborted
XMAX_COMMITTED -- same, for the deleting transaction
XMAX_ABORTED -- ditto

// https://wiki.postgresql.org/wiki/Hint_Bits
```

XMIN_COMMITTED 혹은 XMAX_COMMITTED가 true라면 커밋된 데이터를 뜻한다. where문으로 가져온 rows들에 붙어있는(헤더에 저장돼있는) hit bit를 확인하여 커밋된 데이터인지 아닌지 파악하여 필터링한다.


#### pg_xact(pg_clog) 커밋로그
hint bits의 값이 없다면 해당 row에 대해서 "pg_xact"를 확인한다.

pg_xact는 우리가 흔히 말하는 트랜잭션 로그이다. disk에 따로 저장돼있는 파일로써 트랜잭션에 대한 로그를 저장해둔다. 이파일을 이용하여 트랜잭션들의 상태를 파악할수 있다. pg_xact는 기본적으로 disk에 저장돼있어 느리지만 최신 트랜잭션 로그에 대해서는 메모리에 캐싱해둔어 성능을 높인다.


#### Read Committed에서의 snapshot 이용.
Read Committed는 커밋된 데이터만 보여주는 것이기 규칙이기때문에 XID를 거의 이용하지않는다.
트랜잭션안에서 지금 실행되는 쿼리 시점에서 COMMIT된 데이터만 읽어와서 보여줄 뿐이다. XMIN, XMAX가 이용되는 건 특정 row에 여러번의 수정 작업이 있었다거나 삭제된 기록을 XMAX를 이용하여 필터링한다. XMAX가 0이 아니라면 수정이 발생해서 이전 버전의 데이터거나, 삭제된 데이터를 의미하기때문이다.

트랜잭션안에서 매 statement 시작전에 생성되는 xmin, xmax 정보를 이용하여 "1. where 조건문으로 검색한 데이터", 그리고 "2. 이 데이터를 Read Commiited로 필터링한 데이터"에 다시한번 필터링을 추가한다.

검색된 데이터의 xmin값이 스냅샷의 xmin보다 작거나 같아야만 접근가능한 데이터이다. 크다면 현재 트랜잭션 보다 늦게 시작된 트랜잭션이기때문에 






데이터베이스의 Isolation lever을 MVCC는 데이터베이스에서 알아서ㄴ 해주는 것들이다. 반면에 "낙관적/비관적 락"과 "Avoid dead lock"은 프로그래머가 직접 해야할 일이다.

데이터베이스단에서 직접 동시성 확보를 해주는 것은 Isolation과 MVCC라고 했는데 이 둘이 해주는 일을 간단히 이야기하면 아래와 같다.(postgres를 기준으로)

1. Postgres의 기본 "Isolation" level인 "Read Committed"로 커밋된 데이터만 필터링한다.
2. 커밋으로 한번 필터링 한 뒤에 "MVCC"로 이전 트랜잭션에서 완료된 데이터만 필터링한다.
동시성 관리는 우리가 where로 조건 검색한 뒤에 한번더 필터링하는 과정이라고 생각하면 좋다.

두가지를 좀더 알아보자.

## 1. Isolation
(이 글에서는 PostgresSQL을 기준으로 한다. Isolation level에 대해 모르면 먼저 공부하고 오길 바란다.)
postgresSQL에서는 "Read Committed" level이 기본 설정이다. 서비스의 기능이 Read Committed과 맞지않다면 굳이 바꿀 필요는 없다.

<U>커밋된 데이터만 읽어올 수 있는 "Read Commiited"</U>의 한계는 대표적으로 두가지다. **"Non-repeatable read"**와 **"Phantom read"**. 둘이 비슷하면서도 조금 다르다.

```md
# Non-repeatable Read
하나의 트랜잭션에서 돌일한 row를 검색했음에도 다른 결과가 나온다. 트랜잭션A가 이 id=1인 row를 검색한뒤에 트랜잭션B가 id=1인 row를 변경했다(commit까지 완료). 그 뒤 다시 트랜잭션A가 같은 row를 조회하면 다른 데이터가 나오게 된다.

# Phantom read
하나의 트랜잭션에서 언제는 데이터가 있다가 언제는 없다. 트랜잭션A가 id=1인 row를 검색했을 때는 데이터가 존재했다. 그런데 트랜잭션A가 끝나기 전에 트랜잭션B가 id=1인 데이터를 삭제한다(커밋까지 완료). 그리고 다시 트랜잭션A가 id=1인 데이터를 검색했을 때는 데이터가 존재하지 않는다.
```

이 두가지를 해결하기 위해서 Postgres는 MVCC(Multi Version Concurrrency Control)을 제공한다. (MVCC를 통해 "Non-repeatable read"는 완전히 방지할수 있지만 "Phantom read"는 완전히 절반만 방지가능하다.)

## MVCC
MVCC에서는 **스냅샷**을 이용하여 특정 데이터에 대한 접근 가능성을 판단한다.
스냅샷을 이용하는 흐름은 아래와 같다.

1. 트랜잭션안에서 statement가 실행될때마다 스냅샷을 생성한다.
2. statement(쿼리)를 실행한다.
3. 쿼리의 결과물과 스냅샷을 비교하여(xmin, xmax) 한번 더 데이터를 필터링한다.

<U>스냅샷 생성시기는 Isolation Level에 따라 다르다. Read Committed는 매 statement마다, Serialized, Repeatable Read는 트랜잭션이 시작했을 때만 생성된다.</U>(나는 이걸 몰라서 하루종일 찾았었다. Read Commiitted에서도 트랜잭션 시작 때 스냅샷이 생성되는줄 알았다. 트랜잭션 시작때 스냅샷이 생성되면 Phantom read가 해결돼야하는데 MVCC로는 Phantom read가 해결되지않다니 말이 되지않았다. 반나절이상 정보를 찾다보니 Read Committed에서는 매 statement마다 생성한다는 글을 겨우 찾았다.)

스냅샷이 어떤 형태인지 훑어보자.

```c
typedef struct SerializedSnapshotData
{
	TransactionId xmin;
	TransactionId xmax;
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
이번글에서는 xmin, xmax 이 두가지만 필요하다. 나머지는 직접 찾아보는것도 괜찮다.
xmin: 현재 활성화된 트랜잭션 중 가장 작은 트랜잭션의 ID
xmax: 현재 활성화된 트랜잭션중 가장 큰 트랜잭션의 ID

트랜잭션안에서 매 statement 시작전에 생성되는 xmin, xmax 정보를 이용하여 "1. where 조건문으로 검색한 데이터", 그리고 "2. 이 데이터를 Read Commiited로 필터링한 데이터"에 다시한번 필터링을 추가한다.

검색된 데이터의 xmin값이 스냅샷의 xmin보다 작거나 같아야만 접근가능한 데이터이다. 크다면 현재 트랜잭션 보다 늦게 시작된 트랜잭션이기때문에 







Postgres의 MVCC의 제일 중요한 것은 **XID(트랜잭션 ID)**이다. 매 트랜잭션마다 XID라는 값이 생긴다. 그리고 이 값을 XMIN, XMAX 시스템 필드에 저장하여 데이터 검색에 이용한다. 
	**Read Committed 격리 수준에서는 트랜잭션이 실행하는 각 쿼리마다 새로운 스냅샷이 생성됩니다.**
### XMIN, XMAX
Postgres는 모든 테이블의 시스템 컬럼으로 XMIN, XMAX가 존재한다. 따로 볼수는 없지만 SELECT 쿼리를 실행하면 볼수 있다. (`SELECT xmin, xmax FROM ANY_TABLE` 을 한번 실행해보면 볼수 있다.)

Postgres는 매 트랜잭션마다 생기는 XID를 XMIN와 XMAX에 저장함으로써 실행중인 트랜잭션 시점에 맞는 데이터 검색을 가능하게 한다. XID가 작으면 먼저 실행된 트랜잭션에서 처리된 데이터이고 XID가 크면 나중에 실행된 트랜잭션에서 처리된 데이터다. 이 값을 크기 비교하여 어떤 트랜잭션 데이터까지 열람할수 있는지 조절한다. 현재 진행중인 트랜잭션 이전의 데이터만 접근가능하게 한다.(현재 XID보다 낮은) Query에 따른 XID 설정 방법은 아래와 같다.

```
INSERT문이 실행된다면 XID가 XMIN 값에 저장되고 XMAX는 0이 저장된다.
UPDATE문에서는 기존에 존재하는 row의 XMAX에 현재 XID를 저장한다. 그리고 새로운 row를 만들어서 XMIN에 지금 XID를 저장한다.
DELETE문에서는 기존에 존재하는 row의 XMAX에 현재 XID를 저장한다.
```

현재 XID와 row들의 XMIN, XMAX를 비교함으로써 Read Committed에서 발생하는 non-repeatable read를 예방할수 있게 됐다.

하지만 Phantom read는 아직도 예방해주지 못한다. XMIN, XMAX를 이용하면 해결할 것으로 예상되지만 그렇지 않다.
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


At the Read Committed isolation level, a snapshot is created at the beginning of each transaction statement.

At the Repeatable Read and Serializable levels, the snapshot is created once, at the beginning of the first transaction statement.




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
