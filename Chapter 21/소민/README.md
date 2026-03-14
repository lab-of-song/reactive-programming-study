# 21. Reactive Streaming 데이터 처리

```kotlin
webflux 를 사용해서 SSE 데이터를 스트리밍할 수 있다.

SSE란: Server Sent Event의 약자로, 클라이언트가 HTTP 연결로 서버가 일방적으로 보내는 이벤트를 지속적으로 수신할 수 있는 단방향 서버 푸시 기술
클라이언트가 서버의 이벤트를 자동 수신하기 위해 사용한다.
단, 커넥션 유지 비용에 대한 관리가 필요해진다. (서버 -> 클라이언트에게 계속 이벤트를 줄 수 있는 커넥션이 열려있어야하므로)
```

Reactive Spring 에서 SSE 는 Flux 로 전달해야한다.

```kotlin
fun stream(): Flux<Book> {
  return template.select(Book.class)
                 .all()
                 .delayElements(Duration.ofSecodns(3L)) // 3초에 한 번 씩 데이터 emit

```

위와 같은 로직이 있을 때, 이를 controller 를 사용해서 SSE 로 스트리밍 하려면 아래 내용만 설정하면 된다.

- 헤더를 text/event-stream 으로 설정
- `stream().map(book.toResponse())`
    - 굳이 SErverSentEvent 타입으로 감싸지도 않아도 되는듯

webclient 로 수신할 때

```kotlin
Flux<BooktResponse> response = 
  webClient
    .get()
    .uri(uri)
    .retrieve() 
    // sse 를 받으려면 api 콜을 받았을 때, 커넥션을 열어둔 상태로 계속 response 의 스트림을 수신해야한다.
    // retrieve 로는 그게 가능한데, 커넥션 누수로 인해 retrieve 가 현재 deprecte 된 상태
    // 이것 외에 SSE를 받을 수 있는 (즉, body 로 변환을 해도 커넥션이 닫히지 않는) 기능이 있는지??
    .bodyToFlux(BookResponse.class);
    
response.subscribe(book -> {
  logger.info("book: {}", book)
}
```
