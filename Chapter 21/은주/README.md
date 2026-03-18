# 21. Reactive Streaming 데이터 처리
- SSE (Server-Sent Events) : 클라이언트가 HTTP 연결을 통해 서버로부터 전송되는 업데이트 데이터를 지속적으로 수신할 수 있는 단방향 서버 푸시 기술
  - 주로 클라이언트 측에서, **서버로부터 전송되는 이벤트 스트림을 자동 수신하기 위해 사용됨**
- Spring Webflux 에서 streaming 방식으로 데이터 전송하기 위한 response body 타입은 Flux 이다