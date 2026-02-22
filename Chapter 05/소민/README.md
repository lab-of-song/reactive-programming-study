# 5.  Reactor 개요

## Reactor 란

spring 에서 제공하는 reactive streams 의 구현체 → spring webflux 라이브러리안에 reactor core 라이브러리가 포함되어 있다.

### reactor 의 publisher 종류 2개

1. **Flux[N]**

N 개의 데이터를 emit 하는 publisher

1. **Mono[0|1]**

0개 또는 1개의 데이터를 emit 하는 publisher

- 단발성 emit 에 특화

### backpressure-ready network

backpressure: publisher 로 부터 받은 데이터를 처리하면서 과부하에 걸리지 않도록 지원 → subscriber 가 publisher 가 emit 하는 대량의 데이터를 적절하게 처리하기 위한 제어 방법

```kotlin
Flux<String> sequence = Flux.just("Hello", "world"); // just 내부 데이터: 가공되지 않은 원천 데이터소스
// flux: publisher
// just: list 안의 데이터들을 한 번에 emit 하는 operator => 데이터 생성 후 제공하는 operator
sequence.map(data -> data.toLowerCase()) // map: operator
  .subscriber(data -> sysout) // 내부의 람다가 subscriber
 
```

❓한 번 subscribe 된 데이터는 재사용을 못했던 것 같은데 맞나..
