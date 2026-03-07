# 4. Operator

### 생성형 operator

| operator | 역할 |
| :--- | :--- |
| fromIterable | iterable 에 포함된 데이터를 emit 하는 flux 를 생성한다.<br>리스트 데이터를 순차적으로 emit 한다. |
| fromStream | stream 에 포함된 데이터 emit 하는 flux 생성.<br>Java Stream 특성 상 stream 은 재사용이 불가하다. |
| range(n, m) | n부터 1씩 증가하는 연속된 수 M개를 emit 한다.<br>for 문 같은 반복되는 값 emit 시 사용하기 좋음 |
| defer | operator 선언 시점(assembly 시점) 에 바로 데이터를 emit 하지 않고, 구독 시점에 emit 하는 flux/mono 를 성한다.<br>• 일반적인 Mono/Flux.just() 는 처음 구성 시점에 데이터를 미리 만들어두고 subscribe 가 오면 만들어진 데이터를 바로 발행한다.<br>• defer 는 구독이 들어오면 그 때 데이터를 만들어서 발행한다. → 불필요한 함수 호출을 줄일 수 있다.<br>just/defer publisher 에 대해 1초 뒤에 각각 subscribe(print current time) 하면, just publisher 에 대해서는 publisher 가 만들어진 시점으로 바로 시간이 찍히고, defer publisher 에 대해서는 subscribe 한 시점으로 시간이 찍힌다.<br>(Mono.just는 hot publisher 이기 때문에 구독과 관계없이 바로 데이터를 emit 한다.)<br>defer 를 쓰면 불필요한 시점의 emit 으로 인한 프로세스를 줄일 수 있다.<br>(아래 참고) |
| using(리소스 (e.g., file fromStream) | operator 의 파라미터로 받은 resource 를 emit 하는 flux 를 생성한다. |
| generate | 동기적으로 데이터를 하났기 순차적으로 emit 할 때 사용 |

### 필터링 operator

| operator | 역할 |
| :--- | :--- |
| filter | upstream 에서 emit 된 데이터 중 조건에 일치하는 것만 downstream 으로 emit 한다. |
| skip(n) | upstream 에서 emit 된 데이터 중 n 개를 skip 하고 그 다음 데이터부터 emit 한다. |
| take(n) | upstream 에서 emit 된 데이터 중 앞에서부터 n개만 downstream 으로 emit 한다.<br>n개를 다 채운 후에는 종료된다. |
| next | upstream 에서 emit 된 것들 중, 처음 1개만 downstream 으로 emit 한다. |

### sequeuce 변환 operator

| operator | 역할 |
| :--- | :--- |
| map | |
| flatMap | upstream 에서 emit 된 데이터를 inner stream 에서 평탄화하면서 하나의 sequence 로 병합되어 downstream 으로 emit 된다. |
| concat(flux1, flux2) | 두 개의 publisher sequence 를 연결해서 순차 emit 한다. |
| merge | |
| zip | |
| and | |
| collectList | flux 에서 emit 된 데이터를 모아서 List(단일 객체)로 변환한 후, List 를 emit 하는 Mono 를 반환한다. |
| collectMap | |

### sequence 내부 동작 확인(디버깅)용 operator

- doOnXXX
  - 데이터 변경없이 부수효과만 수행한다.<br>• 별도의 return 값이 없다. 

| operator | 역할 |
| :--- | :--- |
| doOnSubscribe | publisher 가 구독 되고 있을 때 트리거 되는 동작을 추가한다. |
| doOnRequest | publisher 가 subscriber 로 부터 요청을 받았을 때 트리거 (e.g.,데이터 N개씩 주세요) |
| doOnNext | publisher 가 데이터 emit 할 때 트리거 되는 동작 |
| doOnComplete | publish 모두 완료되었을 때 |
| doOnError | publisher 가 error 로 종료되었을 때 |
| doOnCancel | publisher 가 취소되었을 때 |

### 에러처리 operator

| operator | 역할 |
| :--- | :--- |
| error | try-catch 에서 에러를 그대로 던지는 것과 동일하다.<br>→ 지정된 에러로 종료하는 Flux 를 생성한다.<br>return Flux.error(new IllegalArgumentException) |
| onErrorReturn | 에러 이벤트 발생 시, 에러 이벤트를 downstream 으로 넘기지 않고 대체값을 emit 한다.<br>runCatching 의 getOrDefault 와 동일<br>onErorrReturn(”Nothing”)<br>에러 타입 지정도 가능<br>.onErrorReturn(IllegalArgumentException.class, “Nothing”)<br>.onErrorReturn(OtherException.class “None”) |
| onErrorResume | 에러 발생 시, 새로운 publisher 로 대체하여 이어서 publish 한다<br>• catch block 에서 에러 발생 시, 다른 메서드를 호출하는 것과 유사<br>onErrorResume(e → getBooksFromDatabase())<br>• 로컬 캐시에 데이터 없어서 에러 터지면 DB 가져오게 |
| onErrorContinue | 에러 발생 시, 에러가 터진 데이터를 제거하고 upstream 에서 후속 데이터를 emit 한다.<br>map(num → 12/num) // num이 0이면 에러 발생<br>.onErrorContinue((e, num) →<br>log.error(”error”) // 그냥 에러 터진거 핸들링하고 다음 데이터 이어서 받는 것<br>) |
| retry(n) | 에러 발생 시 N번 재시도 → 처음부터 재시도한다.<br>timeoutException 등으로 외부 API 호출에 실패했을 때 등 쓰기 좋다. |

### 멀티캐스팅(hot publisher 로 변환) operator

* 하나의 flux 를 다수의 subscriber 에게 멀티캐스팅하기 위한 operator
* cold sequence 를 hot sequence 로 동작하게 한다.

| operator | 역할 |
| :--- | :--- |
| publish | subscribe 가 되더라도 바로 emit 되지 않고, concat () 을 호출하는 시점에 비로소 emit된다.<br>• hot seq 으로 변환되므로 구독 시점 이후에 emit 된 데이터마 받을 수 있다. |


## defer 를 사용해 불필요한 프로세스 실행 막기

자바와 코틀린 등 언어의 기본 특성상, 메서드의 파라미터로 전달된 함수는 해당 메서드가 호출되기 전에 먼저 평가(실행)된다.

### 1. 문제 상황

```kotlin
Mono.just("hello")
    .switchIfEmpty( Mono.just(dbClient.findDefault()) ) // 조립 시점에 즉시 실행됨
    .subscribe();

// 즉, 구독이 되기도 전에 switchIfEmpty 에 먼저 값을 넘기기 위해 Mono.just 의 데이터가 만들어지므로,
// 불필요하게 findDefault 가 실행된다.
```

위 코드에서 `switchIfEmpty`는 상위 스트림이 비어있을 때만 대체 동작을 수행해야 한다.

하지만 코드가 조립되는 시점에 자바는 파라미터 값을 미리 채워넣기 위해 `dbClient.findDefault()`를 즉시 실행해 버려서 Mono 에 데이터가 있음에도 쿼리가 실행된다.

### 2. 해결책: Mono.defer()를 통한 실행 지연

```kotlin
Mono.just("hello")
    .switchIfEmpty( Mono.defer(() -> Mono.just(dbClient.findDefault())) )
    .subscribe();
// defer 로 감싸진 lambda 가 switchIfEmpty 로 넘어간다. -> 처음 조립 시 람다만 넘어가고,
// 실제로 구독이 발생했을 때에야 람다가 실행되어 Mono 의 empty 유무에 따라 db 쿼리를 수행한다.
```

대체할 퍼블리셔를 `Mono.defer()`의 람다로 감싸서 넘기면, 평가 시점에 람다 객체 자체만 전달될 뿐이므로 람다식 내부는 실행되지 않는다.

이후 실제 구독 시점에 에 상위 스트림이 실제로 비어있어서 `switchIfEmpty`가 대체재를 구독해야 하는 상황이 왔을 때만 람다가 실행되어 DB 쿼리가 발생한다.
