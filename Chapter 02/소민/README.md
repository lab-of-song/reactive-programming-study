# 2. 리액티브 스트림즈(Reactive Streams)

<aside>
💡

리액티브하게 data stream 을 처리하는 라이브러리의 표준 사양

= 연속적으로 들어오는 data stream 을 non-blocking 하게, 비동기적으로 처리하는 리액티브 라이브러리의 표준 사양

</aside>

## 리액티브 스트림즈로 구현해야 하는 컴포넌트

- publisher: 데이터를 생성, 발행
- subscriber: publisher 로 부터 데이터를 받아서 처리하는 역할 (publisher 를 구독함)
- subscription: publisher 한테 데이터를 N개씩 줘! 라고 요청하고, 데이터 구독을 **취소**한다.
- processor: publisher 와 subscriber 두 개의 역할을 모두 할 수 있다. (sub 로써 다른 pub 을 구독하거나 pub 로써 다른 sub 이 얘를 구독할 수도 있다.)

<p align="center"><img width="1956" height="1110" alt="image" src="https://github.com/user-attachments/assets/d1e1d09d-7b7e-4959-8142-e87bac840e54" /></p>

<p align="center"><img width="511" height="221" alt="image" src="https://github.com/user-attachments/assets/8f0d406a-a433-46ec-ae7a-ca27e105d446" /></p>

<br>

> 근데 위 사진같은 거면 subscription 이 일종의 broker 아닌가… 그래도 구독이나 개수 전달, 취소는 sub이 직접 pub에 대고 하니까 카프카랑은 개념이 다른 것 같기도 하고

1. subscriber 가 먼저 publisher 한테 subscribe 으로 구독함 (publisher.subscribe(subscriber))
    1. 실제 구독하는 주체는 subscriber 이지만, reactive 구조상 publisher 에 subscriber 를 등록하는 것 처럼 코드가 됨
    2. subscriber는 람다로 정의할 수 있음
2. publisher는 onSubscribe 를 호출해서 sub 한테 이제 발행할게! 라고 알림 (subscriber.onSubscribe())
    1. 그럼 sub 은 onSubscribe 에 명시한 동작(구독 시작 시 subscriber 가 할 행동)을 수행함
3. sub 은 위의 알림을 받고 pub 에게 데이터 n개씩 달라고 알려줌 (Subscription.request(N개))
4. pub 은 요청받은 N개씩 데이터를 sub 에게 넘겨줌 (onNext) (subscriber.onNext())
    1. 그럼 sub 은 onNext 에 명시된 동작을 수행함
5. 계속 pub이 데이터 발행하다가 pub이 “이제 다 보냈어!” 하면 sub의 onComplete 를 트리거해서 다 끝났다고 알려줌
    1. 혹은 에러 발생하면 pub이 sub 에게 onError 트리거해서 알려줌
    2. 그럼 sub 은 onComplete, onError에 정의해둔 동작을 수행함 (subscriber.onError(), subscriber.onComplete())
    3. 데이터 다 받고난 다음에 subscriber 가 뭘 할지를 명시하려면 onComplete 에 명시하면 된다.
6. 필요 시, subscriber 가 먼저 구독을 취소할 수도 있다. (subscription.cancel())

**pub 과 sub 은 서로 다른 스레드에서 비동기적으로 상호작용한다.**

→ pub 발행 속도가 sub 보다 너무 빠르면 기다리는 데이터가 쌓여서 시스템 부하가 커진다. 그래서 size 를 처음에 합의하는 것

**중요: 카프카처럼 중간에 broker 가 있어서 pub → broker, broker ← sub 이런 구조가 아니다. reactive streams 에서는 pub 이랑 sub 이 직접 통신하기 때문에 구독 요청, 취소 요청 등 요청을 서로를 직접 호출해서 트리거해야한다.**

→ 카프카가 reactive 보다 더 pub, sub 이 느슨한 관계임

## 스트림즈 핵심 용어 정리

> signal

- pub 과 sub 이 서로간에 주고받는 상호작용 == signal    
- onSubscribe, onNext, request, cancel 등 위의 리액티브 스트림즈 api 를 호출하는게 서로간에 signal 을 주고받는 것
    
> demand

- sub이 pub 에게 요청했으나, 아직 받지 않은 데이터

> emit

- 메시지를 발행하는 것 (pub 이 sub 에게)
- onNext 시그널(pub이 sub 에게 n개씩 데이터르 전달)하는걸 “데이터를 emit한다” 라고 표현한다.

> upstream / downstream

- stream 구조에서 메서드 체이닝으로 동작이 명시되어 있을 때, 위에서 먼저 수행되는 동작이 아래 동작에게 upstream 이 되고 위의 동작에게 아래 동작이 downstream 이 된다.

> sequence

- publisher 가 emit 하는 데이터의 연속적인 흐름을 정의해둔 것
- emit 하고, filter 로 처리하고 등등.. 하는 저 위의 스트림 전체를 sequence 라고 부른다.

> operator

- 연산자

> source

 - 최초에 생성된 무언가

<br>

```bash
Flux.just(데이터) // 이 just 가 데이터를 생성한 후 emit 하는 역할을 한다. -> 얘가 publisher 역할
  .filter( n -> n > 1) // upstream
  .map(n -> n*2) // downstream, 위에서부터 여기까지 다 flux 타입이 반환된다.
  .subscribe(println()) // 구독 시 할 동작
  
  // 위의 동작은 일련의 순서대로 진행되도록 정의된 것이므로, 위 자체를 sequence 라고 부른다.
```


## 스트림즈 구현 규칙

**Publisher 구현을 위한 주요 기본 규칙**

1. Publisher가 Subscriber에게 보내는 onNext signal의 총 개수는 요청된 데이터의 총 개수보다 더 작거나 같아야 한다.
    - request 가 4개여도 3개를 보내도 무방함
2. Publisher는 요청된 것보다 적은 수의 onNext signal을 보내고 onComplete 또는 onError를 호출하여 구독을 종료할 수 있다.
    1. 종료 시점 pub 의 개수에 대한 내용, 종료 시점일 때 request 보다 적게 보내고 onComplete 를 호출할 수 있다.
3. Publisher의 데이터 처리가 실패하면 onError signal을 보내야 한다.
4. Publisher의 데이터 처리가 성공적으로 종료되면 onComplete signal을 보내야 한다.
5. Publisher가 Subscriber에게 onError 또는 onComplete signal을 보내는 경우 해당 Subscriber의 구독은 취소된 것으로 간주되어야 한다.
6. 일단 종료 상태 signal을 받으면(onError, onComplete) 더 이상 signal이 발생되지 않아야 한다.
7. 구독이 취소되면 Subscriber는 signal을 받는 것을 중지해야 한다.

**Subscriber 구현을 위한 주요 기본 규칙**

1. Subscriber는 Publisher로부터 onNext signal을 수신하기 위해 Subscription.request(n)를 통해 Demand signal을 Publisher에게 보내야 한다.
    1. 몇 개 받을지 sub이 먼저 pub 에게 알려줘야한다.
2. Subscriber.onComplete() 및 Subscriber.onError(Throwable t)는 Subscription 또는 Publisher의 메서드를 호출해서는 안 된다.
3. Subscriber.onComplete() 및 Subscriber.onError(Throwable t)는 signal을 수신한 후 구독이 취소된 것으로 간주해야 한다.
4. 구독이 더 이상 필요하지 않은 경우 Subscriber는 Subscription.cancel()을 호출해야 한다.
5. Subscriber.onSubscribe()는 지정된 Subscriber에 대해 최대 한 번만 호출되어야 한다.

**Subscription 구현을 위한 주요 기본 규칙**

1. 구독은 Subscriber가 onNext 또는 onSubscribe 내에서 동기적으로 Subscription.request를 호출하도록 허용해야 한다.
    1. 뭔소린지 모르겠음… onSubscribe 내에서 호출되는건 initial 시점의 request 개수일거고, onNext 중에 호출되는건 구도 중간에 구독 개수 바꾸는거라 생각했는데... 동기적으로 호출한다는게 request 호출 스레드(subscriber) 와 onNext 호출 스레드가 같을 수 있단 의미다. → 둘이 다른 스레드면 비동기적으로 돌면서 무한 재귀가 발생할 수 있으므로, 같은 스레드 내에서 진행되어서 반드시 순차호출이 되어야한단건가?
2. 구독이 취소된 후 추가적으로 호출되는 Subscription.request(long n)는 효력이 없어야 한다.
3. 구독이 취소된 후 추가적으로 호출되는 Subscription.cancel()은 효력이 없어야 하낟.
4. 구독이 취소되지 않은 동안 Subscription.request(long n)의 매개변수가 0보다 작거나 같으면 java.lang.IllegalArgumentException과 함께 onError signal을 보내야 한다.
5. 구독이 취소되지 않은 동안 Subscription.cancel()은 Publisher가 Subscriber에게 보내는 signal을 결국 중지하도록 요청해야 한다.
6. 구독이 취소되지 않은 동안 Subscription.cancel()은 Publisher에게 해당 구독자에 대한 참조를 결국 삭제하도록 요청해야 한다.
7. Subscription.cancel(), Subscription.request() 호출에 대한 응답으로 예외를 던지는 것을 허용하지 않는다.
    - Return normally : 유효한 값 이외에는 어떠한 예외도 던지지 않는다는 의미
8. 구독은 무제한 수의 request 호출을 지원해야 하고 최대 2^63-1개의 Demand를 지원해야 한다.
    1. 구독자가 무한한 개수의 데이터를 요청할 수 있고, 이런식으로 무한하게 발생하는 데이터 흐름은 무한스트림 이라고한다.
