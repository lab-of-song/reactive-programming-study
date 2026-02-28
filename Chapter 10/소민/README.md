# 10. Scheduler

```kotlin
scheduler: reactor sequ4ence 에서 사용되는 스레드를 관리하는 관리자
-> 시퀀스의 publisher/subscriber 의 작업이 실행될 스레드를 scheduler 를 통해 지정해줄 수 있다.
```

## 물리 코어, 물리 스레드 개념

- 코어 :CPU  명령을 처리하는 반도체 유닛 (실제 존재하는 물리적인 것)
- 물리적 스레드: 물리적인 코어를 논리적인 스레드로 나눈 것 → 단, 실제 **물리적으로 있는 코어를 하드웨어 차원에서 스레드로 나눈 것이므로 스레드의 측면에서는 물리적 스레드**
    - == 논리적 코어
- 논리적 스레드: 프로그램 레벨에서 소프트웨어적으로 관리하는 스레드
    - context switching 이 되는 스레드

<img width="864" height="498" alt="image" src="https://github.com/user-attachments/assets/4d390f77-8812-4d9c-9491-68402f1bcf96" />


### 병렬성 VS 동시성

- 병렬성: 실제로 동시에 여러 개의 작업이 수행되는 것
- 동시성: 동시에 실행되는 것 처럼 보이는 것 → 논리적 스레드 N개가 물리적 스레드를 번갈아 빠르게 점유하면서 동작하므로, 겉보기에는 마치 동시 실행되는 것 처럼 보인다.

## Scheduler: 어떤 작업을 어떤 스레드에서 실행시킬 것인지 제어

- 운영체제 레벨의 스케쥴러: 프로세스를 선택하고, 실행하여 프로세스의 라이프 사이클을 관리하는 역할
- reactor 의 스케줄러: 비동기 프로그래밍을 위해 사용되는 스레드를 관리하는 역할]

### subscribeOn(scheduler)

**구독이 발생한 직후에 실행될 스레드를 지정하는 operator**

→ 즉, **구독 발생 직후이므로 원천 publisher 가 동작할 스레드만 지정한다.**

```kotlin
Flux.fromArray()
  .subscribeOn(Scheduler.boundedElastic()) // 구독 발생한 직후 원본 publisher 가 동작할 스레드
  // 즉 원본 데이터의 emit 이 위에서 지정한 스레드에서 진행된다.
  .doOnNext(data -> publisher 가 emit 시 수행) 
  .doOnSubscribe(subscription -> 구독 직후 수행할 동작) // 구독 발생 시점에 추가적인 액션 처리 가능
  .subscribe(subscriber); 
```

1. doDoSubscribe 가 먼저 스레드 설정되기 전(main) 스레드에서 실행된다. 
2. subscribeOn 으로 스케줄러 지정했으므로, 데이터 emit 은 다른 스레드에서 실행된다. (bounded pool thread)
    1. doOnNext 도 원천 데이터의 emit 이 발생한 (즉 분리된) 스레드에서 진행된다.
    2. operator 를 분리하지 않았기 때문에 subscriber 도 같은 bounded thread 에서 실행된다. 
        1. 왜???? 원천 publisher 만 제어하는거 아닌가..??

### publishOn(scheduler)

**publisher 는 downstream 으로 데이터를 emit 하므로, publishOn (퍼블리시 시점의 스레드 설정) 의 영향을 받는건 unstream publisher 의 emit 된 데이터를 받는 downstream publisher이다.**

⇒ publishOn() 의 바로 아래 체인에 있는 downstream  publisher 의 실행 스레드를 변경한다.

> **publishOn 의 이후부터 오는 operator 들은 중간에 다른 publishOn 으로 스레드가 바뀌지 않는 이상, 모두 같은 스레드에서 실행된다!!!!**
> 

```kotlin
Flux.fromArray() // 원천 데이터는 따로 스레드 지정되지 않았으므로 main 에서 emit 시작
  .doOnNext(data -> 원천 emit 후 수행)  // main
  .doOnSubscribe(subscription -> 구독 직후 동작) // main
  .publishOn(Schedulers.parallel())
  // 바로 아래 subscribe 의 실행 스레드가 parallel 스레드가 된다.
  .subscribe(data -> doSomething());
```

```kotlin
// publishOn 하위의 모든 operator 는 같은 스레드
Flux.fromArray()
  .doOnNext(data -> 원천 emit 후 수행)  // main
  .doOnSubscribe(subscription -> 구독 직후 동작) // main
  .publishOn(Schedulers.parallel()) // parallel
  .doOnNext(something()) // parallel
  .subscribe(data -> doSomething()); // parallel
```

```kotlin
// publishOn 이 새로 나오면 그 하위부터는 다른 스레드
Flux.fromArray()
  .doOnNext(data -> 원천 emit 후 수행)  // main
  .doOnSubscribe(subscription -> 구독 직후 동작) // main
  .publishOn(Schedulers.parallel()) // parallel
  .doOnNext(something()) // parallel
  .publishOn(Schedulers.boundedElastic()) // bounded pool
  .subscribe(data -> doSomething()); // bounded pool
```

```kotlin
// subscribeOn 과 publishOn 을 함께 사용하면 각각 영향 범위에 영향을 준다.
Flux.fromArray() // A thread
  .subscribeOn(Schdulers A thread) // 위의 원천데이터 emit: A thread 에서 수행하도록
  // 체인에서 위치가 어디에 있든 항상 원천 데이터의 emit 직전에 실행 스레드를 바꾼다!!!
  .doOnNext(data -> 원천 emit 후 수행)  // A thread
  .doOnSubscribe(subscription -> 구독 직후 동작) // main
  .publishOn(Schedulers B thread) // B thread
  .doOnNext(something()) // B thread
  .publishOn(Schedulers C thread) // C thread
  .subscribe(data -> doSomething()); // C thread
```

### parallel()

subscribeOn, publishOn 과 다르게 **실제 물리 스레드의 병렬 실행에 관여한다.**

- 라운드로빈 방식으로 논리 코어 개수(==물리 스레드 개수)만큼 스레드를 병렬로 실행한다.
- 실제로 실행을 관리하진 않고, N개의 flux 데이터가 있으면 이를 물리 스레드에 공평하게 분배하는 역할을 한다.

```kotlin
Flux.fromArray(1,2,3,4,5,6,7,8,9,10)
  .parallel() // emit 되는 데이터를 논리 스레드에 맞춰 골고루 분배
  .runOn(Schedulers.parallel()) // 실제 병렬 수행할 스레드의 할당을 담당
  // 4코어 8스레드 컴퓨터라면, 1~8번 parallel 스레드에서 실행된다.
  // 물리적 스레드를 다 쓸 필요가 없다면, parallel(N) 으로 개수 지정 가능
  .subscribe(doSomething());
```

## scheduler 종류

### Schedulers.immediate()

별도 스레드를 안쓰고 지금 스레드에서 작업을 처리한다.

```kotlin
Flux.fromArray() // main
  .doOnNext(data -> 원천 emit 후 수행)
  .doOnSubscribe(subscription -> 구독 직후 동작) // main
  .publishOn(Schedulers B thread) // B thread
  .doOnNext(something()) // B thread
  .publishOn(Schedulers.immediate()) // B thread
  .subscribe(data -> doSomething()); // B thread
```

> 그냥 비워두지 않고 굳이 immediate 쓰는 이유가 스레드는 필요하지만 별도의 스레드를 추가 할당하고 싶지 않을때라는데,,, 뭔소린지???
> 

### Schedulers.single()

스레드 하나만 생성해서 scheduler 가 제거되기 전까지 재사용

schedulers.single()을 사용하는 flux 의 함수를 하나 만들어두고, 이 함수를  여러번 호출하면 매번 동일한 스레드에서 실행된다.

→ 지연시간이 짧은 작업이 좋다. (다수 작업을 한 스레돌 처리해야하므로)

→ single-1 내에서 모든 작업이 수행

### Schedulers.newSingle()

호출될 때 마다 매번 다른 single-N 스레드를 생성한다.

schedulers.newSingle() 을 사용하는 flux 함수 만들어두고 여러번 재사용하면 재사용될 때 마다 새로운 single-N 스레드에서 실행된다.

```kotlin
schedulers.newSingle("스레드 이름", deamon 여부)
// deamon 이란? 보조 스레드로, 주 스레드가 종료되면 자동 종료된다.
```

### Schedulers.boundedElastic()

executorService 기반의 스레드 풀을 만들어두고, 만들어진 풀 내에서 스레드를 사용해서 작업 처리 → 종료된 스레드는 풀에 반환한다.

- 보통 blocking I/O 같은 작업을 효과적으로 처리하는데 좋다.
    - non-blocking 작업에 영향을 주지 않도록 전용 스레드를 할당해서 blocking I/O 를 처리한다. → 처리시간 효율적으로 통제 가능

### schedulers.parallel()

non-blocking I/O에 최적화 되어있으며(즉 CPU 를 많이 잡아먹는 연산에 최적화) 물리적 스레드 수 만큼 병렬 처리 가능

### Schedulers.fromExecutorService()

기존 executorService 를 재사용 가능하나, 리액터에서 권장 X

### Schedulers.newXXXX()

newSingle, newBoundedElastic, newParallel 등으로 새로운 scheduler 인스턴스를 만들 수 있다. (모든걸 커스텀)
