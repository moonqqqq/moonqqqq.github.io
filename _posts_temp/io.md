결국엔 서버는 IO를 처리하는 프로그램이라서 IO에 대한 이해가 필요하다.

IO가 무엇인가.

CPU, 메인보드,  IO장치들.
키보드 타이핑, 마우스 움직임, 네트워크 요청, 모니터의 보여주는 데이터가 바뀌는 것도 모두 IO.

IO에 걸치는 요소들.
CPU / motherboard / memory controller / io devices.

모든(대부분의) IO는 CPU를 거쳐야 한다. 몇 특수한 IO장치들은 그들만의 작업을 CPU에 전달하기전에 미리 하기도 한다. cpu에 일을 도와주는 것인데 offloading이라고 한다.

NIC

cpu


모든 인풋은 CPU를 거쳐서 다음 단계로 진행될수 있어야 한다.

IO가 돌아가기 위해선
개별 IO용 장치, CPU, 메인보드

우리가 만드는 프로그램이 IO를 처리하는 방식과 다른 장치들이 IO를 처리하는 방식은 꽤나 다르다.


--------


input -> (preprocessing on each devices) -> motherboard controller -> cpu


결국엔 서버는 IO를 처리하는 프로그램이다. 그러니 결국 IO를 공부하게 된다.


NIC가 마더보드에 붙어있는 이유 - 속도/거리 & 리소스 이용.

Check.
- NIC는 마더보드에 꽂혀있다.
- 마우스/키보드의 IO, network IO의 차이.
- "거리가 가까울수록 데이터 전송이 빠르다."
- Bus


///
