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

