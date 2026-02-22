# Chapter 07. Cold Sequence 와 Hot Sequence
## 7.1. Cold 와 Hot 의 의미
- Cold 는 무언가를 새로 시작하고, Hot 은 무언가를 새로 시작하지 않는다

## 7.2. Cold Sequence
- Subscriber 가 구독할 때마다 데이터 흐름이 처음부터 다시 시작됨. 즉, 구독 시점이 달라도 구독할 때마다 publisher 가 처음부터 데이터를 emit

## 7.3. Hot Sequence
- 구독이 발생한 시점 이전에 Publisher 로부터 emit 된 데이터는 Subscriber 가 전달받지 못하고, 구독이 발생한 시점 이후에 emit 된 데이터만 전달 받을 수 있다
- operator 체인 형태로 리턴되는 flux 들은 **모두 다른 참조값을 가짐** <br>원본 flux 를 멀티캐스트 (공유) 한다는 의미는 **여러 subscriber 가 하나의 원본 flux 를 공유한다는 의미**

## 7.4. HTTP 요청과 응답에서 Cold Sequence 와 Hot Sequence 의 동작 흐름
- 구독이 발생할 때마다 데이터 emit 과정이 새로 시작되는 Cold Sequence 는 2번의 HTTP 요청 발생