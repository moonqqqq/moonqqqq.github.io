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
[1.서버는 데이터를 받아서 처리하고 응답값을 반환하는 곳이니 데이터를 받는 과정부터 정리한다.](https://moonqqqq.github.io/before-arriving-server-server-series-1)<br>
[<U>(현재글) 2.서버가 받은 데이터가 어떻게 처리되는지. Node.js만의 방식이 어떤지 정리한다.</U>](https://moonqqqq.github.io/from-the-libuv-server-series-1)</U><br>
3.Node.js만의 방식을 하나하나 조금 더 디테일하게 - Event loop, Libuv, V8, JS

<br>

이전 단계에서 데이터가 receive socket buffer에 도착했다. 이제 Node.js 서버가 데이터를 가져와서 처리해야한다. 데이터를 가져오는 일을 해주는 것이 libuv

일단 Node.js 서버가 가지고 있는 receive socket buffer는 하나가 아니다. 여러 클라이언트와 연결되기때문에 연결되는 클라이언트 수만큼 소켓이 생기게 된다. 이렇게 되면 관리해야하는 socket이 꽤나 많아지는데 


socket에서 데이터를 가져오는 것은 libuv가 책임진다. libuv안에 존재하는 OS별 라이브러리를 이용한다. linux는 epoll, windows는 kqueue가 있다. 우리의 서버는 대부분 리눅스에서 돌아가니 epoll을 중심으로 파악해보자.

epoll은 3가지 함수가 핵심이다. **epoll_create(), epoll_ctl(), epoll_wait()**.

### epoll_create()
epoll_create()는 두가지 자료구조를 만든다. 첫째는 연결된 전체 fd 목록을 가진다.(**Interest list**) 두번째 자료구조는 새로운 이벤트가 발생한 fd목록을 가진다.(**ready list**) node.js 서버가 실행됐을 때 이 과정이 실행된다.

### epoll_ctl()
그 뒤 새로운 연결이 들어오면 ***epoll_ctl()*** 함수를 이용해서 새로운 fd를 Interest list에 등록한다. 이 등록 과정에서 *어떤 작업에 대해 감지*할건지도 명시해주어야 한다. 예를 들어, 새로운 연결이 들어왔을 때는 새로운 데이터가 들어오는 이벤트(EPOLLIN)를 감지하도록 설정해둔다.

이제 새로운 데이터를 받아오는 셋팅이 완료된 상태이다. 이제 우리가 Interest list에 등록한 *fd*와 *특정 이벤트* 두 조건을 만족하는 상황이 발생하면 ready list에 해당 fd가 추가된다. event_crl() 함수를 통해 필터링 프로세스를 추가한 것과 같다.

### epoll_wait()
이제 마지막으로 epoll_wait() 함수가 실행되면 ready list에 있는 fd들을 event loop에 전달한다.

#### epoll_wait() 디테일
epoll_create()와 epoll_ctl()은 운영체제단에서 알아서 실행하지만 epoll_wait()는 event loop(libuv)단에서 실행된다. event loop의 poll phase에서 IO관련된 이벤트들을 처리하는건 이미 알고있을것이다. 그러니 당연히 새로운 IO 또한 poll phase에서 책임진다.

poll phase내에서 epoll_wait() 함수가 실행되어 새로운 IO가 발생한 fd목록을 가져온다. 하지만 ready list에 어떠한 fd도 존재하지 않는다면 새로운 IO가 생길때까지 기다린다.(그래서 이름에 "wait"이 붙어있다.) 다만 이 기다리는 조건은 상황에 따라 달라진다. 아래 event loop 코드(uv_run())을 보면 다른 phase들과 다르게 uv__io_poll() 함수만 두번째 인자로 timeout을 가지고 있는걸 알수 있다. 이 timeout을 통해 얼마나 동안 새로운 IO를 기다릴지를 정한다.

```c
// core.c
while (r != 0 && loop->stop_flag == 0) {
can_sleep =
    uv__queue_empty(&loop->pending_queue) &&
    uv__queue_empty(&loop->idle_handles);

uv__run_pending(loop);
uv__run_idle(loop);
uv__run_prepare(loop);

timeout = 0;
if ((mode == UV_RUN_ONCE && can_sleep) || mode == UV_RUN_DEFAULT)
    timeout = uv__backend_timeout(loop);

uv__metrics_inc_loop_count(loop);

uv__io_poll(loop, timeout); // 두번째 인자로 타입아웃 값 존재.
```

timeout을 정하는 조건은 아래와 같다.

- "timeout = 0": poll phase 이전 phase들에 실행할 작업이 있을 때. 기다리지 않고 바로 응답한다.
- "timeout > 0": 이전 phase들 중 timer에만 작업이 있을 때 제일 임박한 작업을 기준으로 timeout이 정해진다.
- "timeout < 0": 이전 phase들 중 어떠한 곳에서도 진행해야할 작업이 없을 때는 무기한 기다린다.

### 왜 이전 phase들만 기준으로 잡는지?
Event loop은 작업 종류에 따라 phase 별로 분리해뒀다. 그리고 정해진 순서에 따라 반복되며 실행된다. **이 순서 자체가 서버 기능의 우선순위와 같다.**

1. Timers - 언제 실행돼야할지 설정된 작업
2. Pending Callbacks - 이전 iteration에서 미뤄진 작업
3. Idle, prepare - node 시스템 내부 작업 처리
4. Poll - IO 처리
5. Check (setImmediate callbacks) - Poll phase 다음에 실행돼야할 작업
6. Close Callbacks - 마무리된 작업에 대한 정리 작업

1~4번은 이전 iteration에서 IO를 처리하는 목적을 가지고 있다. 

반면에 "5.check phase"는 setImmediate 콜백들을 처리하는데, setImmediate 자체가 poll phase 바로 뒤에 실행되도록 만들어진 함수다. 만들어진 이유가 그러니 당연히 epoll_wait() timeout의 조건에 들어가지 않는다.

"6.close phase"는 완료된 작업의 뒷작업이기 때문에 굳이 빨리 처리할 필요가 없다. 그러니 이 또한 timeout에 조건에 들어가지 않는다.

먼저 받은 IO부터 우선적으로 처리한뒤 새로운 IO를 받도록 설계되었기 때문에 새로운 IO를 받는 시간(timeout)을 0으로 설정해놓는다. timeout이 0보다 커서 기다리게된다면 이전 IO에 대한 작업이 그만큼 멈춰지게 되기때문이다.

> 우리가 만드는 서버라는 것 자체가 Input을 받아 처리하는 Output을 전달해주는 것이기 때문에 Input을 받는 전략을 잘짜야한다. Input받는 작업을 계속 실행해주는 것은 당연히 다른 작업을 못하게 되므로 불가하다. 반면 받은 input을 처리하는데 시간을 많이 쏟고 새로운 input을 받는 시간이 적다면 이것 또한 문제다. input받는 시간과 받은 input을 처리하는 시간이 적절히 균형을 이루어야한다. Node.js에서는 이 **timeout**을 통해 균형을 맞춘다.




```c
// linux.c
void uv__io_poll(uv_loop_t* loop, int timeout) {
  // ~

  for (;;) {
    if (loop->nfds == 0)
      if (iou->in_flight == 0)
        break;

    if (ctl->ringfd != -1)
      while (*ctl->sqhead != *ctl->sqtail)
        uv__epoll_ctl_flush(epollfd, ctl, &prep);

    if (timeout != 0)
      uv__metrics_set_provider_entry_time(loop);

    lfields->current_timeout = timeout;

    nfds = epoll_pwait(epollfd, events, ARRAY_SIZE(events), timeout, sigmask); // epoll_pwait()

    SAVE_ERRNO(uv__update_time(loop));

    // ~
}
```
어디서 event_wait() 함수가 실행되는지 알았으니 이제는 실행가능한 조건에 대해 알아보자. **단순히 poll phase에 진입했다고 실행되는게 아니다.** 




이벤트 루프 코드를 보면 아래 순서대로 phase가 진행된다.

```c
int uv_run(uv_loop_t* loop, uv_run_mode mode) {
  int timeout;
  int r;
  int can_sleep;

  r = uv__loop_alive(loop);
  if (!r)
    uv__update_time(loop);

  if (mode == UV_RUN_DEFAULT && r != 0 && loop->stop_flag == 0) {
    uv__update_time(loop);
    uv__run_timers(loop); // 1
  }

  while (r != 0 && loop->stop_flag == 0) {
    can_sleep =
        uv__queue_empty(&loop->pending_queue) &&
        uv__queue_empty(&loop->idle_handles);

    uv__run_pending(loop);  // 2
    uv__run_idle(loop);     // 3
    uv__run_prepare(loop);  // 3

    timeout = 0;
    if ((mode == UV_RUN_ONCE && can_sleep) || mode == UV_RUN_DEFAULT)
      timeout = uv__backend_timeout(loop);

    uv__metrics_inc_loop_count(loop);

    uv__io_poll(loop, timeout); // 4  !! 여기에 poll phase가 존재. !!

    for (r = 0; r < 8 && !uv__queue_empty(&loop->pending_queue); r++)
      uv__run_pending(loop);

    uv__metrics_update_idle_time(loop);

    uv__run_check(loop);
    uv__run_closing_handles(loop);

    uv__update_time(loop);
    uv__run_timers(loop);
    
    // ~~
```

epoll_wait timeout 설정방법.


Timers
Pending Callbacks (Certain system operations)
Idle, prepare (Internal libuv bookkeeping)
Poll (I/O Events): This is where epoll_wait comes into play.
Check (setImmediate callbacks)
Close Callbacks

setImmediate 라는 이름이 잘못지어진 것도 있다.