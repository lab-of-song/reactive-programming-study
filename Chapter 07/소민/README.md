# 7.  Cold Sequence 와 Hot Sequence

```kotlin
Cold: 처음부터 무언가를 새로 시작한다.
Hot: 무언가를 새로 시작하지 않는다. (이미 진행된 중간 부터 시작)
```

## Cold Sequence

susbscriber 가 구독을 할 때 마다 데이터 흐름이 처음부터 다시 시작된다.

→ 즉, 한 publisher 에 대해 여러 subscriber 가 구독을 하면 각 susbscriber 마다 처음~끝까지의 데이터를 모두 담은 publisher 를 그때그때 새로 만들어준다.

- 만약 publisher 가 webClient 호출 결과를 받아온다면? subscriber 수 만큼 webClient 호출이 발생 → 그래야지 모두에게 같은 데이터를 처음부터 다 줄 수 있다.
- 즉, subscriber A보다 B가 조금 더 늦게 구독(A는 이미 데이터 몇 개를 consume 한 상황) 이어도 B도 A처럼 처음부터 데이터를 다 받을 수 있다.

**subscriber 가 붙을 때 마다 publisher 가 데이터를 처음부터 다시 emit 하는 흐름으로 동작하는 publisher 가 Cold Publisher**

> 궁금한 점:
> 1. 매번 publisher 를 새로 만들면 리소스 낭비 아닌가? 같은 세션(?) 등의 범위 내에서는 publisher 데이터를 처음 subscribe 시점에 한 번 만들어두고, subscriber 들이 동일한 사본(처음~끝의 데이터가 담겨있는) 을 각자 받아가서 쓰는 방식은 불가한가?
> 2. 지금처럼 매번 publisher 가 새로 데이터를 만들어서 emit 해주는 구조라면, 두 subscriber 가 붙는 사이에 새로운 데이터가 webClient 에 하나 더 추가되면 두 subscriber 가 구독하는 데이터가 달라지는 구조가 되는게 맞는지?
> → 그럼 만약 2개의 subscriber 가 완전히 동일한 데이터에 붙어야한다면 어떻게 해야하는가?
> 

일반적인 (별도 옵션을 지정하지 않은) publisher 는 Cold Publisher 이다.

## Hot Sequeunce

하나의 publisher 를 여러 subscriber 가 공유해서 사용한다. (share, cache 의 개념)

→ 따라서, subscriber 하나가 먼저 구독을 시작하고 나중에 subscriber B가 붙었다면 B는 자신이 구독하기 전에 emit 된 데이터는 받지 못한다.

- subscriber 가 구독을 시작한 이후부터 emit 된 데이터만 받을 수 있다.
- 즉, publisher 는 몇 개의 subscriber 가 오더라도 emit 하는 과정을 한 번만 거친다.

```kotlin
Flux.fromArray(something).share(); // N개의 subscriber 가 하나의 publisher 를 공유하는 hot publisher
// share: cold Flux publisher 를 hot publisher 로 바꿔주는 operator
// 원본 flux 를 여러 subscriber 가 공유한다.
```

```kotlin
Mono.just(a).cache(); // N개의 subscriber 가 한 개의 publisher 를 공유
// cache: cold Mono publisher 를 hot publisher 로 바꿔줌. (1개의 mono 를 캐시해두고 구독될 때 마다 캐시 전달)
// 캐시를 통해 인증토큰이 유효한 동안은 인증 토큰을 재발급 받지 않도록 해서 불필요한 api 콜을 줄일 수 있다.
// -> 유효 기간 동안만 cache 를 설정할 수 있는 것인가?
```

> 참고: cold sequence 에서는 구독이 발생하지 않으면 emit 이 발생하지 않는다.
>
