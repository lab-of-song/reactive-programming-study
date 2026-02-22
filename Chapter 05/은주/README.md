# Chapter 05. Reactor 개요
## 5.1. Reactor 란?
- Flux[N]
  - 0개부터 N개까지, 즉 무한대의 데이터를 emit 할 수 있는 Reactor 의 Publisher
- Mono[0|1]
  - 데이터를 한 건도 emit 하지 않거나, 단 한 건만 emit 하는 단발성 데이터 emit 에 특화된 Publisher
- Backpressure-ready network 
  - publisher 로부터 전달받은 데이터를 처리하는 데 있어서 과부하 걸리지 않도록 제어하는 백프레셔 지원
- Reactor 의 핵심 구성 요소
  - 1단계 : 데이터를 생성해서 제공하고
  - 2단계 : 데이터를 가공한 후
  - 3단계 : 전달받은 데이터를 처리한다