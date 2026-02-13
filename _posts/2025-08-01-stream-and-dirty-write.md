---
title: 딥리서치 스트림 - 재실행 보장과 Dirty Write
description: 장시간동안 실행되는 job들이 네트워크 문제로 복수개 실행될때 epoch 처리로 동시성 처리하기.
header: 딥리서치 스트림 - 재실행 보장과 Dirty Write
tags:
 - Nest.js
 - Bullmq
 - Fencing token
---

LLM 딥리서치 기능은 장시간 Stream으로 응답을 전달해줘야하면서도 무조건 성공해야하는 작업에 속한다.
그래서 Queue로 작업들을 안전하게 보관하면서 작업이 실패했을 때는 성공할때까지 재실행을 해야한다. UI적으로도 오래걸린다는걸 표현하고 있기때문에 빠르게 응답을 할 필요도 딱히 없다.
하지만 여기서 문제는 **"어떻게든 성공해야한다"라는 측면에서 발생한다. Queue를 통해 재실행을 관리하다보면 하나의 job 동시에 여러군데서 실행되는 일이 발생한다.**

1. Queue에 있는 작업들을 Worker A에서 가져가서 실행한다.
2. Worker A에 일시적인 네트워크 장애가 발생해 Queue가 해당 워커를 죽었다고 판단한다. 
3. Worker A가 죽었다고 판단했기때문에 새로운 Worker B에 동일한 job을 할당한다. 
4. 그런데 기존 Worker A의 네트워크 문제는 일시적인 것이었고 기존 작업은 계속 진행되고 있었다. 

이렇게 되면 동일한 job을 동시에 실행하고 있기때문에 Worker A와 Worker B의 딥리서치 진행 상황이 동일한 Stream 저장소로 전달된다. 이렇게 되면 클라이언트(프론트엔드)에서 받게 되는 진행 상황은 아래와 같이 보일수 있다.

기본 프로세스가 [시작, 문제파악, 분석중, 웹검색, 정리, 완료] 라고 한다면 두군데서 보내기때문에

>[시작, 문제파악, 분석중, <u>시작</u>, <u>문제파악</u>, 웹검색, 정리, <u>분석</u>, <u>웹검색</u>, <u>정리</u>, <u>완료</u>, 완료]

이렇게 꼬여버릴수 있다. 진행 과정 스트림이 이렇게 꼬이는 것뿐만 아니라, 딥리서치 결과물도 누적되는 데이터로 관리한다면 결과물 또한 데이터가 이상해진다.
LLM 기능 자체가 **하드웨어 제약이 존재하고(Rate limit)** 오래걸리는 작업이기때문에 **대기열**은 필수인데 **어떠한 대기열 프로그램/라이브러리들도 Exactly Once 보장해주지않기때문에** Race condition, dirty write, 멱등성 문제는 복합적으로 관리해줘야한다.

Stream 관련 Dirty Write 문제를 해결하기 위해선 대기열과 함께 필요한 개념이 크게 2가지 있다.

**"Fencing Token"** 낙관적 락이라고 생각하면 된다. job을 실행할 때 해당 job이 몇번째 실행됐는지를 알려주는 Token을 이용해서 가장 최근에 실행된 작업만 살아남게 하기 위해 사용된다.

**"Abort controller"** 백엔드 서버에서 LLM 딥리서치 서버로 요청을 했을 때 백엔드 서버에서 원할때 딥리서치 서버에서 진행중인 요청을 중간에 끊게 할수 있는 도구다.

하나하나 알아보자.

<br>

## Fencing Token
데이터베이스의 낙관적락 기능과 같다. 

1. 특정 job이 개별 worker에서 실행되기 시작할때 먼저 Fencing Token을 얻는다. 이 토큰은 해당 job이 몇번째 실행되는지에 대한 값이다. 
2. 거의 동시에 두군데의 워커에서 동일한 job을 실행하게 되면 1, 2의 값을 각각 가지게 된다. 이 값을 가지고 각각 job을 실행하는 도중 정기적으로 FencingToken을 검사하는 시간을 가진다. 레디스에는 가장 큰 값이 저장되있기때문에 해당 값과 비교하여 그보다 작으면 바로 종료한다. (좀비프로세스 종료)

Fencing Token 기능을 구성하는 것들은 아래와 같다.
```typescript
// fencingToken 관리 클래스 메서드 목록. // pseudo code
class fencingTokenManager {
	// 토큰 생성
	acquire(jobId: string) => token;
    
	// 최신 토큰일 경우에만 true를 응답함.
	validate(jobId: string, token: number) => boolean;
    
	// 토큰이 더이상 필요없을때(작업 완료)
	release(jobId: string) => void;
}
```

**acquire()** 메서드는 여러 worker가 동시에 실행해도 레디스를 이용하기때문에 **원자성이 보장**이 된다.
**validate()** 메서드는 가장 늦게 만들어진 토큰인지 검증하는 메서드로 개별 worker가 job을 실행하는동안 정기적으로 호출된다. valid하지 않다면 해당 worker의 job 실행을 즉시 종료시키는 플로우를 타게 만들어야한다. 알아서 정기적으로 실행되는 프로세스로 만듦으로써 코드가 구조를 깔끔하게 유지될수 있다.
**release()** 메서드는 불필요한 리소스 제거를 위해 기본적으로 필요한 메서드다.

위 메서드들은 Worker 코드 내에서 아래처럼 활용된다. 제일 중요한 점은 개별 프로세스 내에서 정기적으로 fencingToken이 유효한지 확인하는 과정을 거친다는 점이다.
```ts
// pseudo code
runningJobs = jobIds[];

process(딥리서치_job) {
	jobId = 딥리서치_job.getId();
	jobData = 딥리서치_job.getData();

	// 1. 개별 워커에 동일한 요청이 두번올수도 있어서 확인.
	if (runningJobs.has(jobId) throw 프로세스중단;

	// 2. fencingTokenManager.acquire(jobId)
	fencingToken = fencingTokenManager.acquire(jobId)
    
	// 3. abortController를 생성한다.
	abortController = new AbortController();

	// ** 30초마다 정기적으로 이 프로세스가 유효한지 확인한다. **
	interval = setInterval(async () => {
		const tokenOK = fencingTokenManager.validate(jobId, fencingToken);
		
		if (!tokenOK) {
			// abortController로 LLM 서버와 연결 종료. 
			abortController.abort(); // 이부분은 아래서 설명할 예정.
		}

		extendsLock(jobId);
	}, 30) // 30초마다 확인.

	try {
		publish('start', llmDeepResearchStartData);
      
		excueteLlmDeepResearch(job, abortController);
      
		publish('end', llmDeepResearchStartData);
	} catch (error) {
		if (abortController.signal.aborted == true) {
			throw 프로세스끝내기.
		}
	} finally {
		clearInterval(interval);
		runningJobs.delete(jobId);
	}
}
```

**heartbeat기능안에 FencingToken을 정기적으로 실행해두고 unvalid할 시 abortController를 실행시키면 좀비 프로세스는 알아서 종료시키게 된다.**

<br>


## AbortController
>AbortController 인터페이스는 하나 이상의 웹 요청을 취소할 수 있게 해준다. - [MDN](https://developer.mozilla.org/ko/docs/Web/API/AbortController)

요청 받은 곳에서도 클라이언트가 요청을 끊고 싶다는 걸 알려주는 용도로 사용된다. 보통 받는쪽에서 더이상 데이터를 안받으면 되긴하는데. llm 요청들은 오래걸리는 작업들이고. 토큰은 다 돈이라서 안받고 넘겨버리면 비용적으로 너무 아깝다. 그래서 양쪽에서 끊을수 있도록 AbortController로 양측에서 취소되게 처리해줘야한다.

**API요청을 할때 AbortController를 같이 보내고 abortController.abort() 메서드가 실행되면 LLM 서버에서 해당 요청을 바로 종료한다.** 그렇게 되면 양쪽에서 요청이 끝나는거기때문에 토큰의 문제도 없고 좀비프로세스도 데이터를 더이상 받지 않기때문에 더이상 데이터를 쏠 수 없게 돼버린다.(당연하게도 LLM 서버에서도 AbortController 기능을 구현해둬야한다.)

<br>

## 프론트엔드에서도 처리
그리고 마지막으로 중요한 게 있다. 프론트엔드단에서도 같이 Dirty Write를 컨트롤해줘야한다.
프론트단에서도 FencingToken을 관리하여서 이전 시대(epoch)의 데이터는 유저에게 보여주지않고 가장 최신 시대(epoch)의 데이터만 보여줘야한다.
Worker A로 스트림을 받고 있다가 좀비프로세스가 되어 Worker B에서 새로 작업을 시작하게 된다면 프론트엔드에서 Worker B의 데이터만 보이도록 화면을 리셋해줘야한다.

**오래걸리는 작업을 진행할때 + 실패를 허용하지않을 때는 재실행 프로세스**에서는 백엔드만으로는 완벽한 동시성을 제어할수가 없다. 프론트로 이미 보내진거는 무를수 없으니말이다.

하나의 작업이 도중 오류가 발생해서 Worker A, Worker B **여러군데서 실행됐다고 하더라도 SSE 연결은 쭉 하나의 API로 진행됐던 것이기때문에 해당 스트림 연결에서 Reset 이벤트를 전달하면 된다.**

아래 코드처럼 백엔드에서 RESET 이벤트를 전달한다. 
```ts
@Sse('stream/:jobId')
  streamResults(@Param('jobId') jobId: string): Observable<MessageEvent> {
    return new Observable((subscriber) => {
      let currentFencingToken: number = 0;

      const unsubscribe = this.streamService.subscribe(jobId, (data) => {
        const incomingToken = data.fencingToken ?? 0;

        if (incomingToken > currentFencingToken) {
          if (currentFencingToken > 0) {
            console.log(`🔄 Stream: Token changed ${currentFencingToken} → ${incomingToken}`);
            subscriber.next({    // ** RESET 이벤트 전송 **
              data: JSON.stringify({
                status: 'reset',
                message: 'New worker took over',
                token: incomingToken	// ** FencingToken 도 전달 **
              }),
            } as MessageEvent);
          }
          currentFencingToken = incomingToken;
        }

        // 현재 토큰보다 오래된 데이터는 무시
        if (incomingToken < currentFencingToken) {
          console.log(`🚫 Stream: Ignoring stale token ${incomingToken} < ${currentFencingToken}`);
          return;
        }

        subscriber.next({ data: JSON.stringify(data) } as MessageEvent);

        if (data.status === 'completed' || data.status === 'error') {
          subscriber.complete();
        }
      });

      // 클라이언트 연결 종료 시 구독 해제
      return () => unsubscribe();
    });
  }
```

### 프론트엔드 처리
1. 그리고 프론트엔드에서는 Reset 이벤트를 받고 보여주던 데이터들을 새로운 worker에서 오는 데이터들만 처리해주면 된다. 
2. 그리고 좀비프로세스가 죽을 때까지 전송이 계속될수 있기때문에 가장 높은 fencingToken 보다 낮은 값이 들어오면 무시해줘야 한다.

<br>

지금까지 나열된 내용은 LLM 딥리서치 기능에는 잘맞는 구조다. 같은 질문을하더라도 LLM은 결과가 달라지기때문에 두개의 결과가 섞이면 안된다. 하지만 다른 작업들(배치같은?)은 여러번 실행해도 결과가 같은 경우가 있기 때문에 이럴 때는 지금 해결책에서 몇개빼서 진행해도 된다.