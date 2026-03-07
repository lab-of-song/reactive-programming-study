# Chapter 17. 함수형 엔드포인트 (Functional Endpoint)
## 17.1. HandlerFunction 을 사용한 request 처리
- Spring WebFlux 의 함수형 엔드포인트는 들어오는 요청을 처리하기 위해, **HandlerFunction 이라는 함수형 기반 핸들러 사용**

## 17.2. request 라우팅을 위한 RouterFunction
- RouterFunction : 들어오는 요청을 해당 HandlerFunction 으로 라우팅해주는 역할
  - @RequestMapping 애너테이션과 동일한 기능
  - RouterFunctionBuilder 객체로 각각 HTTP METHOD 에 매치되는 request 를 처리하기 위한 라우트를 추가함