# 8. Backpressure

```kotlin
publisher 가 끊임없이 emit 하는 데이터를 subscriber 가 따라가지 못해서 데이터 처리에 과부하가 걸리지 않도록
적절하게 중간에서 제어해주는게 backpressure 의 역할이다.
```

publisher 가 emit 하는 데이터를 subscriber 가 제때 처리하지 못하면 오버플로가 발생하거나 시스템이 다운된다.

## 1) Request 개수로 subscriber 가 받을 수 있는 만큼씩만 emit하도록 조치

```kotlin
.doRequest(publisher 가 sub 으로부터 n개의 데이터를 보내라고 request 받았을 때, 그 개수를 받아서 로그를 찍거나 가공할 수 있는 람다)
// request 요청을 받을 때 마다 실행됨

// request 를 명시하기 위해서는 BaseSubscriber 를 상속해서 `hookOnSubscribe` 에 request(N)을 명시해야한다.
.subscribe(new BaseSubscriber<Integer>() { 
  @Override
  protected void hookOnSubscribe(Subscription sub) { 
    request(1); // subscribe 시점에 개수를 알리는 것으로 onSubscribe(구독 직후 1회 트리거) 와 동일한 역할
    // 최초 요청 개수 명시 
    }
    
  @Override
  protected void hookOnNext(Integer value) {
    // do someting with subscribed value
    request(1); // onNext 는 request 한 것 만큼 데이터를 받은 후에 실행되는 부분으로, 다음에는 데이터를 N개 보내라고 여기에 명시한다.
   }
 }
```

## 2) Reactor 가 제공해주는 Backpressure 전략 사용

### 1. Ignore: 백프레셔를 사용하지 않는다. (처리 하지 않으므로 IllegalStateException 발생 가능)

### 2. Error: subscriber 가 따라가지 못하면 exception 발생 (pub 이 sub 에게 error signal 전송 후 폐기)

```kotlin
.onBackpressureError() // 에러 전략 사용
-> OverflowException 발생
```

### 3. Drop 전략: downstream 이 다시 받을 수 있는 상태가 될 때 까지는 emit 된 데이터들 중 앞의 데이터부터 버린다.

downstream 의 버퍼가 가득찼을 때, 버퍼 밖에서 대기중인 데니터 중에서 먼저 emit 된 데이터부터(즉 오래된 데이터부터) 폐기한다.

- 데이터를 받을 수 있는 상태 == 버퍼에 공간이 있는 상태 == downstream 이 requset 를 보낸 상태 (request 를 명시하지 않을 뿐, 이런 시그널이 오가고 있다.)
- 따라서, downstream 이 더 수용하겠다고 하기 전까지는(데이터 수용이 막힌 동안, 버퍼가 비워지기 전까지) 계속 데이터를 drop 하게 된다.

```kotlin
.onBackpressureDrop(dropped -> drop 된 데이터가 폐기 되기 전에 추가 처리 가능)
```

### 4. Latest 전략: downstream 이 다시 받을 수 있는 상태가 되기 전까지 마지막에 emit 된 데이터 하나만 두고 나머지는 다 버린다.

- downstream 이 받을 수 있는 상태가 아니어도, 가장 마지막에 emit 된거 하나까지는 들고 있어주는 그런 느낌..

```kotlin
.onBackpressureLatest()
```

### 5. Buffer 전략

버퍼 데이터를 폐기하지 않고 버퍼링하는 전략 / **버퍼가 가득차면 버퍼 내 데이터를 버리는 전략** / 버퍼가 가득차면 에러 발생시키는 전략

**drop , latest 와 다르게 버퍼 안에 있는 데이터를 폐기한다.**

**Drop_latest**

- 버퍼가 모두 가득 찼을 때, 가득찬 버퍼에 들어오려하는걸 버린다. (즉 오버플로우를 유발하는 데이터를 바로 버리는 것)

```kotlin
.onBackpressureBuffer(버퍼 사이즈, dropped -> doSomething(), BufferOverflowStrategy.DROP_LATEST);
 버퍼 크기, drop 된 데이터가 폐기되기 전 처리, 버퍼 전략
```

**buffer 전략에서는 사용할 버퍼의 개수를 명시할 수 있다. 그래서 앞의 drop, latest 전략은 단순히 subsriber(downstream) 의 처리 속도에 따라 데이터가 버려졌다면  buffer 전략은 명시한 버퍼의 개수의 영향을 받는다.**

- 위의 drop, latest 도 내부적으로 downstream 이 요청을 받느 queue(버퍼)를 내부에서 가지고 있어서 이걸로 요청을 받을 수 있는 상태/아닌 상태(버퍼가 찬 상태, 아닌 상태)가 결정된다.
- buffer 전략은 여기에 추가적인 버퍼가 더 있는 개념??

**Drop_oldest**

- latset 와 유사하나, 버퍼가 가득 찬 상태에서 버퍼에 새 데이터가 들어오면 버퍼에 있는 가장 오래된 데이터를 버린다.
- 즉, 강제로 공간을 만들어서 새로온 데이터를 어떻게든 버퍼에 넣어준다.
