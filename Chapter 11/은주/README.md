# Chapter 11. Context
## 11.1. Context 란?
- 어떠한 상황에서 그 상황을 처리하기 위한 정보
- Reactor 의 Context 는 ThreadLocal 과 다소 유사하지만, **각각 스레드와 매핑되는 ThreadLocal 과 달리 실행 스레드와 매핑되지 않고 Subscriber 와 매핑됨**
  - 즉, **구독 발생 시 해당 구독과 연결된 하나의 Context 가 생기는 것**
  - **Operator 체인에 전파되는 키, 값 형태의 저장소**
- Context 에 쓰인 데이터 읽기 방식
  1. 원본 데이터 소스 레벨에서 읽기 - deferContextual()
  2. Operator 체인의 중간에서 읽기 - transformDeferredContextual()
   - Context 에 데이터를 쓸 때는 Context 사용, 저장된 데이터 읽을 때는 ContextView 사용
- Reactor 에서는 Operator 체인 상의 서로 다른 스레드들이 Context 의 저장된 데이터에 손쉽게 접근 가능

## 11.3. Context 의 특징
- Context 는 구독이 발생할 때마다 하나의 Context 가 해당 구독에 연결된다
- Context 의 경우 Operator 체인 상의 아래에서 위로 전파되는 특징이 있다
- 일반적으로 모든 Operator 에서 Context 에 저장된 데이터를 읽을 수 있도록 contextWrite() 를 Operator 체인의 맨 마지막에 둔다
- Mono가 어떤 과정을 거치든 상관없이, 가장 마지막에 리턴된 Mono를 구독하기 직전 contextWrite()으로 데이터를 저장함 <br> 따라서, Operator 체인의 위쪽으로 전파되고, Operator 체인 어느 위치에서든 Context에 접근할 수있는 것
  - **Context는 인증 정보 같은 직교성(독립성)을 가지는 정보를 전송하는 데 적합하다**