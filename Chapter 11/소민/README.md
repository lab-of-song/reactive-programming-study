# 11. Context

## context란?

```bash
context란: 어떤 상황에서 그 상황을 처리하기 위해 필요한 정보
- 한 subscriber 마다 하나의 context 를 가질 수 있다.
```

### context 예시

- ServletContext: Servlet 이 Servlet Container 와 통신하기 위해 필요한 정보를 제공하는 인터페이스
- ApplicationContext: Spring framework 내에서 어플리케이션의 정보를 제공하는 인터페이스. Bean 같은 것도 application context 를 통해 제공된다.
- SecurityContextHolder: Security Context 를 관리하는 주체. 사용자 인증 정보를 담는다.

### Reactor context

- operator 같은 reactor 구성요소들 사이에 전파되는 key-value 형태의 저장소.
    - **downstream → upstream 방향으로 context 가 전달된다.**
    - 따라서, downstream 에서 context 에 넣은 데이터가 있어야만 upstream 에서 context 에서 데이터를 꺼내와 사용할 수 있다.
- 일반적인 동기 흐름의 thread local 과 달리, reactor 는 실행 스레드가 계속 변경되므로 스레드에 종속되지 않고 subscriber 에 종속된다.

## context 데이터 읽고 쓰기

### 1. 원천 데이터(최초 emit 데이터) 에서 context 읽기 → deferContextual

```java
public Mono<String> getUserName() {
    return Mono.deferContextual(ctx -> { // 이 context 는 읽기만 가능한 ContextView 객체이다.
        String userId = ctx.get("USER_ID");
        return Mono.just("Name of " + userId); // Context 값을 기반으로 새 Publisher 생성
    });
}
```

```

defer 는 Mono.just 같은 publisher 와 다르게, 
subscriber 로 부터 구독신호가 downstream -> upstream 으로 모두 올라온 후에야 emit 할 데이터를 준비한다.
따라서, chain 의 맨 아래에서 작성한 context 가 defer 까지 올라온 후에야 emit 데이터를 준비하기 때문에 
반드시 context 데이터를 받아 emit 하기 위해서는 defer를 사용해야한다.

---
Mono.just 는 파이프라인 조립(assembly) 시점에 값을 평가한다. 따라서, 아직 subscribe 가 호출되지 않은 시점이므로 context 가 없다.

```

### 2. operator 중간에서 읽기 → transformDeferredContextual

```java
public Mono<String> processData() {
  Mono.deferContextual(ctx -> {
    doSomething()
    }
  ).transformDeferredContextual((mono, ctx) -> {
        String role = ctx.getOrDefault("ROLE", "USER");
        if ("ADMIN".equals(role)) {
            return mono.map(data -> data + " (Admin Processed)");
        }
        return mono.map(data -> data + " (User Processed)");
    });
}
```

만약 publishOn, subscribeOn 으로 Mono.deferContextual 과 Mono.transformDeferredContextual 이 서로 다른 스레드에서 실행되어도 같은 subscriber 가 있으므로 같은 context 를 바라보게 된다.

### 3. context 쓰기

- context.put(key, value)
- context.of(key1, value1, key2, value2)
- context.putAll(ContextView)
    - 두 개의 context 를 합해주는데, ContextView 를 파라미터로 받으므로 context.putAll(Context.of(1,2,3,4).readOnly()) 처럼 readOnly 를 통해 ContextView 로 변환해주어야한다.

한 subscriber 내에서 동일 key 에 대해 작성하면, 더 upstream 에서 작성된게 최종값이다.

```kotlin
Mono.deferContextual(ctx -> Mono.just("최종 승자: " + ctx.get("KEY")))
    .contextWrite(Context.of("KEY", "위쪽 데이터")) // 2. 나중에 실행되어 덮어씀 (Upstream)
    .contextWrite(Context.of("KEY", "아래쪽 데이터")) // 1. 먼저 실행됨 (Downstream)
    .subscribe(System.out::println); 
```

한 publisher 에 서로 다른 subscriber 가 있고, 각각 서로 다른 key 에 대해 put 하면 서로 다른 값이 유지된다.

```kotlin
Mono<String> pipeline = Mono.deferContextual(ctx -> Mono.just(ctx.get("ID")));

// 첫 번째 구독: 독자적인 Context 1 생성
pipeline.contextWrite(Context.of("ID", "Sub-1")).subscribe();

// 두 번째 구독: 독자적인 Context 2 생성
pipeline.contextWrite(Context.of("ID", "Sub-2")).subscribe();
```

## inner/outer context 범위

```kotlin
Mono.just("data")
    .flatMap(data ->  // 여기는 inner scope
        Mono.deferContextual(ctx -> {
            // Inner Chain: Outer의 "OUTER_KEY"를 읽을 수 있음
            return Mono.just(data + ctx.get("OUTER_KEY"));
        })
        .contextWrite(Context.of("INNER_KEY", "Inner Value")) // Inner에만 존재하는 Context
    )
    .contextWrite(Context.of("OUTER_KEY", "Outer Value")) // Outer Context
    // 🚨 여기서 "INNER_KEY"를 꺼내려 하면 에러 발생 (Outer는 Inner에 접근 불가)
    .subscribe();
```

## context 쓰기 좋은 상황

spring webflux 환경에서 filter 에서 인증된 사용자의 정보를 계속 들고다녀야할 때, 매번 함수의 파라미터로 넘겨주기 보단 context 에 넣어두고 계속 꺼내쓸 수 있다. (SecurityContext 와 유사)

```kotlin
@Component
public class SecurityFilter implements WebFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        // 1. HTTP 헤더에서 사용자 ID 추출
        String userId = exchange.getRequest().getHeaders().getFirst("X-User-Id");
        
        // 2. 체인 실행 후, 맨 마지막에 contextWrite를 통해 Context 주입
        return chain.filter(exchange) // 이 filter 에 들어가는게 controller -> service ->repo 의 전체 흐름이라고 보면 된다.
                    .contextWrite(Context.of("USER_ID", userId != null ? userId : "ANONYMOUS"));
                    // 최종적으로 netty 가 사요자 요청이 들어오면 subscribe() 를 호출하여, contextWrite 가 가장 먼저 트리거되어 context 에 인증 정보를 담는다.
    }
}
```

### 참고

[Context가 아래에서 위로 올라가는 과정을 타임라인]

1. **조립 시간 (Assembly Time) [Top -> Bottom]**
    - 코드가 작성된 순서대로 Publisher 체인이 연결된다.. 실제 데이터 흐름이나 실행은 발생하지 않는다.
2. **구독 시간 (Subscription Time) [Bottom -> Top]**
    - 최하단(Netty 웹 서버)에서 `.subscribe()`가 호출
    - 구독 신호는 역방향으로 전파되며, WebFilter의 `contextWrite`를 만나 Context에 데이터를 담아 상단까지 올라간다.
3. **실행 시간 (Execution Time) [Top -> Bottom]**
    - 최상단 원천(Upstream Source, 예: DB 조회 로직)에 구독 신호와 Context가 도달하면, 비로소 Context 안의 데이터를 읽고 실제 쿼리를 실행하여 정방향(위에서 아래로)으로 결과 데이터를 방출(Emit)

### 예시)


```kotlin
// 스프링 웹 서버(Netty)가 최종적으로 구독하는 단일 파이프라인
Mono<ResponseEntity<UserDto>> webFluxPipeline = 
// 1. upstream source
    userRepository.findById()  // 내부에 defer 가 숨어있다고 보면 된다.
    // 2. operator(service)
    .map(user -> new UserDto(user.getId(), user.getName())) 
    // 3. operator(controller)
    .map(dto -> ResponseEntity.ok(dto)) 
    // 4. context 작성
    .contextWrite(Context.of("USER_ID", "user-123"));
```

```kotlin
// UserRepository 클래스 내부
public Mono<User> findById() {
    // 여기서 deferContextual로 구독 시점까지 기다렸다가 Context를 꺼내 읽습니다.
    return Mono.deferContextual(ctx -> {
        String userId = ctx.get("USER_ID"); // 밑에서 올라온 "user-123" 획득!
        
        // 꺼낸 ID로 진짜 DB 프레임워크 호출
        return r2dbcTemplate.select(User.class)
                            .matching(query(where("id").is(userId)))
                            .one();
    });
}
```
