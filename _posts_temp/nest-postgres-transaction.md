prisma.js와 postgresSQL을 이용할 때의 트랜잭션 처리



자동화 서버에서는 여러 job process 서버에서 같은 테이블을 검색한뒤 job을 실행하는 일이 많다.

이때 kafka consumer면 되는구나.

kafka prodcer는 하나만 있나?


"for update" 를 쓰면 잠기나보네?

SELECT문은 잠기지않는다.