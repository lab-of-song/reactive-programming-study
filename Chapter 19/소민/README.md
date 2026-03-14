# 19. (WebFlux용) 예외 처리

### 1. onErrorResume

```kotlin
ServeltRequest.bodyToMono(Dto::class)
  .flatMap()
  .onErrorResume(MyCustomException.class,  // 처리할 예외 타입
    error -> ServerResponse.badRequest() // 어떻게 처리할지
               .bodyValue(new ErrorResponse(HttpStatus.BAD_REQUEST, error.message))
```

에러 발생 시, 에러를 downstream 으로 전파하지 않고 publisher를 통해 에러 이벤트에 대한 대체값을 emit 하거나 이를 포장한 후에 다시 에러 이벤트를 발생시킨다.

### 2. ErrorWebExceptionHandler

→ Global Exception Handler 로 추가할 수 있다.

```kotlin
@Order(-2) // ErrorWebFluxAutoConfiguration 으로 등록된 DefaulrErrorWebExcpetionHandler
// 보다 우선순위가 높아야하므로, -2 로 지정한다.
// WebFIlter 에서 발생하는 오류도여기에서 캐치할 수 있다. (일반적인 controllerAdvice 에서는 webfilter 에러 핸들링 불가)
class GlobalWebExceptionHandler() : ErrorWebExceptionHandler {
   override fun handle(exchange: ServerWebExchange) {
      // databuffer 를 이용해 에러 응답을 작성 -> ErrorResponse 를 만들어서 반환할 수 있다.
      exchange.getReponse().setStatusCode(HttpStatus.BAD_REQUEST)
      
      val bufferFactory = exchange.getResponse().bufferFactory()
      buffer = bufferFactory.wrap(om.writeValueAsBytes("응답할 에러 내용")
      exchange.getReponse().writeWith(Mono.just(buffer)
    }
}
```
