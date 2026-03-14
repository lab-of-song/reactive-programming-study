# 15. Spring Webflux

## **1. Spring WebFlux 기술 스택 (Reactive Stack)**

- Spring MVC: 서블릿 API와 동기식(Blocking) I/O에 기반을 둔 Servlet Stack
    - tomcat 같은 서블릿 컨테이너에서 동기 방식으로 처리
    - 하나의 요청에 하나의 스레드가 통으로 blocking → 다 쓰면 스레드 풀에 반납하는 구조
    - 보안 관점에서 표준 서블릿 필터를 사용하는 Spring Security 사용
- WebFlux: Non-blocking I/O와 Reactive Streams(Project Reactor)를 기반으로 하는 Reactive Stack
    - non-blocking I/O 기반의 netty 서버 엔진 위에서 처리
    - 적은 수의 스레드(이벤트 루프)로 대규모의 동시 요청을 처리하여 시스템 리소스를 효율적으로 사용하는 것.
    - Spring  Security 및 필터로 WebFilter 사용
    - non-blocking 기반의 I/O 도구 (R2DBC, NoSQL 등) 사용

## **2. WebFlux 요청 처리 흐름**


### 요청 흐름

1. Netty Server: HTTP 요청을 가장 앞단에서 수신
2. HttpHandler: 서버 API(Netty) ↔ Webflux 사이의 요청을 표준화해서 반환(추상화 역할)
    1. ServerHttpRequest, ServerHttpResponse 를 포함하는 ServerWebExchange를 만든다.
3. WebFilter: 디스패처로 넘어가기 전에 전처리 담당
4. DisPatcherHandler: MVC의 DispatcherServlet 과 유사한 역할로, 적절한 핸들러 찾고 요청을 위임하는 역할
    1. HandlerMapping 목록을 Flux 로 전달받는다.
5. HandlerAdapter: ServerWebExchange를 처리할 핸들러 호출 (실제 핸들러 실행 역할)
6. Controller/HandlerFunction: 요청 처리 후 응답 데이터 반환
    1. 단, 이 때 모든 요청을 다 처리한 후에 최종 결과를 반환하는 것이 아니다. 
    2. Mono/Flux 로 감싸진 결과를 반환하므로, 호출 즉시 실행되는 것이 아님


### **ServerRequest/Response 객체 차이**

- MVC: 서블릿 기반인 `HttpServletRequest`, `HttpServletResponse`
- WebFlux: `ServerHttpRequest`, `ServerHttpResponse`
    - 이 2개를 다 ServerWebExchange 에 담음
    - exchange.mutate() 를 통해서 재생성(기본적으로 불변이기 때문에 내용 업데이트를 위해서는 재생성을 해야한다) 해서 다음 filter chain 으로 전달한다.

### WebFilter VS HandlerFilterFunction

- WebFilter
    - bean 으로 등록
    - DispatcherHandler 실행 전에 동작
- HandlerFilterFunction
    - 함수형 엔드포인트(다른건 어노테이션 엔드포인트) 에만 route 별로 적용된다.

## Non-Blocking 기반 처리

- **기존의 MVC: 요청 하나당 하나의 스레드가 blocking → API 호출부터 응답이 모두 끝난 후에야 스레드풀에 반환**
- webflux: 받은 요청을 event loop 에 올리고, I/O 가 발생하는 작업들에 대해서는 콜백을 등록한다.
    - I/o 가 완료되면 종료 시그널 받아서 다시 CPU 작업 할 때 이벤트 루프 스레드를 통해 작업한다.

기본적으로 최소 스레드 수가 4개, 서버(컴퓨터)의 CPU가 4개보다 더 많다면 CPU 수만큼 스레드가 생긴다.

**절대 event loop 스레드 내에서 blocking 이 발생하면 안된다.** → blocking 이 발생하는 순간, 가용할 수 있는 스레드가 1개 줄어드는 것(blocking 동안 다른 작업 수행 불가)

블로킹 I/O(예: 기존 JDBC 기반 RDBMS 연동)를 수행해야 한다면, Reactor가 제공하는 별도의 스레드 풀인 `Schedulers.boundedElastic()`로 작업을 격리하여 이벤트 루프 스레드에 영향을 주지 않도록 설계해야 한다.
