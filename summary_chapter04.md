# 4.1 카프카 컨슈머: 개념

## 4.1.1. 컨슈머와 컨슈머 그룹

### 프로듀서가 애플리케이션이 검사할 수 있는 속도보다 더 빠른 속도로 토픽에 메시지를 쓰게 된다면? → 컨슈머 그룹에 컨슈머 추가 필요

<img width="721" height="166" alt="image" src="https://github.com/user-attachments/assets/eebb7243-75d2-49d9-9379-4f5d3f673646" />


### 파티션 개수보다 컨슈머 그룹에 속한 컨슈머가 더 많을 때는 **유휴 컨슈머**가 발생

<img width="385" height="334" alt="image" src="https://github.com/user-attachments/assets/882febe1-071c-48e6-a454-34c36243e016" />


### 여러 애플리케이션이 동일한 토픽에서 데이터를 읽어와야 하는 경우 → 애플리케이션이 각자의 컨슈머 그룹을 갖도록 설정

<img width="377" height="445" alt="image" src="https://github.com/user-attachments/assets/435470c4-9e6f-42bf-8f7b-9b86b42ac88b" />


## 4.1.2 컨슈머 그룹과 파티션 리밸런스

### 리밸런스

`리밸런스`: 컨슈머에 할당된 파티션을 다른 컨슈머에게 할당해주는 작업이며 컨슈머 그룹에 **높은 가용성과 규모 가변성**을 제공하는 기능

- 운영자가 토픽에 새 파티션을 추가하는 경우
- 새로운 컨슈머를 컨슈머 그룹에 추가하는 경우
- 컨슈머가 종료되거나 크래시 나는 경우

### 리밸런스의 종류

1. **조급한 리밸런스 - 버전2.4이후 기본값, 삭제 예정**

모든 컨슈머가 자신에게 할당된 파티션을 포기하고, 파티션을 포기한 컨슈머 모두가 다시 그룹에 참여한 뒤에야 새로운 파티션을 할당받고 읽기 작업을 재개 → **작업 중단 발생**

<img width="377" height="445" alt="image" src="https://github.com/user-attachments/assets/ac0e628d-54fb-4379-a516-0baa20228505" />


1. **협력적 리밸런스(점진적 리밸런스) - 버전 3.1부터 기본값**

컨슈머 그룹 리더가 다른 컨슈머들에게 각자에게 할당된 파티션 중 일부가 재할당될 것이라고 통보하면 컨슈머들은 해당 파티션에서 데이터를 읽어오는 작업을 멈추고 해당 파티션에 대한 소유권 포기

→ 컨슈머 그룹 리더가 이 포기된 파티션들을 재할당

<img width="663" height="350" alt="image" src="https://github.com/user-attachments/assets/868d5754-b59f-41ab-8898-be7dd148a86e" />


- 안정적으로 파티션이 할당될 때까지 시간 필요
- 리밸런싱 작업에 상당한 시간이 걸릴 위험이 있는, 컨슈머 그룹에 속한 컨슈머 수가 많은 경우에 특히 중요

### 리밸런스 동작 흐름

1. 컨슈머 그룹이 시작되면 → 컨슈머들은 그룹 코디네이터에게 가입 요청
2. 그룹 코디네이터 → 파티션을 분배
3. 컨슈머들은 백그라운드에서 하트비트를 주기적으로 보냄
4. 만약 어떤 컨슈머가:
    - **정상 종료**: "나 나갈게요"라고 **`코디네이터`에게 알림 → 즉시 리밸런스
    - **비정상 종료(크래시)**: **`하트비트`가 끊김 → 세션 타임아웃 후 코디네이터가 죽었다고 판단 → 리밸런스 트리거

** `그룹 코디네이터`: 컨슈머 그룹의 “관리자 역할”을 하는 카프카 브로커

** `하트비트`: 컨슈머의 백그라운드 스레드가 자동으로 그룹 코디네이터에게 정기적으로 보내는 “살아있다”는 메시지

## 4.1.3 정적 그룹 멤버십

- `group.instance.id`를 지정하면 컨슈머 재시작 시에도 기존 멤버십과 파티션 할당을 유지(`session.timeout.ms`까지 그룹 멤버 자격 유지) → 불필요한 리밸런스 방지
- 같은 ID 중복 사용 시 에러 발생 (ID는 고유해야 함)
- 중단된 동안 쌓인 메시지를 재시작 후 이어서 처리 → 밀린 메시지들을 따라잡을 수 있는지 확인 필요
- `session.timeout.ms`는 리밸런스를 최소화하면서도 장시간 파티션 정지를 막도록 적절히 설정 필요

# 4.2 카프카 컨슈머 생성하기

## 기본으로 정의해야 하는 속성

- bootstrap.servers
- key.deserializer
- value.deserializer
- group.id

# 4.3 토픽 구독하기

## 토픽 구독에 **정규식** 표현 사용 가능

```java
consuer.subscribe(Pattern.compile("test.*"));
```

- 토픽의 목록이 매우 크고 컨슈머도 많으며 토픽과 파티션의 목록 크기 역시 상당할 경우, 정규식을 사용한 구독은 브로커, 클라이언트, 네트워크 전체에 걸쳐 상당한 오버헤드
- 정규식을 사용해서 토픽을 구독할 경우, 클라이언트는 클러스터 안에 있는 모든 토픽에 대한 상세 정보를 조회할 권한이 있어야 함(전체 클러스터에 대한 완전한 Describe 작업 권한 부여를 의미)

# 4.4 폴링 루프

```java
Duration timeout = Duration.ofMillis(100);

while (true) { 
	ConsumerRecords<String, String> records = consumer.poll(timeout); // 1

	for (ConsumerRecord<String, String> record : records) { // 2
		System.out.printf("topic = %s, partition = %d, offset = %d, " + 
			"customer = %s, country = %s\n",
		record.topic(), record.partition(), record.offset(),
				record.key(), record.value());
		int updatedCount = 1;
		if (custCountryMap.containsKey(record.value())) {
			updatedCount = custCountryMap.get(record.value()) + 1;
		}
		custCountryMap.put(record.value(), updatedCount);

		JSONObject json = new JSONObject(custCountryMap);
		System.out.println(json.toString()); // 3
	}
}
```

1. `poll()`에 전달하는 매개변수는 컨슈머 버퍼에 데이터가 없을 경우 `poll()`이 블록될 수 있는 최대 시간을 결정
    
    1-1. `poll()` 이 `max.poll.interval.ms`에 저장된 시간 이상으로 호출되지 않을 경우, 컨슈머는 죽은 것으로 판정, 컨슈머 그룹에서 퇴출
    
2. `poll()`은 레코드들이 저장된 List 객체를 리턴: 토픽, 파티션, 파티션에서의 오프셋 그리고 키와 밸류 값 포함
3. 결과물을 데이터 저장소에 쓰거나 이미 저장된 레코드를 갱신

## 4.4.1 스레드 안전성

**하나의 스레드당 하나의 컨슈머**가 원칙

- 하나의 애플리케이션에서 동일한 그룹에 속하는 여러 개의 컨슈머를 운용하고 싶다면(병렬 처리):
    1. **컨슈머 여러 개(=스레드 수만큼) + 같은 그룹** : `ExecutorService` 사용 → 파티션을 나눠 맡아 각자 poll/처리
        - 파티션 수가 충분하고, 메시지 처리 시간이 비교적 짧거나 균일할 때
        - 구성/오프셋 관리가 가장 단순
    2. **컨슈머 1개 + 내부 작업 큐 + 워커 스레드 여러 개** → 폴링만 전용 스레드가 하고, 처리만 병렬화
        - 파티션 수가 적은데 처리 로직이 **CPU/GPU/IO 집약**이라 병렬 처리가 필요할 때
        - 리밸런스/세션 관리를 단순화하고 싶을 때
        - 단, **오프셋 커밋을 컨슈머 스레드에서** 안전하게 처리하는 설계를 꼭 넣어야 함

> 백엔드 쪽: 직접 ExecutorService / Thread / queue 같은 방식 많이 씀 
→ 이유: 이벤트 처리 로직이 서비스 도메인에 밀접해서

데이터 엔지니어 쪽: 직접 스레드 짜기보다는 Spark Structured Streaming, 
Flink, Kafka Streams 같은 프레임워크 사용 → 내부적으로는 파티션 수에 따라 병렬 처리.
> 

# 4.5 컨슈머 설정하기

## 4.5.1 fetch.min.bytes

컨슈머가 브로커로부터 레코드를 얻어올 때 받는 데이터의 최소량(바이트)를 지정(**기본값: 1바이트**)

- 새로 보낼 레코드 양이 `fetch.min.bytes`보다 작을 경우, 브로커는 충분한 메시지를 보낼 수 있을 때까지 기다린 뒤 컨슈머에게 레코드 전송

## 4.5.2 fetch.max.wait.ms

얼마나 오래 기다릴 것인지를 결정(**기본값: 500밀리초**)

→ 카프카는 컨슈머로부터의 읽기 요청을 받았을 때 `fetch.min.bytes` 또는 `fetch.max.wait.ms` 두 조건 중 하나가 만족되는 대로 레코드를 리턴

## 4.5.3 fetch.max.bytes

카프카가 리턴하는 최대 바이트 수를 지정(**기본값: 50MB**)

- 브로커가 보내야 하는 **첫 번째 레코드 배치의 크기가 이 설정값을 넘은 경우, 제한값을 무시하고 해당 배치를 그대로 전송** → 컨슈머가 읽기 작업을 계속하도록 보장
- 브로커 설정에도 최대 읽기 크기를 제한할 수 있게 해주는 설정 있음

## 4.5.4 max.poll.records

`poll()`을 호출할 때마다 리턴되는 **최대 레코드 수**

## 4.5.5 max.partition.fetch.bytes

서버가 파티션별로 리턴하는 최대 바이트 수를 결정(기본값: 1MB)

- 이 설정을 사용하여 메모리 사용량 조절하는 것은 꽤 복잡 → `fetch.max.bytes`설정 이용 권장

## 4.5.6 session.timeout.ms 그리고 heartbeat.interval.ms

`hearbeat.interval.ms` : 카프카 컨슈머가 얼마나 자주 그룹 코디네이터에게 하트비트를 보내는지 결정(**기본값: 3초**)

`session.timeout.ms`: 컨슈머가 하트비트를 보내지 않을 수 있는 최대 시간을 결정(**기본값: 10초 → 45초**)

- 두 속성은 서로 밀접하게 연관되어 있기 때문에 대개 함께 변경
- `heartbeat.interval.ms`는 `session.timeout.ms`보다 더 낮은 값이어야 하며 대치로 **1/3**으로 결정하는 것이 보통

## 4.5.7 max.poll.interval.ms

컨슈머가 **poll()을 호출하지 않고도 살아있다고 간주되는 최대 시간**(**기본값: 5분**)

- 하트비트만 정상이어도, poll()이 지연되면 시간 초과 시 그룹에서 제외됨
- 처리 시간이 긴 경우 → `max.poll.records`와 함께 조정 필요
- 권장: 정상 컨슈머는 거의 도달하지 않을 만큼 크게, 하지만 멈춘 컨슈머는 빨리 감지할 수 있을 만큼 적절히 설정

## 4.5.8 default.api.timeout.ms

API를 호출할 때 명시적인 타임아웃을 지정하지 않는 한, 거의 모든 컨슈머 API 호출에 적용되는 타임아웃 값(**기본값: 60초**)

- 적용 예외: `poll()` 메서드 - **호출자가 직접 타임아웃을 정함**

## 4.5.9 request.timeout.ms

컨슈머가 브로커로부터의 응답을 기다릴 수 있는 최대 시간(**기본값: 30초**)

- 30초 이하로 설정하지 않는 것을 권장

## 4.5.10 auto.offset.reset

컨슈머가 예전에 오프셋을 커밋한 적이 없건, 커밋된 오프셋이 유효하지 않을때(대개 컨슈머가 오랫동안 읽은 적이 없어서 오프셋의 레코드가 이미 브로커에서 삭제된 경우), 파티션을 읽기 시작할 때의 작동을 정의

- `latest`(기본값): 유효한 오프셋이 없는 경우 가장 최신 레코드부터 읽기 시작
- `earliest`: 유효한 오프셋이 없을 경우 파티션의 맨 처음부터 모든 데이터를 읽는 방식

## 4.5.11 enable.auto.commit

컨슈머가 자동으로 오프셋을 커밋할지의 여부를 결정(**기본값: true**)

- `auto.commit.interval.ms` 를 사용해서 얼마나 자주 오프셋이 커밋될지 제어 가능

## 4.5.12 partition.assignment.strategy

`PartitionAssignor` 클래스는 컨슈머와 이들이 구독한 토픽들이 주어졌을 때 어느 컨슈머에게 어느 파티션이 할당될지를 결정하는 역할

**토픽 A(6개 파티션)**, **컨슈머 그룹에 컨슈머 3명(C1, C2, C3)** 있는 상황을 가정한 예시

### 1. Range

`org.apache.kafka.clients.consumer.RangeAssignor`

컨슈머가 구독하는 각 토픽의 파티션들을 **연속된 그룹으로 나눠서 할당**

- 각 토픽의 할당이 독립적으로 이루어지기 때문에 첫 번째 컨슈머는 두 번쨰 컨슈머보다 더 많은 파티션을 할당받게 됨

| Consumer | 할당 파티션 |
| --- | --- |
| C1 | 0, 1 |
| C2 | 2, 3 |
| C3 | 4, 5 |

### 2. RoundRobin

`org.apache.kafka.clients.consumer.RoundRobinAssignor`

모든 구독된 토픽의 모든 파티션을 가져다 순차적으로 하나씩 컨슈머에 할당

- 모든 컨슈머들이 완**전히 동일한 수(혹은 많아야 1개 차이)**의 파티션을 할당 받게 됨

| Consumer | 할당 파티션 |
| --- | --- |
| C1 | 0, 3 |
| C2 | 1, 4 |
| C3 | 2, 5 |

### 3. Sticky

`org.apache.kafka.clients.consumer.StickyAssignor`

**목표:**

- 파티션들을 가능한 한 균등하게 할당
- 리밸런스가 발생했을 때 가능하면 많은 파티션들이 같은 컨슈머에 할당되게 함으로써 **할당된 파티션을 하나의 컨슈머에서 다른 컨슈머로 옮길 때 발생하는 오버헤드를 최소화**

→ 할당된 결과물의 균형 정도는 RoundRobin 할당자와 유사하지만 **이동하는 파티션의 수 측면에서는 Sticky 쪽이 더 작다.** 

→ 같은 그룹에 속한 컨슈머들이 서로 다른 토픽을 구독할 경우, Sticky 할당자를 사용한 할당이 RoundRobin 방식보다 더 균형잡히게 된다. 

| Consumer | 할당 파티션 |
| --- | --- |
| C1 | 0, 3 |
| C2 | 1, 4 |
| C3 | 2, 5 |

### 4. Cooperative Sticky

`org.apache.kafka.clients.consumer.CooperativeStickyAssignor`

Sticky 할당자와 기본적으로 동일하지만, 컨슈머가 재할당되지 않는 파티션으로부터 레코드를 계속해서 읽어 올 수 잇도록 해주는 협력적 리밸런스 기능 지원 - 협력적 리밸런스 (p.88) 참고

## 4.5.13 client.id

어떠한 문자열도 될 수 있으며, 브로커가 요청을 보낸 클라이언트를 식별하는 데 사용

- 로깅, 모니터링 지표, 쿼터에서도 사용

## 4.5.14 client.rack

클러스터가 다수의 데이터센터 혹은 다수의 클라우드 가용 영역에 걸쳐 설치되어 있는 경우 컨슈머와 같은 영역에 있는 레플리카로부터 메시지를 읽어 오는 것이 성능적, 비용적으로 유리 

→ `client.rack` 사용

```
# 브로커 설정 (server.properties)

# 브로커가 속한 랙(AZ) 지정
broker.rack=us-east-1a

# 가까운 레플리카에서 읽을 수 있도록 셀렉터 지정
replica.selector.class=org.apache.kafka.common.replica.RackAwareReplicaSelector
```

```java
// 컨슈머 설정
Properties props = new Properties();
props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "broker1:9092,broker2:9092");
props.put(ConsumerConfig.GROUP_ID_CONFIG, "my-consumer-group");
props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
          "org.apache.kafka.common.serialization.StringDeserializer");
props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
          "org.apache.kafka.common.serialization.StringDeserializer");

// 컨슈머가 속한 랙(AZ) 지정
props.put("client.rack", "us-east-1a");

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("my-topic"));
```

## 4.5.15 group.instance.id

컨슈머에 정적 그룹 멤버십 기능을 적용하기 위해 사용되는 설정

- 어떤 고유한 문자열도 사용 가능

## 4.5.16 receive.buffer.bytes, send.buffer.bytes

데이터를 읽거나 쓸 때 소켓이 사용하는 **TCP의 수신 및 수신 버퍼의 크기

- `-1` 로 잡아주면 운영체제 기본값이 사용
- 다른 데이터센터에 있는 보렄와 통신하는 프로듀서나 컨슈머의 경우 이 값을 올려 잡아 주는 것이 좋음
- ****TCP 버퍼**
    - 애플리케이션(예: Kafka Producer/Consumer)이 데이터를 `send()` 하면, 곧바로 네트워크로 날아가는 게 아닙니다.
    - 운영체제(OS)가 관리하는 **소켓 버퍼(socket buffer)**에 잠시 저장됩니다.
        - `send buffer`: **보내기 전 대기실** — 애플리케이션이 보낸 데이터를 OS가 여기 쌓아 두고, 네트워크 카드가 순차적으로 전송
        - `receive buffer`: **받은 후 대기실** — 네트워크 카드가 받은 데이터를 OS가 여기에 쌓아 두고, 애플리케이션(컨슈머)이 꺼내감
    
    즉, 이 값은 **“네트워크 전송/수신 시 대기 공간 크기(Byte)”**를 말해요.
    

## 4.5.17 offsets.retention.minutes

브로커가 컨슈머 그룹의 커밋된 오프셋을 얼마나 오래 보관할지를 결정하는 설정(기본값: **10080분 = 7일**)

**다음 두 가지 조건이 모두 해당될 때, 해당 컨슈머 그룹의 오프셋이 만료되어 삭제:**

- 해당 그룹에 활성 소비자가 모두 없어지고 그룹이 비어 잇는 상태로 일정 시간이 흐른 경우
- 마지막으로 오프셋이 커밋된 이후 일정 시간이 흐른 경우

# 4.6 오프셋과 커밋

`오프셋 커밋:` 파티션에서의 현재 위치를 업데이트하는 작업

- 컨슈머는 파티션에서 성공적으로 처리해 낸 마지막 메시지를 커밋함으로써 그 앞의 모든 메시지들 역시 성공적으로 처리되었음을 암시
- `__consumer_offsets`라는 특수 토픽에 각 파티션별로 커밋된 오프셋을 업데이트하도록 하는 메시지를 보냄으로써 이루어짐
- 커밋된 오프셋이 클라이언트가 처리한 마지막 메시지의 오프셋보다 작을 경우:
    
    <img width="580" height="265" alt="image" src="https://github.com/user-attachments/assets/0b25f394-a067-4f6f-aef5-d9c3c909422f" />

    
- 커밋된 메시지가 클라이언트가 실제로 처리한 마지막 메시지의 오프셋보다 클 경우:
    
    <img width="595" height="313" alt="image" src="https://github.com/user-attachments/assets/9d64a2d9-de93-4f33-b009-f587b2149464" />

  
    

## 4.6.1 자동 커밋

`enable.auto.commit=true`: 커밋 시점과 실제 처리 완료 시점이 어긋날 수 있어 **중복 처리** 발생 가능

- 커밋 주기를 줄이면 중복 구간을 줄일 수 있지만, 완전히 없앨 수는 없음

## 4.6.2 현재 오프셋 커밋하기 (수동 커밋)

- `enable.auto.commit=false`로 설정하면, **개발자가 원하는 시점**에 오프셋 커밋 가능
- `commitSync()`는 **poll()이 반환한 마지막 오프셋**을 커밋하며, 성공할 때까지 재시도 → 실패 시 예외 발생
- **주의점**
    - `commitSync()`를 레코드 처리 전에 호출하면, 처리되지 않은 메시지를 **놓칠 위험** 있음
    - 처리 도중 애플리케이션이 크래시하면, **중복 처리**가 발생할 수 있음
    - 즉, “중복 vs 누락” 중 어느 쪽을 감수할지 트레이드오프 존재
- **베스트 프랙티스**
    - poll()로 받은 레코드를 **모두 처리한 후에** `commitSync()` 호출
    - 애플리케이션의 “처리 완료” 기준(저장/전송/가공 등)에 맞춰 커밋 시점 결정

## 4.6.3 비동기적 커밋

- **특징**
    - 브로커 응답을 기다리지 않고 오프셋 커밋 요청만 보내고 바로 다음 로직으로 진행 → **처리량(throughput)↑**
    - `commitSync()`처럼 재시도(retry)는 하지 않음
- **장점**
    - 애플리케이션이 **브로커 응답 때문에 블로킹되지 않음** → 성능 개선
    - 콜백(callback) 등록 가능 → 커밋 성공/실패 로깅, 모니터링 지표 수집에 활용
- **단점 / 주의점**
    - 재시도가 없기 때문에 커밋 손실 가능성 있음
    - 순서 문제 발생 위험 → 예: offset 3000 커밋 성공 후 offset 2000이 뒤늦게 retry되면 잘못된 오프셋으로 롤백됨 → 중복 처리 증가 가능
- **실무 대응 패턴**
    - 콜백에서 단순히 로깅만 하거나
    - 재시도하려면 sequence number(단조 증가값)를 사용해 최신 커밋인지 확인 후에만 재시도

## 4.6.4 동기적 커밋과 비동기적 커밋을 함께 사용하기

### **Sync + Async 혼합 패턴**

- 평소에는 `commitAsync()` 사용 → 빠르고 실패해도 다음 커밋이 사실상 재시도 역할
- 종료 시점(consumer close 직전, rebalance 직전)에는 `commitSync()` 사용 → 재시도하면서 **최종 커밋 보장**

## 4.6.5 특정 오프셋 커밋하기

- 기본 커밋(`commitSync()` / `commitAsync()`)은 항상 **poll()이 반환한 마지막 오프셋까지** 커밋함
- 만약 poll()에서 받은 배치가 크고, 중간까지 처리했을 때 중간 오프셋을 저장하고 싶으면 → **명시적으로 오프셋 맵(Map<TopicPartition, OffsetAndMetadata>)을 만들어 커밋**
- 오프셋 맵에는 **“다음에 읽을 메시지의 오프셋”**을 넣어야 함 (`record.offset() + 1`)
- 예시: 1000개 처리마다 현재 오프셋 맵을 업데이트하고 `commitAsync(currentOffsets, null)` 호출

→ **효과**: 큰 배치 처리 도중 장애/리밸런스가 발생해도, 이미 처리한 메시지를 다시 읽지 않아도 됨

# 4.7 리밸런스 리스너

### 1. **필요성**

- 리밸런스 시 파티션을 잃기 전에 **오프셋 커밋, 리소스 정리(DB 연결, 파일 핸들 등)** 필요
- 이를 위해 **ConsumerRebalanceListener**를 `subscribe()`에 등록해 특정 시점에 사용자 코드를 실행 가능

### 2. **세 가지 콜백 메서드**

- **`onPartitionsAssigned()`**
    - 새 파티션 할당 직후 실행 (컨슈머 시작 준비: 상태 로드, seek 등)
- **`onPartitionsRevoked()`**
    - 파티션 반납 직전 실행 (일반적 리밸런스 상황)
    - 여기서 **commitSync()**를 호출해 마지막 처리 오프셋 저장
- **`onPartitionsLost()`**
    - *협력적 리밸런스(cooperative)**에서만, 예외적으로 파티션이 곧바로 다른 컨슈머에 넘어간 경우 호출
    - 리소스 정리 필요

### 3. **주의사항 (Cooperative Rebalancing)**

- `onPartitionsAssigned()` : 매번 호출 (새 파티션 없으면 빈 리스트)
- `onPartitionsRevoked()` : 실제 반납할 때만 호출됨
- `onPartitionsLost()` : 예외 상황에서만 호출됨

### 4. **실제 예제 포인트**

- 리밸런스 발생 → `onPartitionsRevoked()` 안에서 `commitSync()` 호출 → 안전하게 오프셋 저장
- 평소에는 `commitAsync()`로 처리량 확보, 리밸런스 직전에는 `commitSync()`로 확실히 커밋

# 4.8 특정 오프셋의 레코드 읽어오기

`poll()`은 **마지막 커밋 오프셋부터 순서대로** 읽음

## API 제공

- `seekToBeginning(partitions)` → 파티션 처음부터 읽기
- `seekToEnd(partitions)` → 파티션 끝부터(새 메시지부터) 읽기
- `seek(partition, offset)` → 특정 오프셋부터 읽기

## 시간 기반 소비

- `offsetsForTimes()` 사용 → 특정 타임스탬프에 해당하는 오프셋 조회 후 `seek()`으로 이동

## 활용예시

- 장애 복구
- 특정 시점부터 재처리
- 늦은 컨슈머가 빠르게 따라잡기

# 4.9 폴링 루프를 벗어나는 방법

컨슈머는 보통 무한 루프(poll loop)로 동작 → 종료 시 별도 처리 필요

## 안전한 종료 방법

- 다른 스레드에서 **`consumer.wakeup()` 호출** → 현재 `poll()`이 **WakeupException** 발생하며 루프 탈출
- 종료 시 반드시 **`consumer.close()`** 호출 → 오프셋 커밋 + 그룹 코디네이터에 “떠난다” 알림 → 빠른 리밸런스 유도
- 보통 **ShutdownHook**에 `consumer.wakeup()` 넣어서 Ctrl+C 등 종료 시 깨끗하게 마무리

→ **poll timeout이 길면** wakeup을 활용, 짧으면 boolean 플래그로 루프 제어 가능

# 4.10 디시리얼라이저

## SerDes

- **SerDes = Serializer + Deserializer**
- Kafka Streams, Kafka Clients에서 메시지를 **바이트 <-> 객체**로 변환할 때 사용하는 기본 인터페이스.
- 예: String, Long, Avro, JSON, Protobuf 등을 직렬화/역직렬화할 때 활용.

## 실무에서의 활용

1. **Kafka Streams / Kafka Connect**
    - Kafka Streams DSL에서 `Consumed.with(keySerde, valueSerde)` 같은 형태로 여전히 사용
    - Connect에서는 Converter 개념으로 비슷한 역할 (AvroConverter, JsonConverter 등)
2. **Confluent Schema Registry 연동**
    - `KafkaAvroSerializer`, `KafkaAvroDeserializer` → Avro SerDes
    - `KafkaProtobufSerializer`, `KafkaProtobufDeserializer` → Protobuf SerDes
    - `KafkaJsonSchemaSerializer` → JSON Schema SerDes
        
        → 단순 SerDes보다 **Schema Registry 기반 직렬화**가 사실상 표준처럼 쓰임
        
3. **Spring Kafka / 기타 프레임워크**
    - Spring Kafka에서도 `Serde` 구현체를 지정해서 메시지를 객체로 매핑
    - 다만 JSON(ObjectMapper)이나 Protobuf 같은 고수준 라이브러리와 통합되어 자동화가 많이 되어 있음

## 4.10.1 커스텀 디시리얼라이저

커스텀 시리얼라이저와 디시리얼라이저를 직접 구현하는 것은 비권장 → 표준 메시지 형식 사용

## 4.10.2 Avro 디시리얼라이저 사용하기

```java
Duration timeout = Duration.ofMillis(100);
Properties props = new Properties();
props.put("bootstrap.servers"
,
"broker1:9092,broker2:9092");
props.put("group.id"
,
"CountryCounter");
props.put("key.deserializer"
,
"org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer"
,
"io.confluent.kafka.serializers.KafkaAvroDeserializer");
props.put("specific.avro.reader"
,
"true");
props.put("schema.registry.url"
, schemaUrl);
String topic = "customerContacts"
KafkaConsumer<String, Customer> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Collections.singletonList(topic));
System.out.println("Reading topic:" + topic);
while (true) {
ConsumerRecords<String, Customer> records = consumer.poll(timeout);
for (ConsumerRecord<String, Customer> record: records) {
System.out.println("Current customer name is: " +
record.value().getName());
}
consumer.commitSync();
}
```

# 4.11 독립 실행 컨슈머(standalone consumer): 컨슈머 그룹 없이 컨슈머를 사용해야 하는 이유와 방법

일반적으로는 컨슈머 그룹을 사용해 파티션이 자동 할당/리밸런스됨

하지만 **특정 컨슈머가 항상 특정 파티션(또는 모든 파티션)을 읽어야 할 때**는 굳이 그룹/리밸런스가 필요 없음

### **방법:**

- `subscribe()` 대신 **`assign()`*을 사용해 직접 파티션을 지정
- 오프셋 커밋은 여전히 가능하지만, 자동 리밸런스는 없음
- 코드 예시: `consumer.partitionsFor("topic")` → 파티션 리스트 조회 → `consumer.assign(partitions)`

### **특징 & 주의점:**

- 리밸런스 없음 → 단순하고 안정적
- 하지만 **토픽에 파티션이 늘어나도 자동 반영되지 않음** → 주기적으로 `partitionsFor()`를 확인하거나 애플리케이션 재시작 필요
