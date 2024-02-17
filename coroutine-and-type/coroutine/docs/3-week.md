# 16 채널

- 채널에는 송신자와 수신자가 존재
	- 한번 보내고 받은 값은 다시 받을 수 없다.
	- Channel 은 두 개의 서로 다른 인터페이스를 구현한 하나의 인터페이스
		- SendChannel
			- 원소를 보내거나(더하거나) 채널을 닫는 용도
		- ReceiveChannel
			- 원소를 받을 때(꺼낼 때) 사용
코틀린의 `SendChannel`과 `ReceiveChannel`의 인터페이스 정의를 보여드리겠습니다. 이 두 인터페이스는 각각 데이터를 송신하고 수신하는 데 사용됩니다.

```

`import kotlinx.coroutines.channels.SendChannel
import kotlinx.coroutines.channels.ReceiveChannel

// SendChannel 인터페이스 정의
interface SendChannel<in E> {
    suspend fun send(element: E) // 데이터 송신 함수
    fun close(cause: Throwable? = null): Boolean // 채널 닫기 함수
}

// ReceiveChannel 인터페이스 정의
interface ReceiveChannel<out E> {
    suspend fun receive(): E // 데이터 수신 함수
    val isEmpty: Boolean // 채널이 비어있는지 여부 확인
    val isClosedForReceive: Boolean // 채널이 수신을 위해 닫혔는지 여부 확인
}` 
```
-   `SendChannel`: 데이터를 송신하는 채널을 표현하는 인터페이스입니다. `send` 함수는 데이터를 송신하며, `close` 함수는 채널을 닫습니다.
    
-   `ReceiveChannel`: 데이터를 수신하는 채널을 표현하는 인터페이스입니다. `receive` 함수는 데이터를 수신하며, `isEmpty` 속성은 채널이 비어있는지 여부를 확인하고, `isClosedForReceive` 속성은 채널이 수신을 위해 닫혔는지 여부를 확인합니다.
    

`SendChannel`과 `ReceiveChannel`은 공변성을 갖고 있습니다. `SendChannel`은 입력 위치의 공변성을 가지며(`in E`), `ReceiveChannel`은 출력 위치의 공변성을 가집니다(`out E`). 이는 송신 채널은 데이터를 송신하는 데 사용되며, 수신 채널은 데이터를 수신하는 데 사용되기 때문입니다.

-  send, receive 는 중단함수임
	- send
		- 채널의 용량이 꽉 찼을 때 중단된다.
	- receive
		- 채널의 원소가 없으면 원소가 들어올 때까지 중단된다.
- 중단 함수가 아닌 함수로 보내거나 받아야 한다면 trySend, tryReceive 를 사용할 수 있다.
	- 연산이 성공했는지 실패했는지 정보를 담고 있는 ChannelResult 를 즉시 반환한다.
	- 용량이 제한적인 채널에서만 사용해야 한다.
	- 랑데뷰 채널(버퍼가 없는)에서는 동작하지 않는다.
- 채널은 송신자와 수신자의 수에 제한이 없다.


코틀린에서 `Channel`을 사용한 간단한 예제를 보여드리겠습니다. `Channel`은 코루틴 간에 데이터를 안전하게 전달하고 통신하는 데 사용됩니다.

```

`import kotlinx.coroutines.channels.Channel
import kotlinx.coroutines.launch
import kotlinx.coroutines.runBlocking

fun main() = runBlocking {
    // Channel 생성 (버퍼 크기 지정 가능)
    val channel = Channel<Int>(capacity = 5)

    // 송신자 코루틴 시작
    val sender = launch {
        repeat(10) {
            println("Sending $it")
            channel.send(it)
        }
        channel.close() // 모든 데이터 전송 후 채널 닫기
    }

    // 수신자 코루틴 시작
    val receiver = launch {
        repeat(5) {
            val value = channel.receive()
            println("Received $value")
        }
    }

    // 송신자와 수신자가 끝날 때까지 대기
    sender.join()
    receiver.join()
}` 
```
이 예제에서는 `Channel`을 사용하여 채널을 생성하고, 송신자(`sender`)와 수신자(`receiver`) 코루틴을 시작합니다. 송신자는 0에서 9까지의 숫자를 채널에 보내고, 수신자는 채널에서 데이터를 받아 출력합니다. `capacity` 매개변수를 사용하여 채널의 버퍼 크기를 조절할 수 있습니다.

`channel.send()`는 데이터를 채널에 보내고, `channel.receive()`는 채널에서 데이터를 받습니다. 채널은 버퍼가 가득 차거나 비어있을 때 송신자가 또는 수신자가 대기하게 됩니다.

마지막으로, 모든 데이터가 전송된 후에는 `channel.close()`를 호출하여 채널을 닫습니다. 이를 통해 수신자는 더 이상의 데이터가 없음을 알 수 있습니다.

- produce 빌더를 사용하면 코루틴이 어떻게 종료되든 상관없이 채널을 닫는다.
	- 끝나거나, 중단되거나, 취소되거나
		- 중단되거나..?
## 채널 타입
- 설정한 용량 타입에 따라 채널을 네 가지로 구분할 수 있다.
- 무제한(Unlimited)
	- 제한이 없는 용량 버퍼를 가진 채널로, Channel.UNLIMITED 로 설정된 채널
	- send 가 중단되지 않는다.
- 버퍼(Buffered)
	- 특정 용량 크기 또는 Channel.BUFFERED(기본값 64) 인 채널
	- JBM 의 kotlinx.coroutines.channels.defaultBuffer 설정할 시 기본값을 오버라이드 가능
- 랑데뷰(Rendezvous)
	- 용량이 0 이거나 Channel.RENDEZVOUS(용량이 0) 인 채널
	- 송신자와 수신자가 만날 때만 원소를 교환한다.
- 융합(Conflated)
	- 버퍼 크기가 1인 Channel.CONFLATED 를 가진 채널
	- 새로운 원소가 이전 원소를 대체한다.
- 이들은 channel 에 직접 설정도 가능하지만 produce 빌더에(함수를 호출하여) 설정할 수도 있다.
```
import kotlinx.coroutines.channels.Channel
import kotlinx.coroutines.launch
import kotlinx.coroutines.runBlocking

fun main() = runBlocking {
    val channel = Channel<Int>(capacity = Channel.UNLIMITED)

    // 송신자 코루틴 시작
    val sender = launch {
        var counter = 1
        while (true) {
            channel.send(counter++)
            // 적절한 딜레이를 추가하여 무한하게 데이터를 생성하도록 함
            kotlinx.coroutines.delay(1000)
        }
    }

    // 수신자 코루틴 시작
    val receiver = launch {
        repeat(10) { // 필요한 만큼 데이터 수신
            val data = channel.receive()
            println("Received: $data")
        }
        // 수신이 끝난 후 송신자에게 채널 닫기를 요청
        sender.cancel()
    }

    // 송신자와 수신자가 끝날 때까지 대기
    sender.join()
    receiver.join()
}
```

## 버퍼 오버플로우일 때
- 채널을 커스텀화할 때 버퍼가 꽉 찼을 때(onBufferOverflow 파라미터)의 행동을 정의할 수 있음
	- SUSPEND(기본값)
		- buffer 가 가득 찼을 때, send 메서드가 중단된다.
	- DROP_OLDEST
		- 버퍼가 가득 찼을 때, 가장 오래된 원소가 제거된다.
	- DROP_LATEST
		- 버퍼가 가득 찼을 때, 가장 최근의 원소가 제거된다.
- produce 함수에서는 onBufferOverflow 를 설정할 수 없으므로, 채널 함수를 사용해 채널을 정의해야 하낟.
	- val channel = Channel<Int>(capacity = 2, onBufferOverflow = BufferOverflow.DROP_OLDEST)

## 전달되지 않은 원소 핸들러
- onUndeliveredElement 파라미터가 존재
	- 원소가 어떠한 이유로 처리되지 않을 때 호출된다.
		- 대부분 채널이 닫히거나 취소된 상황이다.
		- send, receive, receiveOrNull 또는 hasNext 가 에러를 던질 때 발생할 수 있다.
		- 주로 채널에서 보낸 자원을 닫을 때 사용한다.
	- val channel = Channel<Resource>(capacity) { resource -> resource.close() }
	- val channel = Channel<Resource>(capacity, onUndeliveredElement = { resource -> resource.close() })
## 팬아웃(Fan-out)
- 여러 개의 코루틴이 하나의 채널로부터 원소를 받을 수도 있다.
- 하나의 채널을 수신하는 리시버가 여러개 생성되면 FIFO 방식으로 수신한다.

## 팬인(Fan-In)
- 여러 개의 코루틴이 하나의 채널로 원소를 전송할 수 있다.
- 다수의 채널을 하나의 채널로 합쳐야 할 경우가 있다.
	- produce 함수로 여러 개의 채널을 합치는 fanIn 함수를 사용할 수 있다.
```
fun <T> CoroutineScope.fanIn(
	channels: List<ReceiveChannel<T>>
): ReceiveChannel<T> = produce {
	for (channel in channels) {
		launch {
			for (elem in channel) {
				send(elem)
			}
		}
	}
}
```
## 파이프라인
- 한 채널로부터 받은 원소를 다른 채널로 전송하는 경우가 있다.
- 이를 파이프라인이라 한다.

## 통신의 기본 형태로서의 채널
- 채널은 서로 다른 코루틴이 통신할 때 유용하다.
	- 충돌이 발생하지 않으며(공유 상태로 인한 문제가 일어나지 않는다), 공평함을 보장한다.

# 17 셀렉트
- - 실험적 기능이다.
- `select` 구문은 코틀린 코루틴에서 여러 채널이나 작업 중에서 첫 번째로 완료된 작업을 선택하는 데 사용되는 특별한 구문입니다. 다음은 `select`의 구성과 동작에 대한 상세한 설명입니다:

1.  **구문 형태:**
    
    ```
    `val result = select<T> {
        onReceive(channel) { value ->
            // 채널에서 데이터를 수신한 경우
            // value 변수에 수신된 데이터가 들어옴
            value
        }
        onAwait { job ->
            // 작업이 완료된 경우 (delay 또는 비동기 작업 완료)
            // job 변수는 완료된 작업을 나타냄
            // 이 블록에서 반환한 값은 select 구문의 결과로 사용됨
            // (일반적으로 job 변수를 통해 결과를 가져옴)
            // 예: job.getCompletion()
        }
        // 다른 onX 블록들도 추가 가능
    }` 
    ```
2.  **동시적 모니터링:**
    
    -   `select`는 여러 채널이나 작업을 동시에 모니터링할 수 있습니다.
    -   여러 `onReceive` 또는 `onAwait` 블록을 `select` 내부에 추가하여 각각의 채널이나 작업을 지정할 수 있습니다.
3.  **첫 번째 완료된 작업 선택:**
    
    -   `select`는 여러 블록 중에서 가장 먼저 완료된 작업의 결과를 반환합니다.
    -   채널 수신이나 작업 완료가 동시에 발생하면 임의로 선택됩니다.
4.  **비동기 작업 대기:**
    
    -   `onAwait` 블록은 비동기 작업의 완료를 기다립니다.
    -   `delay` 함수를 사용하는 경우나 다른 코루틴이 완료되기를 기다리는 경우에 활용됩니다.
5.  **결과 처리:**
    
    -   `select` 구문의 결과는 선택된 블록에서 반환한 값입니다.
    -   이 값은 `val result = ...` 구문을 통해 저장되거나 직접 사용할 수 있습니다.
6.  **작업 종료 대기:**
    
    -   작업의 완료를 기다리는 경우 `job.join()`과 같은 메서드를 사용하여 작업이 완료될 때까지 대기할 수 있습니다.
7.  **주의 사항:**
    
    -   `select`는 한 번에 하나의 블록만을 실행합니다.
    -   여러 블록에서 동시에 완료되는 경우 임의로 하나를 선택합니다.

이러한 특성을 통해 `select`는 여러 코루틴이나 채널을 동시에 모니터링하면서 가장 먼저 완료된 작업을 선택하는 강력한 도구로 활용될 수 있습니다.

## 채널에서 값 선택하기
셀렉트 표현식에서 사용하는 주요 함수는 다음과 같다.
- onReceive
	- 채널이 값을 가지고 있을 때 선택된다.
- onReceiveCatching
	- 채널이 값을 가지고 있거나 닫혔을 때 선택된다.
	- 값을 나타내거나 채널이 닫혔다는 걸 알려 주는 ChannelResult 를 받으며, 이 값을 람다식의 인자로 사용한다.
	- onReceiveCatching 이 선택되었을 때, select 는 람다식의 결괏값을 반환한다.
- onSend
	- 채널의 버퍼에 공간이 있을 때 선택된다.
# 18 핫 데이터 소스와 콜드 데이터 소스
- 채널은 값을 핫 스트림으로 가진다.
- 콜드 스트림이 필요할 때가 있다.
## 핫 스트림과 콜드 스트림
- 데이터 동작에 차이가 있다.
	- 핫은 각각의 스트림 연산이 모든 요소 대상으로 처리된 뒤 다음 연산이 진행된다.
	- 콜드는 각각의 원소 하나하나가 모든 스트림을 다 타고 나야 다음 연산이 수행된다.
## 핫 채널, 콜드 플로우
- flow 를 생성하는 가장 일반적인 방법은 produce 함수와 비슷한 형태의 빌더인 flow 를 사용하는 것이다.
- 채널과 플로우의 방식이 아주 다르기 때문에 두 함수에 중요한 차이가 있다.
	- 채널은 핫이라 값을 곧바로 계산한다.
		- 별도의 코루틴에서 계산을 수행한다.
		- 따라서 produce 는 CoroutineScope 의 확장 함수로 정의되어 있는 코루틴 빌더가 되어야 한다.
		- 수신자가 붙든 말든 계산이 진행되며, 단지 버퍼 사이즈에 따라 중단될 뿐이다.
	- 플로우는 콜드 데이터 소스이기 때문에 값이 필요할 때만 생성한다.
		- flow 는 빌더가 아니며 어떤 처리도 하지 않는다.
		- collect 와 같은 최종 연산이 호출될 때 원소가 어떻게 생성되어야 하는지 정의한 것에 불과하다.
		- CoroutineScope 가 필요하지 않다.
		- 이 빌더를 호출한 최종 연산의 스코프에서 실행된다.
		- 플로우의 각 최종 연산은 처음부터 데이터를 처리하기 시작한다.
		- 첫 수신기가 데이터를 처리하였다고 하더라도 두번째 수신기가 처음부터 데이터를 동일하게 받을 수 있게 된다.
# 19 플로우란 무엇인가?
- 플로우는 비동기적으로 계산해야 할 값의 스트림을 나타낸다.
- Flow 인터페이스 자체는 떠다니는 원소들을 모으는 역할을 하며, 플로우의 끝에 도달할 때까지 각 값을 처리하는 걸 의미한다.
```
interface Flow<out T> {
	suspend fun collect(Collector: FlowCollector<T>)
}
```
- collect 는 컬렉션의 forEach 와 비슷하다.

## 플로우와 값들을 나타내는 다른 방법들의 비교
- 플로우는 코루틴을 사용해야 하는 데이터 스트림으로 사용되어야 한다.
## 플로우의 특징
- (collect 와 같은) 프롤우의 최종 연산은 스레드를 블로킹하는 대신 코루틴을 중단시킨다.
- 코루틴 컨텍스트를 활용하고 예외를 처리하는 등의 코루틴 기능도 제공한다.
- 플로우 처리는 취소 가능하며 고조화된 동시성을 기본적으로 갖추고 있다.
- flow 빌더는 중단 함수가 아니며 어떠한 스코프도 필요로 하지 않는다.
- 플로우의 최종 연산은 중단 가능하며, 연산이 실행될 때 부모 코루틴과의 관계가 정립된다.
## 플로우 명명법
- 플로우는 어딘가에서 시작되어야 한다.
	- 플로우 빌더
	- 다른 객체에서의 변환
	- 또는 헬퍼 함수
- 플로우의 마지막 연산은 최종 연산이라 불린다.
- 중단 가능하거나 스코프를 필요로 하는 유일한 연산이라는 점에서 아주 중요하다.
- 시작 연산과 최종 연산 사이에 플로우를 변경하는 중간 연산을 가질 수 있다.

# 20 플로우의 실제 구현
- 코틀린 코루틴의 플로우는 대부분의 개발자들이 생각하는 것보다 간단한 개념이다.
- 내부 구현을 모사한다.

## 플로우 이해하기
- 간단한 람다식에서 시작한다.
```
fun main() {
	val f: () -> Unit = {
		print("A")
		print("B")
		print("C")
	}
}
```
이를 지연이 있는 람다식으로 변경한다.
```
suspend fun main() {
	val f: suspend () -> Unit = {
		print("A")
		delay(1000)
		print("B")
		delay(1000)
		print("C")
	}
}
```
람다식은 함수를 나타내는 파라미터를 가질 수 있습니다. 이 파라미터를 emit 이라고 하자. 람다식 f 를 호출할 때 emit 으로 사용될 또 다른 람다식을 명시해야 한다.
```
suspend fun main() {
	val f: suspend ((String) -> Unit) -> Unit = {
		emit -> 
		emit("A")
		emit("B")
		emit("C")
	}
	f { print(it) }
	f ( print(it) }
}
```
- 이 때 emit 은 중단 함수가 되어야 한다.
- 함수형이 많이 복잡해진 상태이므로, emit 이라는 추상 메서드를 가진 FlowCollector 함수형 인터페이스를 정의해 간단하게 만들어 본다.
```
fun interface FlowCollector {
	suspend fun emit(vlue: String)
}
suspend fun main() {
	val f: suspend (FlowCollector) -> Unit {
		it.emit("A")
		it.emit("B")
		it.emit("C")
	}
	f { print(it) }
	f { print(it) }
}
```

- it 에서 emit 을 호출하는 것이 불편하므로, FlowCollector 를 리시버로 만든다.
```
fun interface FlowCollector {
	suspend fun emit(vlue: String)
}
suspend fun main() {
	val f: suspend FlowCollector.() -> Unit {
		emit("A")
		emit("B")
		emit("C")
	}
	f { print(it) }
	f { print(it) }
}
```
- 람다식을 전달하는 대신에, 인터페이스를 구현한 객체를 만드는 편이 낫다. 
- 이 인터페이스를 Flow 라 하고, 정의는 객체 표현식으로 래핑한다.
```
fun interface FlowCollector {
	suspend fun emit(value: String)
}
interface Flow {
	suspend fun collect(collecotr: FlowCollector)
}
suspend fun main() {
	val builder: suspend FlowCollector.() -> Unit = {
		emit("A")
		emit("B")
		emit("C")
	}
	val flow: Flow = object : Flow {
		override suspend fun collect(
			collector: FlowCollector
		) {
			collector.builder()
		}
	}
	flow.collect { print(it) }
	flow.collect { print(it) }
}
```
- 플로우 생성을 간단하게 하기 위해 flow 빌더를 정의한다.

```
fun flow(
	builder: suspend FlowCollector.() -> Unit
) = object : Flow {
		override suspend fun collect(collector: FlowCollector) {
		collector.builder()
	}
}

suspend fun main() {
	val f: Flow = flow {
		emit("A")
		emit("B")
		emit("C")
	}
	f.collect { print(it) }
	f.collect { print(it) }
}
```
- 마지막으로 제네릭 타입으로 변경한다.
```
fun interface FlowCollector<T> {
	suspend fun emit(value: T)
}
interface Flow<T> {
	suspend fun collect(collector: FlowCollector<T>)
}

fun <T> flow(
	builder: suspend FlowCollector<T>.() -> Unit
) = object : Flow<T> {
	override suspend fun collect(
		collector: FlowCollector<T>
	) {
		collector.builder()
	}
}
```

## 동기로 작동하는 Flow
- 플로우 또한 중단 함수처럼 동기로 작동하기 때문에, 플로우가 완료될 때까지 collect 호출이 중단된다.
- 플로우는 새로운 코루틴을 시작하지 않는다.

## 플로우와 공유 상태
- 플로우 처리를 통해 좀더 복잡한 알고리즘을 구현할 때는 언제 변수에 대한 접근을 동기화해야 하는지 알아야 한다.

# 21 플로우 만들기

- 플로우는 어디선가 시작되어야 한다.
## 원시값을 가지는 플로우
- flowOf 함수를 사용하는 방법
```
suspend fun main() {
	flowOf(1, 2, 3, 4, 5).collect { print(it) } //12345
}
```
- 값이 없는 플로우
- emptyFlow() 를 사용한다.
```
emptyFlow<Int>().collect { print(it) } //(아무것도 출력되지 않음)
```
## 컨버터
- asFlow 함수를 사용하여, Iterable, Iterator, Sequence 를 Flow 로 바꿀 수도 있다.
- 즉시 사용 가능한 원소들의 플로우를 만든다.
```
suspend fun main() {
	listOf(1,2,3,4,5) // setOf or sequenceOf 도 가능
	.asFlow()
	.collect { print(it) } //12345
}
```
## 함수를 플로우로 바꾸기
- 플로우는 시간상 지연되는 하나의 값을 나타낼 때 자주 사용된다.
- 따라서 중단 함수를 플로우로 변환하는 것 또한 가능하다.
- 이 때 중단 함수의 결과가 플로우의 유일한 값이 된다.
- (suspend() -> T) 혹은 (() -> T) 같은 함수의 확장 함수인 asFlow 를 사용할 수 있다.
```
suspend fun getUserName(): String {
	delay(1000)
	return "UserName"
}
suspend fun main() {
	val function = suspend {
		delay(1000)
		"UserName"
	}
	function.asFlow().collect { println(it) }
	::getUserName
	.asFlow().collect { println(it) }
}

```
## 플로우와 리액티브 스트림
- 애플리케이션에서 Reactive 스트림을 사용하고 있다면 코드를 별로 바꾸지 않고 플로우를 적용할 수 있다.
- Flux, Flowable, Observable 은 kotlinx-coroutines-reactive 라이브러리의 asFlow 함수를 사용해 Flow 로 변환 가능한 Publisher 인터페이스를 구현한다.
```
suspend fun main() = coroutineScope {
	Flux.range(1, 5).asFlow().collect { print(it) } //12345
	Flowable.range(1, 5).asFlow().collect { print(it) }//12345
	Observable.range(1, 5).asFlow().collect { print(it) }//12345
}
```
- 역으로 Flow 를 반대로 바꿀 수도 있다.
	-	flow.asFlux()
	-	flow.asflowable()
	-	flow.asObservable
-	kotlinx-coroutines-rx3(kotlinx-coroutines-rx2) 라이브러리가 필요하다.
## 플로우 빌더
- 가장 많이 사용되는 방법
- flow 필더를 사용하는 것으로 sequence, produce 빌더와 비슷하게 작동한다.
- Channel 이나 Flow 에서 모든 값을 방출하려면 emitAll 을 사용할 수 있다.
```
flow {
	repeat(3) {
		num -> delay(1000); emit(num)
	}
}
```

## 플로우 빌더 이해하기
- 플로우 빌더는 플로우를 만드는 가장 기본적인 방법이다.
```
public fun <T> flowOf(vararg elements: T): Flow<T> = flow {
	for (element in elements) {
		emit(element)
	}
}
```
이는 이전 장에서 살펴본 내용과 같은 원리이다.
## 채널 플로우
- Flow 는 콜드 데이터 스트림이다.
- 따라서 요청을 한 경우에 한해 데이터를 제공한다.
- 그러나 원소를 처리하고 있을 때 미리 다음 데이터를 받아오고 싶을 때가 있다.
- 이는 핫 데이터 스트림의 특징이다.
	- 채널과 플로우를 합친 형태인 channelFlow 함수를 이용할 수 있다.
- 이는 채널이지만 Flow 인터페이스를 구현하므로 플로우가 가지는 특징을 제공한다.
- 채널플로우 빌더는 일반 함수이며 최종 연산으로 시작된다.
- 한 번 시작되기만 하면 리시버를 기다릴 필요 없이 분리된 코루틴에서 값을 생성한다는 점이 채널과 비슷하다.
```
channelFlow {
	send(3)
	send(4)
}
```
- channelFlow 는 ProducerScope 에서 작동한다.
- ProducerScope 는 produce 빌더가 사용하는 것과 같은 타입이다.
- CoroutineScope 를 구현했기 때문에 빌더에서 새로운 코루틴을 시작할 때 사용할 수 있다.
```
interface ProducerScope<in E>: CoroutineScope, SendChannel<E> {
	val channel: SendChannel<E>
}
```
- send 로 데이터를 전송한다.
- 내부 sendChannel 에 접근해서 직접 메서드를 다룰 수도 있다.
- 내부에서 launch 같은 코루틴 빌더를 직접 시작할 수 있다.
	- 이는 flow 는 하지 못하는 기능이다(코루틴 빌더가 필요로 하는 스코프를 만들지 못하기 때문).?
```
fun <T> Flow<T>.merge(other: Flow<T>): Flow<T> = 
	channelFlow {
		launch(Dispatchers.IO) {
			collect { send(it) }
		}
		other.collect { send(it) }
	}

```
## 콜백플로우
- 이벤트 반응형 플로우가 필요할 경우 사용할 수 있다.
- callbackFlow 를 이용하는 방법이다.
- ProducerScope<T> 에서 작동한다.
- 이전에는 channelFlow 와 크게 다른 점이 없었다.
- 유용한 몇가지 함수를 제공한다.
	- awaitClose { ... }
		- 채널이 닫힐 때까지 중단되는 함수이다.
		- 채널이 닫힌 다음에 인자로 들어온 함수가 실행된다.
		- 이 호출이 없으면 콜백 등록 후 코루틴은 곧바로 끝나게 된다.
			- callbackFlow 빌더의 실행이 끝나고 기다릴 자식 코루틴이 없으면 코루틴은 종료되기 때문이다.
	- trySendBlocking(value)
		- send 와 비슷하지만 중단하는 대신 블로킹하여 중단 함수가 아닌 함수에서도 사용할 수 있다.
	- close()
		- 채널을 닫는다
	- cancel(throwable)
		- 채널을 종료하고 플로우에 예외를 던진다

```
fun flowFrom(api: CallbackBasedApi): Flow<T> = callbackFlow {
	val callback = object: Callback {
		override fun onNextValue(value: T) {
			trySendBlocking(value)
		}
		override fun onApiError(cause: Throwable) {
			cancel(CancellatitonException("API ERROR", cause))
		}
		override fun onCompleted() = channel.close()
	}
	api.register(callback)
	awaitClose { api.unregister(callback) }
}
```
# 22 플로우 생성주기 함수
- 플로우는 요청이 한쪽 방향으로 흐르고 요청에 의해 성성된 값이 다른 방향으로 흐르는 파이프다.
- 모든 정보가 플로우를 통해 전달되므로, 시작 완료와 같은 다른 특정 이벤트를 감지할 수 있다.
- onEach, onStart, onCompletion, onEmpty 그리고 catch 같은 메서드를 사용하면 된다.
## onEach
-	플로우의 값을 하나씩 받기 위해 사용한다.
```
suspend fun main() {
	flowOf(1,2,3)
	.onEach { print(it) }
	.collect() //123
}
```
- 중단함수다.
- 원소는 순서대로 처리된다.
- 이곳에 delay 를 넣으면 각각의 값이 흐를 때마다 지연이 발생한다.
## onStart
- 최종 연산이 호출될 때와 같이 플로우가 시작되는 경우에 호출되는 리스너를 설정하는 함수이다.
- 첫번째 원소가 생성되는 걸 기다렸다가 호출되는 게 아니다.
- 첫 번째 원소를 요청했을 때 호출된다.
```
suspend fun main() {
	flowOf(1, 2)
	.onEach { delay(1000) }
	.onStart { println("before"); emit(0) }
	.collect { println(it) } 
	//Before
	//0
	//1 second later
	//1
	//1 second later
	//2
}
```
- 이 내부에서도 원소를 내보낼 수 있다.
	- onCompletion, onEmpty, catch 도 동일하다.
	- 원소들은 onStart 부터 아래로 흐른다.
## onCompletion
- 플로우를 완료할 수 있는 여러 방법이 있다.
	- 잡히지 않은 예외가 발생하는 경우
	- 코루틴이 취소되는 경우
	- 플로우 빌더가 끝나는 경우
		- 마지막 원소가 전송되었을 때
- 이 순간에 즉 플로우가 완료되었을 때 호출되는 리스너를 추가할 수 있다.
```
flowOf(1,2)
	.onEach { delay(1000) }
	.onCompletion { println("Completed") }
	.collect { println(it) }
// 1 sec later
//1
// 1 sec later
//2
// Completed
```
## onEmpty
- 플로우는 예기치 않은 이벤트가 발생하면 값을 내보니기 전에 완료될 수 있다.
- 이 함수는 원소를 내보내기 전에 플로우가 안료되면 실행된다.
- 기본값을 내보니기 위한 목적으로 사용될 수 있다.
```
flow<List<Int>> { delay(1000) }
.onEmpty { empt(emptyList()) }
.collect { println(it) }
//1 sec later
//[]
```
## catch
- 플로우를 만들거나 처리하는 도중에 예외가 발생할 수 있다.
- 예외는 아래로 흐르면서 처리하는 단계를 하나씩 닫는다.
- 예외를 잡고 관리할 수도 있다.
- catch 가 이 용도로 이용된다.
	- 리스너는 예외를 인자로 받고 정리를 위한 연산을 수행할 수 있다.
```
val flow = flow {
	emit(1)
	emit(2)
	throw Error()
}

suspend fun main(): Unit {
	flow.onEach { println("Got $it") }
		.catch { println("Caught $it") }
		.collect { println("Collected $it") }
}
//Got 1
//Collected 1
//Got 2
//Colelcted 2
//Caught Error
```
- onEach 는 예외에 반응하지 않는다.
- catch 는 예외를 잡아 전파되는 걸 멈춘다.
- 이전 단계는 이미 완료된 상태지만, catch 는 새로운 값을 여전히 내보낼 수 있어 남은 플로우를 지속할 수 있다.
- catch 윗부분에서 던진 예외에만 반응한다.
## 잡히지 않은 예외
- 플로우에서 잡히지 않은 예외는 플로우를 즉시 취소하며, collect 는 예외를 다시 던진다.
- 중단 함수가 예외를 처리하는 방식과 같다.
- 플로우 밖에서 전통적인 try-catch 를 사용해서 예외를 잡을 수도 있다.
- catch 는 최종 연산에서 발생한 예외를 처리하는 데 도움이 되지 않는다.
- 따라서 collect 의 로직을 onEach 로 옮기고 catch 를 그 뒤로 이동시키는 방법을 주로 사용하기도 한다.
## flowOn
- (onEach, onStart, onCompletion 과 같은) 플로우 연산과 (flow or channelFlow 와 같은) 플로우 빌더의 인자로 사용되는 람다식은 모두 중단 함수다.
- 중단 함수이므로 컨텍스트가 필요하다.
- 이 컨텍스트는 어디서 얻어오는가 하면 collect 가 호출되는 곳의 컨텍스트이다.
- flowOn 을 사용해서 컨텍스트를 변경할 수도 있다.
- 윗부분에 있는 함수에서만 작동한다.
## launchIn
-	유일한 인자로 스코프를 받는다. collect 를 새로운 코루틴에서 시작할 수 있다
```
fun <T> Flow<T>.launchIn(scope: CoroutineScope): Job = scope.launch { collect() }
suspend fun main(): Unit = coroutineScope {
	flowOf("User1", "User2")
		.onStart { println("Users:") }
		.onEach { println(it) }
		.launchIn(this)
}
```
# 23 플로우 처리
- 플로우를 값이 흐르는 파이프로 바라보았다. 여러가지 방식으로 따라서 값을 변경해 보는 것이 가능하다.

## map
- 이는 functor 로 stream 의 map 과 같은 fmap 형태의 함자이다.
## filter
- 이는 요소 분별기로 stream 의 filter 와 같은 predicate 이다.
## take, drop
- 이는 함수형에서 특정 요소의 갯수를 챙기기 위한 take 함수와 기능이 같다.
- take 는 취득하며 drop 은 무시한다.
	- drop 은 stream 의 skip 과 유사하다.
## merge, zip, combine
- 모두 2개의 플로우를 하나의 플로우로 합치는 연산이다.
- merge
	- 한 플로우의 원소가 다른 풀로우를 기다리지 않는다.
- zip
	- 한 플로우의 원소는 자신과 짝이 될 원소가 다른 플로우에서 나타날 때까지 기다려야 한다.
	- 쌍을 이루지 못하고 남은 원소는 유실되고 zip 으로 생성된 플로우는 종료된다.
- combine
	- zip 과 유사하지만 모든 새로운 원소가 전임자를 대체하게 된다.
	- 유실되는 건은 없으며 잔여 원소는 다른 플로우의 마지막 원소와 함께 쌍이 된다.
## fold, scan
- 컬렉션 처리 함수를 사용할 때 fold 를 보았을 것이다. 같은 연산이다.
- scan 은 fold 의 연산 중간중간의 누적되는 과정의 모든 값을 생성하는 중간 연산자이다. 생성은 보통 리스트로 리턴된다.
## flatMapConcat, flatMapMerge, flatMapLatest
- flatMap 은 bind 함수 형태의 모나드이다.
- 플로우에서는 변환 함수가 평탄화된 플로우를 반환한다고 생각하는 것이 직관적이다.
- 문제는 플로우 원소가 나오는 시간이 다르다는 것이다.
- 두번쩨 원소에서 만들어진 플로우가 첫번째 플로우에서 만들어진 원소를 기다릴지 동시에 처리할지 등에 기준이 모호해진다.
- 따라서 위 3가지 동작 형태에 따른 함수가 제공된다.
- flatMapConcat
	- 생성한 플로우를 하나씩 처리한다.
	- 두번째 플로우는 첫번째 플로우가 완료되었을 때 싲가할 수 있다.
```
fun flowFrom(elem: String) = flowOf(1,2,3)
.onEach { delay(1000) }
.map { "${it}_${elem}" }

suspend fun main() {
	flowOf("A", "B", "C")
	.flatMapConcat { flowFrom(it) }
	.collect { println(it) }
}
//1 sec later
//1_A, //1 sec later
//2_A, //1 sec later
//3_A, //1 sec later
//1_B, //1 sec later
//2_B, //1 sec later
//3_B, //1 sec later
//1_C, //1 sec later
//2_C, //1 sec later
//3_C
```
- flatMapMerge
	-	만들어진 플로우를 동시에 처리한다.
```
fun flowFrom(elem: String) = flowOf(1,2,3)
.onEach { delay(1000) }
.map { "${it}_${elem}" }

suspend fun main() {
	flowOf("A", "B", "C")
	.flatMapMerge { flowFrom(it) }
	.collect { println(it) }
}
//1 sec later
//1_A, //1 sec later
//1_B, //1 sec later
//1_C, //1 sec later
//2_A, //1 sec later
//2_B, //1 sec later
//2_C, //1 sec later
//3_A, //1 sec later
//3_B, //1 sec later
//3_C
```
- concurrency 인자를 사용해 동시에 처리할 수 있는 플로우의 수를 설정할 수 있다.
- 기본값은 16이나 JVM 에서 DEFAULT_CONCURRENCY_PROPERTY_NAME 프로퍼티를 사용해 변경 가능하다.
- flatMapLatest
	- 새로운 플로우가 나타나면 이전에 처리하던 플로우를 잊는다.
	- 새로운 값이 나올 때마다 이전 플로우 처리는 사라진다.
	- "A", "B", "C" 사이에 지연이 없다면 "1_C", "2_C", "3_C" 만 보게 된다.
```
fun flowFrom(elem: String) = flowOf(1,2,3)
.onEach { delay(1000) }
.map { "${it}_${elem}" }

suspend fun main() {
	flowOf("A", "B", "C")
	.flatMapLatest { flowFrom(it) }
	.collect { println(it) }
}
//1 sec later
//1_C, //1 sec later
//2_C, //1 sec later
//3_C
```
- 시작 플로우를 생성할 때 지연이 발생하면 재밌는 상황이 발생한다.
```
fun flowFrom(elem: String) = flowOf(1,2,3)
.onEach { delay(1000) }
.map { "${it}_${elem}" }

suspend fun main() {
	flowOf("A", "B", "C")
	.onEach { delay(1200) }
	.flatMapLatest { flowFrom(it) }
	.collect { println(it) }
}
//2.2 sec later
//1_A, //1.2 sec later
//1_B, //1.2 sec later
//1_C, //1 sec later
//2_C, //1 sec later
//3_C
```
## 재시도(retry)
- 예외는 플로우를 딸 ㅏ흐르면서 각 단계를 하나씩 종료한다.
- 종료된 단계는 비활성화되기 때문에, 예외가 발생한 뒤 메시지를 보내는 건 불가능하지만 각 단계가 이전 단계에 대한 참조를 가지고 있다.
- 이 참조는 플로우를 다시 시작하기 위해 사용될 수 있다.
	- 이 원리를 기반으로, 코틀린은 retry, retryWhen 함수를 제공한다.
- retryWhen
	- 이전 단계에서 예외가 발생할 때마자 조건자(predicate)를 확인한다.
	- 참을 리턴하면 재시도 한다.
	- 중단 함수이므로 재시도 간 대기를 위해 delay 를 호출할 수도 있다.
## 중복 제거 함수
- distinctUntilChanged 는 람다 표현식을 받아 두 원소가 비교되는 방법을 정의한다.
- distinctUntilChangedBy 는 두 원소가 동일한지 판단하기 위해 비교할 키 선택자를 인자로 받는다.
## 최종 연산
- 대표적으로 collect 가 있었다.
- count
	- 원소의 수를 센다
- first
	- 가장 첫번째 원소만 얻는다
- firstOrNull
- fold
- reduce
	- 원소들을 누적하여 하나의 객체를 만든다.

# 24. 공유플로우와 상태플로우
- 일반적으로 플로우는 콜드 데이터이기 때문에 요청할 때마다 값이 계산된다.
- 여러 개의 수신자가 하나의 데이터가 변경되는지 감지하는 경우도 있다. 이 때 공유플로우, 상태플로우를 사용한다.
- 이들은 감지 가능한 값들을 다룬다.
## 공유플로우
- 브로드캐스트 채널과 비슷한 MutableSharedFlow 를 먼저 본다.
	- 공유플로우에 메시지를 보내면 대기하고 있는 모든 코루틴이 수신한다.
	- replay 인자를 설정하면 마지막으로 전송한 값들이 정해진 수만큼 저장된다.
		- 코루틴이 감지를 시작하면 저장된 값들을 먼저 받게 된다.
	- resetReplayCache 를 사용하면 값을 저장한 캐시를 초기화한다.
- 코틀린에서는 감지만 하는 인터페이스와 변경하는 인터페이스를 구분하는 것이 관행이다.
	- 앞에서 SendChannel, ReceiveChannel, Channel 로 구분하는 것이 그 예다.
	- MutableSharedFlow 는 SharedFlow, FlowCollector 모두를 상속한다.
	- SharedFlow 는 Flow 를 상속하고 감지하는 목적으로 사용되고 후자는 값을 내보내는 목적으로 사용된다.
	- 이 둘은 값을 내보내거나 또는 수집하는 함수만 노출하기 위해 자주 사용된다.
## shareIn
- 여러 가지 플로우를 하나로 합치는 방법이 있었다.
- Flow 를 SharedFlow 로 바꾸는 가장 쉬운 방법이 shareIn 함수를  사용하는 것이다.
- 첫번째 인자는 코루틴 스코프다.
- 세번째 인자는 replay 며 기본값이 0 이다.
- 두 번째 인자인 started 는 다음과 같다.
```
val sharedFlow: SharedFlow<String> = flow.shareIn(
	scope = //scope
	started = SharingStarted.Eagerly,
	// replay = 0 (default)
)
```
	- ShareingStarted.Eagerly
		- 값을 즉시 감지하기 시작하고 플로우로 값을 전송한다.
		- replay 값에 제한이 있고 감지를 시작하기 전에 값이 나오면 일부를 유실할 수 있다.
	- SharingStarted.Lazily
		- 첫 번째 구독자가 나올 때 감지하기 시작한다.
		- 첫 번째 구독자는 내보내진 모든 값을 수신하는 것이 보장된다.
		- 이후의 구독자는 replay 수만큼 가장 최근에 저장된 값들을 받게 된다.
		- 모든 구독자가 사라져도 업스트림(데이터를 방출하는) 플로우는 액티브 상태지만, 구독자가 없으면 replay 수만큼 가장 최근의 값들만 캐싱한다.
	- WhileSubscribed()
		- 첫 번째 구독자가 나올 때 감지하기 시작하며, 마지막 구독자가 사라지면 플로우도 멈춘다.
		- 새로운 구독자가 나오면 플로우가 다시 시작된다.
		- stopTimeoutMillis(기본값 0)이 있고 마지막 구독자가 사라지고 난 뒤 감지할 시간을 나타낸다.
		- replayExpirationMillis (기본값 Long.MAX_VAULE) 멈춘 뒤 리플레이 값을 가지는 시간을 나타낸다.
	- SharingStarted
		- 인터페이스를 구현하여 커스텀화된 전략을 정의하는 것도 가능하다.

## 상태플로우
- 공유플로우의 개념을 확장시킨 것으로, replay 인자 값이 1인 공유플로우와 비슷하게 작동한다.
- value 프로퍼티로 접근 가능한 값 하나를 가지고 있다.
- 초깃값은 생성자를 통해 전달되어야 한다.
- value 프로퍼티로 값을 얻어올 수도 있고 설정할 수도 있다.
- MutableStateFlow 는 값을 감지할 수 있는 보관소이다.
- 상태플로우는 데이터가 덮어 씌워지기 때문에, 관찰이 느린 경우 상태의 중간 변화를 받을 수 없는 경우도 있다.
## stateIn
-	Flow<T> 를 StateFlow<T> 로 변환하는 함수이다.
-	스코프에서만 호출 가능하지만 중단 함수이기 도 하다.
-	항상 값을 가져야만 하므로, 값을 명시하지 않았을 때는 첫 번째 값이 계산될 때까지 기다려야 한다.
```
suspend fun main() = coroutineScope {
	val flow = flowOf("A", "B", "C")
	.onEach { delay(1000) }
	.onEach { println("produced $it") }
	val stateFlow: StateFlow<String> = flow.stateIn(this)
}
```
- stateIn 의 두 번째 형태는 중단 함수가 아니지만 초기 값과 started 모드를 지정해야 한다.
	- 모드는 shareIn 과 같은 옵션을 가진다.
