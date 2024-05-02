# game-server-note
게임 서버 학습 노트
## 서버?
- 다른 컴퓨터에서 연결이 가능하도록 대기 상태로 상시 실행중인 프로그램

## 서버의 종류
### 웹서버
- HTTP Server
- Stateless
- 질의/응답 형태
- 처음부터 만드는 경우는 거의 없고 프레임워크를 골라서 사용
  - ASP.NET (c#)
  - Spring (java)
  - Node.js (javascript)
  - ...
### 게임 서버
- TCP Server
- Stateful (서버에서 클라이언트로 접근이 가능하다)
- 실시간 Interaction이 있음
- 게임, 장르에 따라 요구사항이 매우 다름. 최적의 프레임워크가 딱히 존재하는 것은 아님
