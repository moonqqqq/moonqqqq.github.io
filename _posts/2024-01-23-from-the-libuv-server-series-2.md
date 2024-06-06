---
title: 서버가 데이터를 받기까지 [epoll] - 서버가 작동하는 흐름 2
description: 클라이언트의 요청이 서버도착하기 전까지의 흐름.
header: API 서버가 데이터를 받기까지 - 서버가 작동하는 흐름 1
tags:
 - node.js
 - TCP/IP
 - OS
---

#### 시리즈 목차
[1.서버가 데이터를 받기까지 [운영체제에서]](https://moonqqqq.github.io/before-arriving-server-server-series-1) <br>
[<U>(현재글) 2.서버가 데이터를 받기까지 [libuv - epoll]</U>](https://moonqqqq.github.io/from-the-libuv-server-series-2) <br>
3.서버가 받은 데이터가 어떻게 처리되는지. Node.js만의 방식이 어떤지 정리한다. <br>
4.Node.js만의 방식을 하나하나 조금 더 디테일하게 - Event loop, Libuv, V8, JS

---

<br>

이전 단계에서 데이터가 "receive socket buffer"에 도착했다. 이제 Node.js 서버가 데이터를 가져와서 처리해야한다. 데이터를 가져오는 일을 해주는 것이 libuv의 한가지 책임 중 하나이다. (가져오는 일이라고 하기보단 서버와 클라이언트간의 연결인 소켓들을 관리해주는 일)

일단 서버로 향하는 모든 연결은 하나의 소켓으로 가는게 아니다. 여러 클라이언트와 연결되기때문에 연결되는 클라이언트 수만큼 소켓이 생기게 된다. 이렇게 되면 관리해야하는 socket이 꽤나 많아지는데. 이를 관리하는 매커니즘 종류에 따라 IO 성능의 차이가 있다. 고전적인 방법(select, poll)보다 libuv가 탁월히 좋다. (select, poll, libuv 비교는 글 마지막에..)

socket에서 데이터를 가져오는 것은 libuv가 책임이라고 했는데, libuv안에 존재하는 OS별 라이브러리를 이용하여 작동한다. linux는 epoll, windows는 kqueue가 있다. 우리의 서버는 대부분 리눅스에서 돌아가니 "epoll"을 중심으로 파악해보자.

epoll은 3가지 함수가 핵심이다. **"epoll_create()", "epoll_ctl()", "epoll_wait()"**".

### epoll_create()
epoll_create()는 두가지 자료구조를 만든다. <U>첫째는 연결된 전체 fd 목록을 가진다.(**Interest list**) 두번째 자료구조는 새로운 이벤트가 발생한 fd목록을 가진다.(**ready list**)</U> node.js 서버가 실행됐을 때 epoll_create() 또한 실행된다.

### epoll_ctl()
새로운 연결이 들어오면 ***epoll_ctl()*** 함수를 이용해서 새로운 fd를 Interest list에 등록한다. 이 등록 과정에서 *어떤 작업에 대해 감지*할건지도 명시해주어야 한다. 예를 들어, 새로운 연결이 들어왔을 때는 새로운 데이터가 들어오는 이벤트(EPOLLIN)를 감지하도록 설정해둔다.

이제 새로운 데이터를 받아오는 셋팅이 완료된 상태이다. 이제 Interest list에 등록된 *fd*와 *특정 이벤트* 두 조건이 만족하는 상황이 발생하면 ready list에 해당 fd가 추가된다. event_crl() 함수를 이용하요 필터링 프로세스를 추가한 것이라고 생각하면 된다.

### epoll_wait()
이제 마지막으로 epoll_wait() 함수가 실행되면 ready list에 있는 fd들을 event loop에 전달한다.

#### epoll_wait() 디테일
epoll_create()와 epoll_ctl()은 운영체제단에서 알아서 실행하지만 epoll_wait()는 event loop(libuv)단에서 실행된다. event loop의 poll phase에서 IO관련된 이벤트들을 처리하는건 이미 알고있을것이다. 그러니 당연히 새로운 IO 또한 poll phase에서 책임진다.

poll phase내에서 epoll_wait() 함수를 실행하여 새로운 IO가 발생한 fd목록을 가져온다. 하지만 ready list에 어떠한 fd도 존재하지 않는다면 새로운 IO가 생길때까지 기다린다.(그래서 이름에 "wait"이 붙어있다.) 다만 이 기다리는 조건은 상황에 따라 달라진다. 아래 event loop 코드(uv_run())을 보면 다른 phase들과 다르게 uv__io_poll() 함수만 두번째 인자로 timeout을 가지고 있는걸 알수 있다. 이 timeout을 통해 얼마동안 새로운 IO를 기다릴지를 정한다.

```c
// core.c
~

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

~
```

timeout이 정해지는 조건은 아래와 같다.

- timeout = 0: poll phase 이전 phase들에 실행할 작업이 있을 때. 기다리지 않고 바로 응답한다.
- timeout > 0: 이전 phase들 중 timer에만 작업이 있을 때 제일 임박한 작업을 기준으로 timeout이 정해진다.
- timeout < 0: 이전 phase들 중 어떠한 곳에서도 진행해야할 작업이 없을 때는 무기한 기다린다.

#### 왜 이전 phase들만 기준으로 잡는지?
Event loop은 작업 종류에 따라 phase 별로 분리해뒀다. 그리고 정해진 순서에 따라 반복되며 실행된다. **이 순서 자체가 서버 기능의 우선순위와 같다.**

1. Timers - 프로그램이 계획 잡아둔 작업
2. Pending Callbacks - 이전 iteration에서 미뤄진 작업
3. Idle, prepare - node 시스템 내부 작업 처리
4. Poll - IO 처리
5. Check (setImmediate callbacks) - Poll phase 다음에 실행돼야할 작업
6. Close Callbacks - 마무리된 작업에 대한 정리 작업

#### poll phase 이전
1~4번은 한바퀴 전 iteration에서 발생한 IO를 처리하는 목적을 가지고 있다. 먼저 받은 IO부터 우선적으로 처리한뒤 새로운 IO를 받도록 설계되었기 때문에 새로운 IO를 받는 시간(timeout)을 0으로 설정해놓는다. timeout이 0보다 커서 기다리게된다면 이전 IO에 대한 작업이 그만큼 멈춰지게 되기때문이다.


#### poll phase 이후
반면에 "5.check phase"는 setImmediate 콜백들을 처리한다. 그런데 이 setImmediate 자체가 poll phase 바로 뒤에 실행되도록 설계된 함수다. 만들어진 이유가 그러니 당연히 epoll_wait() timeout의 조건에 들어가지 않는다. (setImmediate라는 이름 자체가 잘못지어진게 아닌가 싶다. 처음엔 Immediate인데 왜 poll 뒤에 작동하는지 납득이 안됐다.)

"6.close phase"는 완료된 작업의 뒷작업이기 때문에 굳이 빨리 처리할 필요가 없다. 그러니 이 또한 timeout에 조건에 들어가지 않는다.

> 우리가 만드는 서버라는 것 자체가 Input을 받아 처리하는 Output을 전달해주는 것이기 때문에 Input을 받는 전략을 잘짜야한다. Input받는 작업을 계속 실행해주는 것은 당연히 다른 작업을 못하게 되므로 불가하다. 반면 받은 input을 처리하는데 시간을 많이 쏟고 새로운 input을 받는 시간이 적다면 이것 또한 문제다. input받는 시간과 받은 input을 처리하는 시간이 적절히 균형을 이루어야한다. Node.js에서는 이 **timeout**을 통해 균형을 맞춘다.

## epoll_wait() 으로 받아온 데이터는.
epoll_wait()으로 새로운 IO를 가진 fd목록을 가져왔다. 이 목록을 순회하면서 콜백을 실행시킨다. 이 콜백은 epoll_ctl()를 실행시킬 때 같이 등록해놓은 콜백이다. 데이터를 읽어오거나 에러를 처리하는 콜백이 실행된다.
Node.js에서 socket.on('data', callback); 처럼 실행된다고 보면 간단하다.
***주의해야 할 점은 이 콜백들은 poll phase의 queue에 들어가거나 나오는 작업없이 epoll_wait()에서 응답받자마자 바로 실행된다.***

이제 libuv에서 받아온 데이터가 http request로 변화할 차례다. 이건 다음글에서 이야기하자.


### Select/Poll VS Epoll
끝내기 전에 고전적인 방식(select, poll)과 epoll의 차이점을 간단히 정리해보고 가자.
select, poll의 방식은 서버가 정기적인 시간마다 서버와 연결된 socket들에 변화가 있는지 하나하나 확인한다. socket이 10000개라면 정기적인 시간마다 10000개를 확인하는 것이다. 반면에 epoll은 소켓들을 관리하는 자료구조를 따로 만들고. 소켓에 들어오는 이벤트들을 검사한다. libuv에 전달해줘야하는 이벤트인지 아닌지. 맞다면 관리 자료구조에 해당 fd를 적어둔다. 그리고 정기적으로 서버가 새로운 이벤트에 대해 요청하면 그 데이터만 전달한다. 다른 검사 과정은 없다. 이러니 획기적으로 fd 검사 횟수가 줄어들수 밖에 없다. 처음엔 소켓을 관리하는 데이터를 만들고. 모든 이벤트를 필터링하는 것도 부하가 들지 않을까 생각했지만 필터링 하나 넣는것이 그렇게 큰 부하가 드는 일이 아니라서 월등히 성능이 좋아질수 밖에 없다는 걸 받아들였다.

필터링 과정이 부하가 되지는 않을까 생각해봤지만 매번 모든 fd를 확인해서 까보는 것에 비하면 비교할수 없을 정도로 부하가 작다.