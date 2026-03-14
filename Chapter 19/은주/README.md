# Chapter 19. 예외 처리
## 19.1. onErrorResume() Operator 를 이용한 예외 처리
- onErrorResume() : 에러 이벤트 발생 시 에러 이벤트를 downstream 으로 전파하지 않고, **대체 publisher 를 통해 에러이벤트에 대한 대체값을 emit 하거나 발생한 에러 이벤트를 래핑**한 후 다시 에러 이벤트를 발생시킴

## 19.2. ErrorWebExceptionHandler 를 이용한 글로벌 예외 처리
- 각 sequence 마다 onErrorResume() 을 일일이 추가해야 되고 중복 코드가 발생할 수 있으므로 Global Exception Handler 를 추가로 작성할 수 있음
