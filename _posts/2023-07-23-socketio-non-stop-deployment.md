---
title: Socket.io 무중단 배포
description: Socket.io 무중단 배포
header: Socket.io 무중단 배포
tags:
 - node.js
 - Socket.io
---

채팅 서버 무중단 배포 방식을 정리하고 가자.

BlueGreen, Canary과 같은 배포 방식에 관련된 글이라기보다는 무중단 배포에 있어 채팅 서버의 어떤 점들을 유의해야하는지에 대한 글이겠다. (Socket.io를 기준으로)

Socket 배포에 있어서 중요한 건 "client-server간의 관련 데이터 관리", "client-server connection 관리"이 두가지가 되겠다.

## socket.io 관련 데이터 관리.
socket.io에서는 실시간 통신을 위해 필요한 데이터들이 몇가지 있다. 보통 session이라고 한다. 

SocketId: 기본적으로 클라-서버가 연결됐을 때 만들어진다.
Namespaces: 소켓에서 구획을 나눠서 이용할수 있게 해주는 기준이다. 보통 기능에 따라 나눈다. (e.g. normal-chat, group-chat)
Rooms: namespace안에서 한번 더 구분되는 단위라고 생각하면 된다. (e.g. normal-chat(namespace)중에서 1번 room)
Event Handler: 특정 이벤트에 대한 행동

socket.io에서는 session들을 **adapter**라는 도구로 관리한다. 하나의 인스턴스에서 socket.io가 작동한다면 기본적으로 memory adapter로 관리된다. 인스턴스 개수가 두개가 될때부터는 데이터 동기화를 위해 외부 저장장치로 adapter를 설정해야한다. redis, db등 여러가지가 있고 redis가 주로 많이 쓰인다. 
socketId, namespace등이 adapter를 통해 실시간으로 동기화되어 데이터를 목적지 client들에게 전달할 수 있게된다.
실시간 동기화되는 데이터저장소가 있기때문에 업데이트시에도 기존 상태 데이터가 잘 보관될수 있게 된다.

## Connection 관리.
Socket.io로 연결된 클라이언트는 socketId를 받게 된다. 서버쪽에서는 socketId를 adapter에 저장해두어 커낵션 정보를 관리한다. socket.io 서버는 아래와 같은 형식으로 데이터들을 저장한다.

```js
sids: Map<SocketId, Set<Room>> // 소켓ID와 room 매칭 데이터

// 예시
sids: {
    socketId1: [
        roomId1,
        roomId2
    ],
    socketId2: [
        roomId2,
        roomId3
    ]
}

rooms: Map<Room, Set<SocketId>> // room과 소켓ID 매칭 데이터

// 예시
rooms: {
    roomId1: [
        socketId1,
        socketId2
    ]
}
```

무중단 배포가 시작되고 한 인스턴스가 업데이트 된다면 해당 인스턴스와 연결된 클라이언트들은 socket 연결이 끊긴다. 그런데 Socket.io 자체에는 자동 재연결 기능이 구현되어있다. 연결이 끊긴다면 바로 재연결을 시도하고 재연결이 완료되면 새로운 socketId를 받게 된다.
(재연결 시도 할 때, 로드밸런서의 도움으로 지금 업데이트중인 인스턴스가 아니라 이용가능한/살아있는 다른 인스턴스와 연결이 된다.)

이때 우리가 서버쪽에 추가적인 작업을 하나 셋팅해둬야한다. **"reconnection" 이벤트가 발생했을 때 유저가 이용중이던 채팅방에 다시 join() 시켜주는 기능이 필요하다.**
우선 중앙 저장소에 현재 접속되어있는 채팅방-유저 데이터를 저장하고 있어야한다. 채팅방1: [유저1, 유저2, 유저3] 이런 데이터를 실시간으로 유지하고 있다면 reconnection 이벤트가 발생했을 때 해당 유저가 이용중이던 room에 바로 다시 join() 시켜줄수 있게 된다.
이렇게 된다면 ***socket 서버에 다시 연결하는 시간 + 내가 이용중인 채팅방 정보 가져오는 시간 + 해당 채팅방으로 다시 join() 하는 시간 으로 중단 시간이 단축된다.***



