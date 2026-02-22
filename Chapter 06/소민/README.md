# 6. 마블 다이어그램

마블(구슬)처럼 생긴 숫자로 reactive 타임라인을 표현한다. → operator 들의 이해해 도움이 된다. 

<img width="3052" height="2044" alt="image" src="https://github.com/user-attachments/assets/6f4fe6e8-5038-41d2-b68e-a4706e7c76ed" />


1. upstream publisher (source flux) 가 데이터소스들을 emit 한다. (왼쪽부터 순차적으로 emit)
2. upstream publisher 에서 모든 데이터의 emit 이 끝나면 `|` 으로 표기되는 onComplete 시그널을 보낸다.
3. 중간의 operator 에서 upstream flux 를 가공하여 가공된 데이터를 내보내는 새 downstream publisher 를 만들어준다.
- flux chaining 을 통하면 아래 chain 이 downstream 이 된다.
1. downstream publisher 도 마찬가지로 왼쪽부터 순차적으로 데이터를 전달한다. 
- operator 의 출력으로 만들어진 flux 는 output flux 라고도 한다.
1. 에러 발생으로 데이터 처리가 종료된 경우는 `X` 로 표기하며 onerror signal 을 의미한다.

⇒ 비동기적인 데이터 흐름을 시간의 흐름으로 표현한 다이어그램

참고) 위의 다이어그램은 마블이 여러개, 즉 emit 되는 데이터가 여러개이므로 flux publisher 이다.

## reactor 의 publisher


### Mono

<img width="3152" height="1692" alt="image" src="https://github.com/user-attachments/assets/8b36f9cd-b1c0-438e-976f-fceeb34f5b21" />

단 하나의 데이터를 emit 하는 publisher → 위의 다이어그램에서도 한 개의 데이터만 표기된다.

0 or 1 이므로, `3` 이라고 표기된 데이터가 emit 되지 않고 onComplete 시그널만 전송될 수도 있다.

```kotlin
Mono.just("Hello world") // 1개의 데이터만 emit
  .subscribe(sysout)
```

```kotlin
참고)
원래 Publisher.just() 는 한 개의 데이터만 emit 하므로, flux 처럼 여러개의 데이터가 전달될 경우 내부적으로 
fromArray() operator 를 사용해서 데이터를 emit 한다.
```

```kotlin
Mono.empty()
.subscribe(
  none -> sysout(onNext called) 
  error -> {}
  () -> sysout(onComplete)
}
```

0건을 emit 하는 경우 위와 같이 동작한다.

emit 할 데이터가 없으므로 바로 onComplete 시그널이 호출된다.

**⇒ 주로 데이터를 전달할 필요는 없으나, 특정 작업이 끝났음을 알리고 이에 따른 후처리(subscriber) 를 할 때 사용**

RestTemplate 같은 non-blocking 을 reactor 로 감싸는 경우, api 의 응답이 1개의  response 이므로 아래와 같이 mono 로 처리할 수 있다.

```kotlin
Mono.just(
  restTemplate.exchange(uri, method, body)
  )
  .map(response -> doSomething())
  .subscribe(
    data -> sysout
    error -> syserr
  )
```


### Flux

<img width="3026" height="1177" alt="image" src="https://github.com/user-attachments/assets/a9fda8c9-7fba-4a7a-9fb4-67ee836bf7ce" />


아래처럼 array 를 원천 데이터로 줄 수도 있다.

```kotlin
Flux.fromArray(new Integer[] {3,6,7,9}) // fromArray operator 를 사용한다.
  .filter(n -> n>6)
  .map(toString())
  .subscribe(systout)
```


### 여러 publisher 연동해서 flux 로 만들기

```kotlin
Flux<String> flux = Mono.justOrEmpty("steve")
    .concatWith(Mono.justOrEmpty("jobs"));
// 2개의 mono 를 혼합해서 여러 데이터를 emit 하는 flux publisher 를 새로 생성
flux.subscribe(sysout);

// concatWith 은 A.concatwith(B) 로 2개만 연결 가능
```

<img width="3088" height="1352" alt="image" src="https://github.com/user-attachments/assets/e90c5601-2541-4272-a164-63fa63afc9f6" />


```kotlin
Flux.concat( // 2개 이상의 소스를 연결 가능
  Flux.just("1","2","3"),
  Flux.just("hello", "world"),
  Flux.just("other", "thing")
) // 9개의 원천 데이터이므로 flux
  .collectList() // upstream 에서 emit 되는 데이터를 모아서 List 로 변환된 새로운 데이터소스로 만든다.
  // list 자체는 하나의 객체이므로 위의 결과는 Mono 가 된다.
  .subscribe(sysout); // list 라는 1개의 객체가 되므로 Mono.subscribe 가 된다.
  // 출력데이터: List
```
