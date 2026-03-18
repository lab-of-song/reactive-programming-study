# Chapter 20. WebClient
- WebClient는 Spring 5부터 지원하는 Non-blocking HTTP request 를 위한 리액티브 웹 클라이언트
- WebClient 는 Non-blocking, Blocking HTTP request 를 모두 지원하기에 RestTemplate 을 대체할 것으로 예상됨

## 20.3. WebClient Connection Timeout 설정
- `option()` : HTTP Connection 이 연결되기 까지의 시간
- `doOnConnected()` : 파라미터로 전달되는 람다식을 통해 connection 이 연결된 이후에 수행할 동작을 정의함