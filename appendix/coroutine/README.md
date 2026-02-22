# 1. WebFlux  + Coroutine 기본 개념

### **WebFlux Event Loop**

- WebFlux는 소수의 Event Loop 스레드만으로 비동기 요청을 처리한다.
- 동기식 DB 커넥션(Jdbc), 동기식 Redis 클라이언트(Reactive 가 아닌 일반 RedisTemplate), RestTemplate 등 블로킹 I/O 작업이 Event Loop에서 실행되면 스레드가 I/O 응답을 기다리며 유휴 상태로 블로킹된다.
- 소수로 운영되는 Event Loop 중 하나라도 점유 상태에 빠지면 가용 스레드가 tomcat 과 비교했을 때 현저하게 적기 때문에 소수의 요청만 블로킹되어도 스레드 풀이 고갈되며 서버 전체의 장애로 이어진다.
    - **가용 가능한 event loop thread 가 없으면 다른 작업들을 처리할 수가 없어진다.**

### Coroutine 이란

- **Coroutine:** 중단과 재개가 가능한 논리적 작업 단위(경량 스레드).
- **Dispatcher:** 코루틴(작업)을 어느 물리적 스레드 풀에서 실행할지 결정하는 역할
    - `Dispatchers.Default`: CPU 를 많이 쓰는 연산
    - `Dispatchers.IO`: 네트워크 통신, DB 처리 등 블로킹 대기가 발생하는 작업용.
    - **컨텍스트를 명시하지 않으면 부모의 실행 컨텍스트(WebFlux의 Event Loop 스레드)를 그대로 상속받는다.**

### **Coroutine을 통한 Non-blocking I/O 제어 (Suspend & Resume)**

코루틴의 `suspend` 키워드가 붙은 함수는 I/O 대기가 발생할 때 스레드를 점유하지 않고 반환(Suspend)한다.

```kotlin
suspend fun doSomething() {
  // 여기에서 실행되는 작업 중 중단이 되는 내용이 있을 수 있다는 것을 suspend 키워드로 알려준다.
}
```

- `doSomething()` 을 처리하다가 I/O 작업으로 넘어가면, doSomething 을 처리하던 event loop 는 다시 풀로 돌아가서 다른 요청을 처리한다.
- I/O 작업이 완료되면 코루틴(실행할 작업 모음) 이 가용 스레드(event loop)를 다시 할당받아 로직을 재개(Resume)한다.

### **Reactor vs Coroutine**

- **Reactor (Mono/Flux):** 체이닝 연산자(`.map()`, `.flatMap()`)를 기반으로 동작하여 로직이 복잡해질수록 콜백 패턴처럼 가독성이 크게 저하된다.
- **Coroutine:** 비동기 흐름을 동기식 코드와 동일하게 위에서 아래로 순차적으로 작성할 수 있어 코드의 직관성과 유지보수성이 향상된다.
    - 코드만 봤을 때는 동기식으로 순차 실행되는 것으로 보이나, suspend 나 coroutine scope 덕분에 작업 내용을 다른 스레드로 분리 → event loop 가 블락킹되는 현상을 막을 수 있다.

# 2. Coroutine Scope

- **`coroutineScope` (동시성 제어)**
    - 하나의 coroutineScope 로 묶은 경우, 묶인 내부의 자식 코루틴들이 모두 완료될 때까지 부모 코루틴의 종료를 대기시키는 논리적 스코프를 생성한다.
    - 실행 스레드(Context)를 변경하지 않으며 부모 스레드를 그대로 상속한다.
    
    ```kotlin
    suspend fun doSomething() = coroutineScope { // context 를 명시하지 않았으므로 부모 스레드 그대로 사용
       async { callSometingAndWaitForResponse() }.await()
       launch ( fireAndForget() }
    }
    ```
    
- **`withContext` (스레드 전환)**
    - 현재 코루틴의 실행 스레드를 전환(`Dispatchers.IO` 등)하고, 해당 블록의 실행이 완료될 때까지 대기한 후 결과를 반환한다.
    - WebFlux 환경에서 레거시 동기식 라이브러리를 호출해야 할 때, Event Loop 스레드가 블로킹되는 것을 막기 위해 동기식 작업은 `withContext(Dispatcher.IO)` 내에서 실행되도록 전환해야한다.
    
    ```kotlin
    // withContext를 활용한 블로킹 I/O 격리 처리
    override suspend fun saveSession(session: HubSession) { // 이벤트 루프에서 실행되는 함수
        // Event Loop 스레드 대신 IO 스레드 풀에서 블로킹 작업을 처리
        withContext(Dispatchers.IO) {
            redisTemplate.opsForValue().set(session.id, session)
        }
    }
    ```
    

# 3. launch → Job, async → Deferred

### **`launch` , `Job`**

- **목적:** 로깅, 이벤트 발행 등 **결과값을 반환할 필요**가 없는 비동기 작업에 사용. → fire and forget
- **반환 (`Job`):** 작업의 생명주기를 관리하는 핸들 객체. 실제 데이터 결과값은 포함하지 않는다.
    - `job.cancel()`을 호출하여 작업을 강제 취소하고 리소스를 회수할 수 있다. `job.join()`으로 작업 완료를 강제로 대기할 수도 있다.

### **`async` , `Deferred<T>`**

- **목적:** 외부 API 병렬 호출 등 비동기 작업 후 결과 데이터가 반드시 필요한 경우에 사용.
- **반환 (`Deferred`):** `Job` 인터페이스를 상속받은 객체로, 취소 제어 기능과 더불어 비동기 연산의 최종 결과값을 담는 컨테이너 역할을 한다.
    - 즉, 트리거 시긴 코루틴의 결과를 받아야한다면 async 로 호출 후 await 로 기다려서 결과를 받는다.
    - `deferred.await()`를 호출하면 실제 결과값이 연산될 때까지 코루틴을 중단(Suspend)시키고, 처리가 완료되면 `T` 타입의 데이터를 반환한다.

```kotlin
// 외부 API 병렬 호출 및 Deferred 제어
suspend fun doSomething(servers: List<Server>) = coroutineScope {
    // async 호출 즉시 비동기 작업을 큐에 넣고 Deferred 객체 리스트 반환
    val deferredResults: List<Deferred<Result>> = servers.map { server ->
        async { apiClient.call(server.url) } 
    } // async 덕분에 여러 호출이 모두 동시에 진행된다. 
    // 각 호출 당 3초가 소모된다하면, 3개의 서버 호출이 동시에 되므로 총합 3초만 기다리면 된다.
    
    // awaitAll()을 통해 모든 병렬 작업의 처리가 완료될 때까지 대기 후 결과 취합
    return@coroutineScope deferredResults.awaitAll()
}
```

# 4. coroutineScope vs supervisorScope

다수의 자식 코루틴을 병렬로 실행할 때, **특정 자식 코루틴에서 발생한 예외를 어떻게 처리할지 결정**하는 개념

### **`coroutineScope` : 자식 코루틴이 실패하면 부모 코루틴 전체 실패**

- 한 자식 코루틴에서 예외가 발생하면 예외가 부모로 전파되며, 실행 중이던 다른 형제 코루틴들 역시 모두 강제 취소(Cancel)된다.
- 전체 작업의 트랜잭션적 일관성이 필요할 때 사용한다. 개별 실패를 허용하려면 로직 내부에서 `runCatching`으로 예외를 처리해야 한다.

### **`supervisorScope` : 자식 코루틴이 실패해도 다른 코루틴은 영향을 받지 않음**

- 자식 코루틴의 예외가 부모나 다른 형제에게 전파되지 않는다.
- 특정 백그라운드 작업이 실패하더라도 다른 비동기 작업들은 취소되지 않고 각자의 생명주기 끝까지 독립적으로 실행을 보장해야 할 때 사용한다.

```kotlin
// supervisorScope를 활용한 독립적 비동기 실행 보장
suspend fun sendNotifications() = supervisorScope {
    launch { sendEmail() }
    launch { sendKakao() } // 이 작업에서 예외가 발생해도 Email, Slack 발송 로직은 취소되지 않음
    launch { sendSlack() }
}
```

# 5. Flow와 WebFlux SSE(Server-Sent Events) 스트리밍 처리

## **Flow 란**

시간에 따라 발생하는 다수의 데이터를 비동기 스트림으로 처리하기 위한 코루틴 API이다. (Reactor의 `Flux`를 완벽히 대체함)

### **Cold Flow vs Hot Flow**

- **Cold Flow (일반 `Flow`):** 터미널 연산자(`collect` 등)가 호출되어 구독이 시작되기 전까지는 내부 로직이 전혀 실행되지 않는 지연(Lazy) 특성을 갖는다.
- **Hot Flow (`SharedFlow`, `StateFlow`):** 구독자의 존재 여부와 무관하게 메모리 상에서 이벤트를 계속 생산하고 상태를 유지한다.

### **WebClient SSE 응답 파이프라인 구성**

- WebClient가 수신한 Reactor의 `Flux` 스트림을 확장 함수인 `bodyToFlow()`를 사용하여 코루틴의 `Flow`로 직접 변환한다.

```kotlin
// WebClient를 이용한 SSE to Flow 패스스루 구현
override fun streamMessages(url: String): Flow<ServerSentEvent<String>> {
    return webClient.get()
        .uri("$url/stream")
        .accept(MediaType.TEXT_EVENT_STREAM)
        .retrieve()
        .bodyToFlow<ServerSentEvent<String>>()
}
// webclient 의 sse 응답을 그대로 스트리밍 할 수 있음
```
