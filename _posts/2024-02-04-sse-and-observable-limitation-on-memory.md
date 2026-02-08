---
title: Node.js의 스트림 배압 처리 고도화 (Observable의 한계점)
description: see 데코레이터와 Observable만을 이용할때의 한계점.
header: Node.js의 스트림 배압 처리 고도화 (Observable의 한계점)
tags:
 - Nest.js
 - Stream
 - Observable
 - RxJS
---


LLM 응답 스트리밍 서버가 OOM(Out of memory)으로인해 꺼져버리는 현상이 간간히 로그에 찍혀있었다.

기존 서버는 llm 응답값을 sse로 스트리밍해주는 서버다. 단순히 @Sse 데코레이터에 Observable을 리턴하는 형식으로 구현되어있다. 단순화하면 아래 구조로 표현할 수 있다.

>Stream API server --(subscribe)--> Redis pubsub  <--(push)-- llm 워커에서 llm 응답값 전달

@Sse가 알아서 헤더 설정도 해주고 응답값도 잘 정리해주고, 커넥션 에러도 잘 처리해주기때문에 초기 구현에는 좋았지만. 트레픽이 몰릴때는 백프레셔 문제가 발생했다. 아래 코드는 아주 간단한 @Sse 구현 코드다.

```typescript
@Sse('chat-message/:messageId')
  getLlmAnswer(@Param('messageId') messageId: string): Observable<MessageEvent> {
    return this.redisService.subscribe(`llm:answer:${messageId}`).pipe(
      map((message) => ({ data: message })),
    );
  }
```

위 서버에 동시에 많은 트레픽을 쏴버리면 아래처럼 OOM 에러가 발생하면서 서버가 종료된다.

```
server-1  | ======================
server-1  | === Memory Monitor ===
server-1  | Heap Used: 46MB
server-1  | Heap Total: 59MB
server-1  | RSS: 114MB
server-1  | External: 5MB

~

server-1  | ======================
server-1  | === Memory Monitor ===
server-1  | Heap Used: 155MB
server-1  | Heap Total: 199MB
server-1  | RSS: 246MB
server-1  | External: 17MB
server-1  | ======================
server-1  | === Memory Monitor ===
server-1  | Heap Used: 165MB
server-1  | Heap Total: 210MB
server-1  | RSS: 244MB
server-1  | External: 5MB
server-1  | ======================
server-1  | === Memory Monitor ===
server-1  | Heap Used: 174MB
server-1  | Heap Total: 222MB
server-1  | RSS: 261MB
server-1  | External: 13MB
server-1  | ======================
server-1  | === Memory Monitor ===
server-1  | Heap Used: 183MB
server-1  | Heap Total: 234MB
server-1  | RSS: 278MB
server-1  | External: 21MB
server-1  | ======================
server-1 exited with code 137  # OOM 발생
```

몰린 트레픽으로 인해 메모리 사용량이 확 올라 서버가 갑자기 꺼져버린다.

## Observable의 한계

1. Observable은 백프래셔가 제대로 구현돼있지 않다.

Node.js Stream은 메모리를 적당량만 꾸준히 이용하는 선에서 안정성을 유지한다고 알고있었는데 Observable 내부를 까보니 단순히 Observable에 데이터 소스를 연결해두면 **Observable 내부 버퍼에 데이터가 계속 쌓이는 문제**가 발생한다. 클라이언트가 데이터를 정상적으로 받고 있다면 문제가 되지않지만 클라이언트에서 데이터를 받을수 없는 상황이 오면 데이터를 멈추지않고 메모리에 계속 쌓는 상황이었다. 그러다 OOM 이 발생했다.

Observable이 작동하는 방식을 파악해보니 아래와 같았다.

```
데이터소스 -> observer.next() -> [RxJS 전용(concatMap) 버퍼] -> [WritableStream 버퍼] -> [Socket 버퍼]
```

Node.js 자체적으로 
1. TCP socket 버퍼에 데이터가 꽉차면 writableStream 버퍼에서 데이터를 안보내게 되고. 
2. WritableStream 버퍼에 데이터가 꽉차면 RxJS 전용 버퍼에서 데이터를 안보내게 된다. 
3. **하지만 RxJS의 버퍼는 꽉찼는지 아닌지에 대한 기준이 존재하지않았다.** 
그러다보니 당연하게도 Out Of Memory가 발생할수 밖에 없었다. 클라이언트쪽이던 서버쪽이던 네트워크문제가 생긴다면 어쩔수 없는거였다.

### Push 매커니즘이기때문에
이 문제의 근본 원인은 Observable, RxJS 자체가 Push 매커니즘이기때문이라고 한다.
Observable 다큐먼트를 들어가면 뜬금없게도 pull, push 에 대한 내용부터 나온다. [Observable Document](https://rxjs.dev/guide/observable)

**Push** 매커니즘은 데이터 송신측에서 보낼 수 있다면 어떠한 상황도 상관없이 그냥 보내는 구조.
**Pull** 매커니즘은 데이터 수신측에서 받고 싶을때만 받도록 하는 구조다. (Kafka가 그래서 안정성이 좋다고들한다)

### 왜 push 매커니즘을 사용하는지
근본적으로 메모리에 대해선 불안전한 편인데 왜 Push 매커니즘을 쓰는지 찾아보니 RxJS자체가 프론트엔드 이벤트 처리용으로 만들어졌기때문이라고 한다. 이벤트가 언제 발생한지 알수 없기때문에 pull 방식은 불가능하다. JS가 callback을 이용하는 이유와 거의 같지않나 생각이 든다.

## 해결책
해결책은 간단하다. **@Sse() 데코레이터 + Observable 조합**에서 **Response 객체**를 직접 이용하는 방식으로 전환하면 된다. 그리고 데이터소스 또한 **Pull** 방식의 도구로 변경해야한다. (Redis PubSub은 push 방식이기때문에 Pull 방식인 Redis Stream으로 전환했다.)

1. 제일 중요한점은 Response객체를 이용해서 클라이언트의 상태를 파악하는 것이다.

**res.write()는 응답값으로 더 데이터를 받을수 있는지 아닌지 여부를 리턴해준다.**
이값이 true일 때만 데이터를 계속 받아와서 전달할지 말지 코드를 짜면 된다.

2. Pull 방식으로 전환

Redis Pubsub에서 Redis Stream으로 변경했기때문에 특정 시점부터 데이터를 받아올수 있다.(XREAD).

3. 그리고 "drain" 이벤트 구독

클라이언트가 데이터를 받을수 없는 상태라면 res.write() 가 false를 응답하기때문에 송신프로세스를 멈추고 대기해야한다. 그리고 drain 이벤트가 발행하면 그때 다시 보내면 된다.
drain(배수라는 뜻) 물이 빠졌으니 데이터를 다시 보내라는 신호다.

아래는 변환된 샘플 코드다.

```ts
// pseudo code
@Get('chat-message/:messageId')
  async getLlmAnswer(
    @Param('messageId') messageId: string,
    @Res() res: Response,
  ): Promise<void> {
    // @Sse 를 안쓰기 때문에 직접 다 설정해야함.
    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');
    res.setHeader('Connection', 'keep-alive');
    res.setHeader('X-Accel-Buffering', 'no');
    res.flushHeaders();

    const streamKey = `llm:answer:${messageId}`;
    let lastId = '$';
    let active = true;

    res.on('close', () => {
      active = false;
      reader.disconnect();
    });

    while (active) {
      try {
        const message = await this.redisService.xread(streamKey, lastId, 5000);
        if (!active) break;

        if (message) {
          lastId = message.id;
          const canWrite = res.write(`data: ${message.data}\n\n`);

          // 백프레셔: 버퍼 가득 차면 "drain" 대기
          if (!canWrite) {
            await new Promise<void>((resolve) => res.once('drain', resolve));
          }
        }
      } catch {
        break;
      }
    }

    if (!res.writableEnded) res.end();
  }
```

res.write(), 'drain' 이벤트만으로도 부하를 조절할수 있는 구조가 됐다.

동일한 테스트를 실행하면 동일한 요청수에 OOM 이 발생하지않는다. 물론 요청수가 늘어나면 하드웨어가 제한돼어있으니 메모리 문제가 발생하지만 이건 하드웨어 수평확장으로 진행하면 된다.