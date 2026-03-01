# Chapter 08. Backpressure
## 8.1. Backpressure 란?
- publisher 가 끊임없이 emit 하는 무수히 많은 데이터를 적절하게 제어하여 데이터 처리에 과부하 걸리지 않게 제어하는 역할

## 8.2. Reactor 에서의 Backpressure 처리 방식
### 8.2.1. 데이터 개수 제어
- Subscriber 가 데이터 요청 개수를 직접 제어하기 위해, Subscriber 인터페이스의 구현 클래스인 BaseSubscriber 사용 가능 
  - `hookOnSubscribe` : 구독 시점에 request() 메서드 호출 -> 최초 데이터 요청 개수 제어
  - `hookOnNext` : publisher 가 emit 한 데이터 전달받아 처리 후 publisher 에게 또 다시 데이터 요청

### 8.2.2. Backpressure 전략 사용
- `IGNORE` : backpressure 적용하지 않음
- `ERROR` : Downstream 의 데이터 처리 속도가 느려서 Upstream 의 emit 속도 따라가지 못할 경우 예외 발생
  - pubisher 는 error signal 을 subscriber 에게 전송 후 삭제한 데이터 폐기
- `DROP` : publisher 가 downstream 으로 전달할 데이터가 버퍼에 가득 찰 경우, 버퍼 밖에서 대기중인 데이터 중 먼저 emit 된 데이터부터 drop 시키는 전략
- `LATEST` : publisher 가 downstream 으로 전달할 데이터가 버퍼에 가득 찰 경우, 버퍼 밖에서 대기중인 데이터 중 가장 최근에 (나중에) emit 된 데이터부터 버퍼에 채우는 전략
  - 새로운 데이터가 들어오는 시점에 가장 최근 데이터만 남겨두고 나머지 데이터 폐기
- `BUFFER` : 버퍼가 가득 찼을 때 버퍼 바깥쪽이 아닌 버퍼 안에 있는 데이터 폐기
- `DROP_LATEST` : publisher가 Downstream으로 전달할 데이터가 버퍼에 가득 찰 경우, 가장 최근에(나중에) 버퍼 안에 채워진 데이터를 Drop하여 폐기 -> 확보된 공간에 emit된 데이터를 채우는 전략
- `DROP_OLDEST` : publisher가 Downstream으로 전달할 데이터가 버퍼에 가득 찰 경우, 버퍼 안에 채워진 데이터 중에서 가장 오래된 데이터를 Drop하여 폐기 -> 확보된 공간에 emit된 데이터를 채우는 전략