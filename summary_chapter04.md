# 4.1 카프카 컨슈머: 개념

## Kafka Consumer란?

- **정의**: Kafka의 토픽(Topic)에 저장된 메시지를 **읽어가는 주체(클라이언트)**
- Producer가 토픽에 메시지를 **쓰기(write)** 하면, Consumer가 이를 **읽기(read)** 해서 애플리케이션 로직에서 활용합니다.

## 핵심 특징

1. **구독(subscribe) 기반**
    - Consumer는 특정 토픽(혹은 여러 토픽)에 구독을 신청합니다.
    - 이후 Kafka Broker는 해당 토픽의 메시지를 Consumer에게 전달하지 않고, Consumer가 직접 `poll()` 메서드로 가져갑니다.
2. **Consumer Group 개념**
    - Consumer는 **그룹** 단위로 동작합니다.
    - 같은 그룹에 속한 Consumer들은 **토픽의 파티션을 나눠서 할당**받아 병렬 처리합니다.
    - 서로 다른 그룹에 속한 Consumer는 **각각 모든 메시지를 다 읽을 수 있습니다.**
    - 즉, **1개의 메시지를 여러 그룹이 독립적으로 소비 가능**
3. **오프셋 관리**
    - Consumer는 자신이 어디까지 읽었는지를 **Offset** 으로 추적합니다.
    - `auto.commit`(자동 커밋) 또는 `commitSync/commitAsync`(수동 커밋)을 통해 브로커에 저장.
    - 장애 발생 후 재시작하면, 저장된 offset부터 이어서 읽을 수 있음.
4. **Pull 기반**
    - Kafka는 Consumer에게 **푸시(push)** 하지 않습니다.
    - Consumer가 `poll()` 메서드를 호출해야만 데이터를 가져옵니다.
    - → backpressure(과부하 제어)가 자연스럽게 가능.

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

<img width="695" height="360" alt="image" src="https://github.com/user-attachments/assets/870a3575-0648-4016-88e4-9c8aa83f9c4b" />


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

### +) polling과 pull

**1. 폴링 (Polling)**

- **뜻**: 어떤 주체(클라이언트)가 주기적으로 다른 대상(서버, 큐 등)에 가서 "새로운 데이터가 있나요?" 하고 확인하는 방식.
- **특징**
    - **능동적 요청**: 요청자가 직접 계속 확인해야 함.
    - **비효율적일 수 있음**: 새로운 데이터가 없는데도 계속 확인해야 해서 리소스를 낭비할 수 있음.
    - **예시**
        - Kafka Consumer가 `poll()` 메서드로 브로커에서 메시지를 가져오는 동작.
        - 웹 브라우저가 일정 시간마다 서버에 Ajax 요청을 보내서 상태를 확인하는 방식.

**2. 풀 (Pull)**

- **뜻**: 어떤 주체(클라이언트)가 필요할 때마다 다른 대상(서버, 브로커 등)에게 요청을 보내고 그 응답으로 데이터를 가져오는 통신 패턴.
- **특징**
    - **요청 기반**: 데이터 전송은 항상 클라이언트 요청에 의해 시작됨.
    - **독립적 요청**: 매 요청은 이전 요청과 무관하게 처리됨 (stateless 구조일 때 특히 명확).
    - **확장성 용이**: 서버는 요청에만 응답하면 되므로 단순하고 다양한 클라이언트에서 접근하기 쉬움.
- **예시**
    - 웹 브라우저가 서버에 HTTP 요청을 보내고 응답 페이지를 받는 방식.
    - Kafka Consumer가 브로커에서 메시지를 가져오는 방식 (Kafka는 push가 아니라 pull 모델).
    - REST API 호출 (`GET /users?id=1`)

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

### +) latest와 earliest 사용 예제

### 1. `latest` (기본값)

- 의미: 유효한 오프셋이 없을 경우 **가장 최신(offset의 끝)** 부터 읽기 시작.
- 특징:
    - 새 메시지만 읽음.
    - 과거 데이터는 무시.
- 사용 예시:
    1. **실시간 알림 서비스**
        - 예: 주문 이벤트가 발생하면 카톡/메일 알림 발송.
        - 과거 주문 알림을 다시 보낼 필요는 없고, 새로 들어오는 주문만 처리하면 됨.
        - `auto.offset.reset=latest`
    2. **스트리밍 대시보드**
        - 실시간으로 들어오는 센서 데이터만 집계해서 차트에 표시.
        - 예전 데이터는 별도 저장소(DB, DWH)에서 조회.

 **2. `earliest`**

- 의미: 유효한 오프셋이 없을 경우 **파티션 맨 앞(0번 offset)** 부터 모든 데이터를 읽음.
- 특징:
    - 토픽에 쌓인 데이터를 처음부터 전부 소비.
    - 데이터 재처리 가능.
- 사용 예시:
    1. **데이터 적재 파이프라인 초기 구동 시**
        - 예: `orders` 토픽에 쌓인 과거 주문 데이터를 전부 읽어 **데이터 웨어하우스**로 적재.
        - 처음 파이프라인을 켰을 때, 과거 주문까지 다 가져가야 함.
        - `auto.offset.reset=earliest`
    2. **배치/백필(Backfill)**
        - 과거 로그 데이터를 처음부터 다시 분석할 필요가 있을 때.
        - 새로운 Consumer Group을 만들고 `earliest` 로 읽으면 과거 데이터부터 전부 처리 가능.

## 4.5.11 enable.auto.commit

컨슈머가 자동으로 오프셋을 커밋할지의 여부를 결정(**기본값: true**)

- `auto.commit.interval.ms` 를 사용해서 얼마나 자주 오프셋이 커밋될지 제어 가능

## 4.5.12 partition.assignment.strategy

`PartitionAssignor` 클래스는 컨슈머와 이들이 구독한 토픽들이 주어졌을 때 어느 컨슈머에게 어느 파티션이 할당될지를 결정하는 역할

**Kafka 2.5 ~ 현재 (3.x, 4.x 포함)**

- 기본값:
    
    ```java
    partition.assignment.strategy = [RangeAssignor, CooperativeStickyAssignor]
    ```
    
- 즉, **Range + CooperativeSticky** 두 개가 리스트로 설정됨
- 컨슈머끼리 협상(negotiate)해서 **가능하다면 CooperativeSticky**를 쓰고, 그렇지 못할 때는 RangeAssignor로 fallback

### 1. Range

`org.apache.kafka.clients.consumer.RangeAssignor`

컨슈머가 구독하는 각 토픽의 파티션들을 **연속된 그룹으로 나눠서 할당**

- 각 토픽의 할당이 독립적으로 이루어지기 때문에 첫 번째 컨슈머는 두 번쨰 컨슈머보다 더 많은 파티션을 할당받게 됨

### 2. RoundRobin

`org.apache.kafka.clients.consumer.RoundRobinAssignor`

모든 구독된 토픽의 모든 파티션을 가져다 순차적으로 하나씩 컨슈머에 할당

- 모든 컨슈머들이 완**전히 동일한 수(혹은 많아야 1개 차이)**의 파티션을 할당 받게 됨

### 3. Sticky

`org.apache.kafka.clients.consumer.StickyAssignor`

**목표:**

- 파티션들을 가능한 한 균등하게 할당
- 리밸런스가 발생했을 때 가능하면 많은 파티션들이 같은 컨슈머에 할당되게 함으로써 **할당된 파티션을 하나의 컨슈머에서 다른 컨슈머로 옮길 때 발생하는 오버헤드를 최소화**

→ 할당된 결과물의 균형 정도는 RoundRobin 할당자와 유사하지만 **이동하는 파티션의 수 측면에서는 Sticky 쪽이 더 작다.** 

→ 같은 그룹에 속한 컨슈머들이 서로 다른 토픽을 구독할 경우, Sticky 할당자를 사용한 할당이 RoundRobin 방식보다 더 균형잡히게 된다. 

### 4. Cooperative Sticky

`org.apache.kafka.clients.consumer.CooperativeStickyAssignor`

Sticky 할당자와 기본적으로 동일하지만, 컨슈머가 재할당되지 않는 파티션으로부터 레코드를 계속해서 읽어 올 수 잇도록 해주는 협력적 리밸런스 기능 지원 - 협력적 리밸런스 (p.88) 참고

### +) partition.assignment.strategy 4가지 이해하기

- **예시**
    
    # 예시 A) 단일 토픽, 파티션 수가 나누어떨어지지 않을 때
    
    - 토픽 **T(5파티션: 0..4)**
    - 컨슈머 그룹: **C1, C2, C3** (모두 T 구독)
    
    ### Range
    
    - 원리: 각 토픽별로 파티션을 **연속 구간**으로 쪼개서 앞 컨슈머부터 배분
    - 결과(예):
        - **C1**: 0,1
        - **C2**: 2,3
        - **C3**: 4
    - 특징: 파티션 5개를 3명이 나눠 갖다 보니 **앞쪽 컨슈머가 더 많이** 가져가는 경향
    
    ### RoundRobin
    
    - 원리: 구독된 **모든 파티션을 한 줄로 세워** 컨슈머에 **돌려가며** 1개씩 배정
    - 결과(예):
        - **C1**: 0,3
        - **C2**: 1,4
        - **C3**: 2
    - 특징: 가능한 한 **개수 균형**이 맞음(±1 이내)
    
    ### Sticky
    
    - 원리: **균등성 + 기존 배정 최대 유지**
    - 초기 결과(최초 배정): RoundRobin과 거의 동일 →
        - **C1**: 0,3 / **C2**: 1,4 / **C3**: 2
    
    ### CooperativeSticky
    
    - 초기 결과는 Sticky와 동일
    - 차이는 **리밸런스 시 동작**(예시는 아래 C에서)
    
    ---
    
    # 예시 B) 서로 **다른 토픽을 구독**하는 컨슈머들이 섞여 있을 때
    
    - **T1(6파티션: 0..5), T2(2파티션: 0..1)**
    - **C1, C2**: T1과 T2 **둘 다** 구독
    - **C3**: **T1만** 구독
    
    ### Range
    
    - **토픽별**로 독립 배정 → T1은 C1,C2,C3에게 연속구간으로, T2는 **C1·C2에게만** 배정
    - 결과(예):
        - T1: **C1**: 0,1 / **C2**: 2,3 / **C3**: 4,5
        - T2: **C1**: 0 / **C2**: 1
    - 총합:
        - **C1**: T1(2) + T2(1) = **3**
        - **C2**: T1(2) + T2(1) = **3**
        - **C3**: T1(2) = **2**
    - 특징: 이 케이스에서는 의외로 **균형이 맞게** 나옴. 하지만 토픽·구독 조합에 따라 불균형이 커질 수 있음(아래 예시 C 전 단계 참고)
    
    ### RoundRobin
    
    - 모든 구독 파티션을 한 줄로 세우되, **각 컨슈머가 구독하지 않은 토픽 파티션은 건너뜀**
    - 결과(예):
        - T1(6개)은 셋이 분배, T2(2개)는 C1·C2만 분배 → 최종 개수 **균형**이 잘 맞는 편
    
    ### Sticky
    
    - 구독이 다른 경우에도 **최대한 균등 + 기존 유지**를 목표로 잡음
    - 이 **“구독 불일치”** 상황에서 Sticky가 RoundRobin보다 **더 잘 맞추는 경우가 많음**
        - 이유: Sticky는 “개수 균등”을 맞추는 과정에서, 구독 불일치로 생기는 편향을 보정하려는 탐욕적 재배치를 수행
    
    ### CooperativeSticky
    
    - 배정 결과 성격은 Sticky와 동일
    - 리밸런스가 일어날 때 **중단 최소화**(부분 해제 → 부분 재배정)로 체감 성능 유리
    
    ---
    
    # 예시 C) **리밸런스**가 발생할 때 (컨슈머 **이탈**)
    
    - 초기 상태: 예시 A의 Sticky/CoopSticky 초기 결과
        - **C1**: 0,3 / **C2**: 1,4 / **C3**: 2 (T=5)
    
    ## 1) Sticky (이탈: C3 다운)
    
    - 목표: **기존 배정 최대 유지**
    - C3가 가진 파티션 **2만** 새로 배정, 나머지는 그대로
    - 결과:
        - **C1**: 0,3
        - **C2**: 1,4,**2** ← 파티션 2만 C2로 이동
    - 특징: **이동 파티션 수 최소화(=1개)**
    
    ## 2) RoundRobin (이탈: C3 다운)
    
    - 재계산 시 전체 파티션을 다시 도는 특성상, 결과가 크게 바뀔 수 있음
    - 결과(예):
        - **C1**: 0,3
        - **C2**: 1,2,4
    - 이번 케이스는 우연히 Sticky와 같지만, 일반적으로는 **더 많은 이동**이 발생 가능
    
    ## 3) CooperativeSticky (이탈: C3 다운)
    
    - **단계적 재배정**(협력적 리밸런스):
        1. **첫 단계**: “누가 무엇을 포기해야 하는지” 최소만 알림(이번 케이스는 C3가 사라져 포기분 2만 남음)
        2. **두 번째 단계**: 필요한 파티션만 **천천히 재할당**
    - 효과: **멈춤 시간 최소화**, 기존 처리 파이프라인 **흔들림 감소**
    
    ---
    
    # 예시 D) Range의 **토픽별 독립 배정** 때문에 생기는 불균형
    
    - **T1(5파티션), T2(5파티션)**
    - **C1**: T1만, **C2**: T2만, **C3**: T1+T2 구독
    
    ### Range
    
    - T1은 C1·C3끼리 5개 분할, T2는 C2·C3끼리 5개 분할
    - 결과(예):
        - T1: **C1**: 0,1,2 / **C3**: 3,4
        - T2: **C2**: 0,1,2 / **C3**: 3,4
    - 총합:
        - **C1**: 3
        - **C2**: 3
        - **C3**: 2(T1) + 2(T2) = **4** ← **C3가 과도하게 많아짐**
    - 특징: Range는 **토픽 단위로 각각 “연속 구간”을 나누기 때문에**, 다토픽·구독 불일치 상황에서 **특정 컨슈머로 쏠림**이 자주 발생
    
    ### RoundRobin / Sticky
    
    - 두 방식 모두 **전체 균형**을 더 잘 맞추는 편
    - Sticky는 이 균형을 유지한 채, **리밸런스 시 이동 최소화**까지 가져감
    - CooperativeSticky는 위에 더해 **중단 최소화(점진적 재배정)**
    
    ---
    
    ## 실무 요약 가이드
    
    - **균등 + 리밸런스 안정성**을 원하면: **CooperativeSticky**(최근 Kafka 기본값 경향)
    - **가장 단순하고 빠른 할당**만 필요하고, 구독이 모두 같고 토픽도 1~2개 수준이면: **RoundRobin**도 무난
    - **Range**는 토픽별 독립·연속 배정 특성 때문에 **구독 불일치/다토픽 환경**에서 **불균형**이 잦을 수 있음
    - **Sticky vs CooperativeSticky**: 결과는 비슷하지만, **Cooperative**는 리밸런스 시 **부분 해제 → 부분 재배정**으로 **밀림/정지 시간**을 훨씬 줄여줌

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

## 4.5.11 enable.auto.commit

컨슈머가 자동으로 오프셋을 커밋할지의 여부를 결정(**기본값: true**)

- `auto.commit.interval.ms` 를 사용해서 얼마나 자주 오프셋이 커밋될지 제어 가능

## 4.5.12 partition.assignment.strategy

`PartitionAssignor` 클래스는 컨슈머와 이들이 구독한 토픽들이 주어졌을 때 어느 컨슈머에게 어느 파티션이 할당될지를 결정하는 역할

**Kafka 2.5 ~ 현재 (3.x, 4.x 포함)**

- 기본값:
    
    ```java
    partition.assignment.strategy = [RangeAssignor, CooperativeStickyAssignor]
    ```
    
- 즉, **Range + CooperativeSticky** 두 개가 리스트로 설정됨
- 컨슈머끼리 협상(negotiate)해서 **가능하다면 CooperativeSticky**를 쓰고, 그렇지 못할 때는 RangeAssignor로 fallback

### 1. Range

`org.apache.kafka.clients.consumer.RangeAssignor`

컨슈머가 구독하는 각 토픽의 파티션들을 **연속된 그룹으로 나눠서 할당**

- 각 토픽의 할당이 독립적으로 이루어지기 때문에 첫 번째 컨슈머는 두 번쨰 컨슈머보다 더 많은 파티션을 할당받게 됨

### 2. RoundRobin

`org.apache.kafka.clients.consumer.RoundRobinAssignor`

모든 구독된 토픽의 모든 파티션을 가져다 순차적으로 하나씩 컨슈머에 할당

- 모든 컨슈머들이 완**전히 동일한 수(혹은 많아야 1개 차이)**의 파티션을 할당 받게 됨

### 3. Sticky

`org.apache.kafka.clients.consumer.StickyAssignor`

**목표:**

- 파티션들을 가능한 한 균등하게 할당
- 리밸런스가 발생했을 때 가능하면 많은 파티션들이 같은 컨슈머에 할당되게 함으로써 **할당된 파티션을 하나의 컨슈머에서 다른 컨슈머로 옮길 때 발생하는 오버헤드를 최소화**

→ 할당된 결과물의 균형 정도는 RoundRobin 할당자와 유사하지만 **이동하는 파티션의 수 측면에서는 Sticky 쪽이 더 작다.** 

→ 같은 그룹에 속한 컨슈머들이 서로 다른 토픽을 구독할 경우, Sticky 할당자를 사용한 할당이 RoundRobin 방식보다 더 균형잡히게 된다. 

### 4. Cooperative Sticky

`org.apache.kafka.clients.consumer.CooperativeStickyAssignor`

Sticky 할당자와 기본적으로 동일하지만, 컨슈머가 재할당되지 않는 파티션으로부터 레코드를 계속해서 읽어 올 수 잇도록 해주는 협력적 리밸런스 기능 지원 - 협력적 리밸런스 (p.88) 참고

### +) partition.assignment.strategy 4가지 이해하기

- **예시**
    
    # 예시 A) 단일 토픽, 파티션 수가 나누어떨어지지 않을 때
    
    - 토픽 **T(5파티션: 0..4)**
    - 컨슈머 그룹: **C1, C2, C3** (모두 T 구독)
    
    ### Range
    
    - 원리: 각 토픽별로 파티션을 **연속 구간**으로 쪼개서 앞 컨슈머부터 배분
    - 결과(예):
        - **C1**: 0,1
        - **C2**: 2,3
        - **C3**: 4
    - 특징: 파티션 5개를 3명이 나눠 갖다 보니 **앞쪽 컨슈머가 더 많이** 가져가는 경향
    
    ### RoundRobin
    
    - 원리: 구독된 **모든 파티션을 한 줄로 세워** 컨슈머에 **돌려가며** 1개씩 배정
    - 결과(예):
        - **C1**: 0,3
        - **C2**: 1,4
        - **C3**: 2
    - 특징: 가능한 한 **개수 균형**이 맞음(±1 이내)
    
    ### Sticky
    
    - 원리: **균등성 + 기존 배정 최대 유지**
    - 초기 결과(최초 배정): RoundRobin과 거의 동일 →
        - **C1**: 0,3 / **C2**: 1,4 / **C3**: 2
    
    ### CooperativeSticky
    
    - 초기 결과는 Sticky와 동일
    - 차이는 **리밸런스 시 동작**(예시는 아래 C에서)
    
    ---
    
    # 예시 B) 서로 **다른 토픽을 구독**하는 컨슈머들이 섞여 있을 때
    
    - **T1(6파티션: 0..5), T2(2파티션: 0..1)**
    - **C1, C2**: T1과 T2 **둘 다** 구독
    - **C3**: **T1만** 구독
    
    ### Range
    
    - **토픽별**로 독립 배정 → T1은 C1,C2,C3에게 연속구간으로, T2는 **C1·C2에게만** 배정
    - 결과(예):
        - T1: **C1**: 0,1 / **C2**: 2,3 / **C3**: 4,5
        - T2: **C1**: 0 / **C2**: 1
    - 총합:
        - **C1**: T1(2) + T2(1) = **3**
        - **C2**: T1(2) + T2(1) = **3**
        - **C3**: T1(2) = **2**
    - 특징: 이 케이스에서는 의외로 **균형이 맞게** 나옴. 하지만 토픽·구독 조합에 따라 불균형이 커질 수 있음(아래 예시 C 전 단계 참고)
    
    ### RoundRobin
    
    - 모든 구독 파티션을 한 줄로 세우되, **각 컨슈머가 구독하지 않은 토픽 파티션은 건너뜀**
    - 결과(예):
        - T1(6개)은 셋이 분배, T2(2개)는 C1·C2만 분배 → 최종 개수 **균형**이 잘 맞는 편
    
    ### Sticky
    
    - 구독이 다른 경우에도 **최대한 균등 + 기존 유지**를 목표로 잡음
    - 이 **“구독 불일치”** 상황에서 Sticky가 RoundRobin보다 **더 잘 맞추는 경우가 많음**
        - 이유: Sticky는 “개수 균등”을 맞추는 과정에서, 구독 불일치로 생기는 편향을 보정하려는 탐욕적 재배치를 수행
    
    ### CooperativeSticky
    
    - 배정 결과 성격은 Sticky와 동일
    - 리밸런스가 일어날 때 **중단 최소화**(부분 해제 → 부분 재배정)로 체감 성능 유리
    
    ---
    
    # 예시 C) **리밸런스**가 발생할 때 (컨슈머 **이탈**)
    
    - 초기 상태: 예시 A의 Sticky/CoopSticky 초기 결과
        - **C1**: 0,3 / **C2**: 1,4 / **C3**: 2 (T=5)
    
    ## 1) Sticky (이탈: C3 다운)
    
    - 목표: **기존 배정 최대 유지**
    - C3가 가진 파티션 **2만** 새로 배정, 나머지는 그대로
    - 결과:
        - **C1**: 0,3
        - **C2**: 1,4,**2** ← 파티션 2만 C2로 이동
    - 특징: **이동 파티션 수 최소화(=1개)**
    
    ## 2) RoundRobin (이탈: C3 다운)
    
    - 재계산 시 전체 파티션을 다시 도는 특성상, 결과가 크게 바뀔 수 있음
    - 결과(예):
        - **C1**: 0,3
        - **C2**: 1,2,4
    - 이번 케이스는 우연히 Sticky와 같지만, 일반적으로는 **더 많은 이동**이 발생 가능
    
    ## 3) CooperativeSticky (이탈: C3 다운)
    
    - **단계적 재배정**(협력적 리밸런스):
        1. **첫 단계**: “누가 무엇을 포기해야 하는지” 최소만 알림(이번 케이스는 C3가 사라져 포기분 2만 남음)
        2. **두 번째 단계**: 필요한 파티션만 **천천히 재할당**
    - 효과: **멈춤 시간 최소화**, 기존 처리 파이프라인 **흔들림 감소**
    
    ---
    
    # 예시 D) Range의 **토픽별 독립 배정** 때문에 생기는 불균형
    
    - **T1(5파티션), T2(5파티션)**
    - **C1**: T1만, **C2**: T2만, **C3**: T1+T2 구독
    
    ### Range
    
    - T1은 C1·C3끼리 5개 분할, T2는 C2·C3끼리 5개 분할
    - 결과(예):
        - T1: **C1**: 0,1,2 / **C3**: 3,4
        - T2: **C2**: 0,1,2 / **C3**: 3,4
    - 총합:
        - **C1**: 3
        - **C2**: 3
        - **C3**: 2(T1) + 2(T2) = **4** ← **C3가 과도하게 많아짐**
    - 특징: Range는 **토픽 단위로 각각 “연속 구간”을 나누기 때문에**, 다토픽·구독 불일치 상황에서 **특정 컨슈머로 쏠림**이 자주 발생
    
    ### RoundRobin / Sticky
    
    - 두 방식 모두 **전체 균형**을 더 잘 맞추는 편
    - Sticky는 이 균형을 유지한 채, **리밸런스 시 이동 최소화**까지 가져감
    - CooperativeSticky는 위에 더해 **중단 최소화(점진적 재배정)**
    
    ---
    
    ## 실무 요약 가이드
    
    - **균등 + 리밸런스 안정성**을 원하면: **CooperativeSticky**(최근 Kafka 기본값 경향)
    - **가장 단순하고 빠른 할당**만 필요하고, 구독이 모두 같고 토픽도 1~2개 수준이면: **RoundRobin**도 무난
    - **Range**는 토픽별 독립·연속 배정 특성 때문에 **구독 불일치/다토픽 환경**에서 **불균형**이 잦을 수 있음
    - **Sticky vs CooperativeSticky**: 결과는 비슷하지만, **Cooperative**는 리밸런스 시 **부분 해제 → 부분 재배정**으로 **밀림/정지 시간**을 훨씬 줄여줌

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
	- **상황**:
    - 애플리케이션은 offset 2까지 커밋했지만, 사실 offset 2~7까지 처리를 이미 끝낸 상태
    - 다만 아직 "커밋"은 offset 2까지만 되어 있음
- **리밸런스(또는 장애 복구) 발생** 시:
    - 컨슈머가 재시작되면 offset 2 이후(=3부터)의 이벤트를 다시 읽음
    - → 이미 처리했던 3~7이 다시 처리됨.
    - 즉 **중복 처리(duplicate processing)** 발생
- 문제: **At-Least-Once 처리**가 보장되지만, **중복 허용 로직**(idempotent 처리)이 필요함

    
- 커밋된 메시지가 클라이언트가 실제로 처리한 마지막 메시지의 오프셋보다 클 경우:
    
    <img width="595" height="313" alt="image" src="https://github.com/user-attachments/assets/9d64a2d9-de93-4f33-b009-f587b2149464" />
	- **상황**:
    - 애플리케이션은 offset 10까지 커밋했지만, 실제로는 offset 4까지밖에 처리 못함.
    - offset 5~10은 아직 “처리 중이거나 처리 전”인데 커밋만 먼저 됨.
- **리밸런스 발생** 시:
    - 컨슈머가 offset 11부터 다시 읽기 시작.
    - → 5~10 메시지는 **스킵**되고 영영 처리되지 않음.
    - 즉 **데이터 유실(message loss)** 발생.
- 💡 문제: **At-Most-Once 처리**가 되어버림. 손실이 가장 치명적.

### +) 주의: 메시지 처리 로직과 commit은 **분리**되어 실행

- **poll()**: 브로커에서 메시지를 가져오는 동작
- **메시지 처리 로직**: poll()로 가져온 record를 실제 비즈니스 로직에 전달해서 DB 저장, API 호출 등 “애플리케이션 레벨 처리”
- **commit**: “여기까지는 처리 완료했으니 안전하게 오프셋 저장하자”라고 브로커에 알려주는 단계

즉, “메시지 받기 → 처리하기 → commit하기”가 하나의 싸이클인데,

**commit 호출 시점**을 언제로 잡느냐에 따라 **중복 / 유실**이 달라짐

```java
ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
for (ConsumerRecord<String, String> record : records) {
    // 여기서 메시지 처리 로직 실행 - 1
    process(record);
}
// 처리가 끝나면 offset commit - 2
consumer.commitSync();
```

- commit이 실제 처리보다 **늦으면** 중복
- commit이 **빠르면** 유실이 발생
    

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
    - `commitAsync()`: 브로커 응답을 기다리지 않고 오프셋 커밋 요청만 보내고 바로 다음 로직으로 진행 → **처리량(throughput)↑**
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

### +) 비동기적 커밋 실무 대응패턴 - 최신 커밋만 유효하게 만들기

1. **단조증가를 사용해 최신 커밋인지 확인 후에만 재시도**
- 매 커밋 시도마다 **시퀀스 번호**를 1씩 증가
- 콜백 안에서 “내 콜백이 **가장 최신 시도**의 콜백인가?”를 확인
- 최신이 아니면(더 새로운 커밋이 이미 시도됨) **아무 것도 하지 않고 무시**
- 최신이고 실패했다면, 그때만 재시도(또는 플래그 세워 다음 루프에서 `commitSync()`)

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.TopicPartition;
import java.time.Duration;
import java.util.*;
import java.util.concurrent.atomic.AtomicLong;

public class AsyncCommitWithSeq {
  private static final AtomicLong commitSeq = new AtomicLong(0);      // 단조 증가 시퀀스
  private static volatile long lastTriedSeq = 0L;                      // 마지막 시도 번호 (옵션)
  private static volatile Map<TopicPartition, OffsetAndMetadata> lastTriedOffsets = Map.of();

  public static void main(String[] args) {
    Properties p = new Properties();
    p.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "broker:9092");
    p.put(ConsumerConfig.GROUP_ID_CONFIG, "g1");
    p.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
    p.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringDeserializer");
    p.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "false"); // 수동 커밋이 기본

    try (KafkaConsumer<String,String> consumer = new KafkaConsumer<>(p)) {
      consumer.subscribe(List.of("t1"));

      // 파티션별 "다음에 읽을 오프셋"(= 처리 완료 후 커밋할 오프셋)
      Map<TopicPartition, OffsetAndMetadata> nextCommit = new HashMap<>();

      while (true) {
        ConsumerRecords<String,String> records = consumer.poll(Duration.ofMillis(500));

        // 처리
        for (ConsumerRecord<String,String> r : records) {
          process(r); // 외부 시스템 반영까지 끝냈다고 가정
          TopicPartition tp = new TopicPartition(r.topic(), r.partition());
          // 커밋은 "다음에 읽을 위치"여야 하므로 현재 r.offset()+1
          nextCommit.put(tp, new OffsetAndMetadata(r.offset() + 1));
        }

        if (!nextCommit.isEmpty()) {
          // 1) 커밋 시퀀스 증가
          long seq = commitSeq.incrementAndGet();
          lastTriedSeq = seq;
          lastTriedOffsets = new HashMap<>(nextCommit); // 참고용 복사

          // 2) 비동기 커밋
          consumer.commitAsync(new HashMap<>(nextCommit), (offsets, exception) -> {
            // 3) 이 콜백이 최신 시도에 대한 콜백인지 확인
            if (seq != commitSeq.get()) {
              // 이미 더 새로운 커밋이 시도됨 → 오래된 콜백. 아무 것도 하지 않음.
              return;
            }

            if (exception != null) {
              // 최신 시도였고 실패했다면, 여기서만 재시도 정책 수행
              // - 즉시 재시도(주의: 과도한 루프 피하기)
              // - 또는 플래그 세워 다음 루프에서 commitSync() 수행
              try {
                // 간단 즉시 재시도 예시 (1회):
                consumer.commitSync(offsets); // 최신 시도이므로 롤백 위험 없음
              } catch (Exception e) {
                // 로그/알람
              }
            } else {
              // 성공 → 메트릭/로그만
            }
          });

          // 필요 시 배치마다 초기화
          nextCommit.clear();
        }
      }
    }
  }

  private static void process(ConsumerRecord<String,String> r) {
    // 비즈니스 처리 (DB upsert 등 멱등 설계 권장)
  }
}
```

1. **오프셋 자체 비교**

시퀀스 대신 **파티션별 최고 오프셋**을 저장해 두고, 콜백에서 **커밋하려는 오프셋이 현재 보관 중인 최고치보다 작으면 무시**하는 방식도 자주 씁니다.

```java
// 파티션별로 현재까지 "가장 큰(가장 최신)" 커밋 목표 오프셋
Map<TopicPartition, Long> maxCommitOffset = new HashMap<>();

// commitAsync 호출 직전
offsetsToCommit.forEach((tp, o) -> {
  maxCommitOffset.merge(tp, o.offset(), Math::max); // 최고값 갱신
});

consumer.commitAsync(offsetsToCommit, (offsets, ex) -> {
  boolean stale = offsets.entrySet().stream().anyMatch(e -> {
    TopicPartition tp = e.getKey();
    long committed = e.getValue().offset();
    Long maxSeen = maxCommitOffset.get(tp);
    return maxSeen != null && committed < maxSeen; // 더 오래된 커밋이면 무시
  });

  if (stale) return; // 오래된 콜백 → 무시

  if (ex != null) {
    // 최신 커밋 실패 → 재시도/보강
  }
});
```

**3. 운용 팁**

- **혼합 전략**: 평소엔 `commitAsync()`로 처리량↑,
    
    **주기적으로/종료 시/리밸런스 직전**에 `commitSync()`로 마지막 상태를 확정.
    
- **리밸런스 훅**: `ConsumerRebalanceListener`에서 **revoke** 이벤트 시 `commitSync()`로 안전하게 마무리.
- **배치 크기 제한**: `max.poll.records`/`pause-resume`로 처리시간을 제어해 커밋 주기 안정화.
- **멱등 처리**: DB는 **UPSERT(유니크키)**, 버전/타임스탬프 비교 등으로 **중복 처리 무해화**.

## 4.6.4 동기적 커밋과 비동기적 커밋을 함께 사용하기

### **Sync + Async 혼합 패턴**

- 평소에는 `commitAsync()` 사용 → 빠르고 실패해도 다음 커밋이 사실상 재시도 역할
- 종료 시점(consumer close 직전, rebalance 직전)에는 `commitSync()` 사용 → 재시도하면서 **최종 커밋 보장**

## 4.6.5 특정 오프셋 커밋하기

- 기본 커밋(`commitSync()` / `commitAsync()`)은 항상 **poll()이 반환한 마지막 오프셋까지** 커밋함
- **poll()은 batch 단위**로 레코드를 가져옴
- 그런데 장애가 나면 “batch 전부”를 다시 읽을 수 있음
    - 이 경우, 이미 처리 끝난 메시지도 다시 들어오므로 중복 처리 위험
- 그래서 **중간까지 처리했을 때도, 그 시점까지의 오프셋을 명시적으로 commit** 해두면
    
    → 재시작/리밸런스 시 거기부터 이어갈 수 있음
    
- 만약 poll()에서 받은 배치가 크고, 중간까지 처리했을 때 중간 오프셋을 저장하고 싶으면 → **명시적으로 오프셋 맵(Map<TopicPartition, OffsetAndMetadata>)을 만들어 커밋**
    
    ```java
    // 처리
            for (ConsumerRecord<String,String> r : records) {
              process(r); // 외부 시스템 반영까지 끝냈다고 가정
              TopicPartition tp = new TopicPartition(r.topic(), r.partition());
              // 커밋은 "다음에 읽을 위치"여야 하므로 현재 r.offset()+1
              nextCommit.put(tp, new OffsetAndMetadata(r.offset() + 1));
            }
    ```
    
- 오프셋 맵에는 **“다음에 읽을 메시지의 오프셋”**을 넣어야 함 (`record.offset() + 1`)
- 예시: 1000개 처리마다 현재 오프셋 맵을 업데이트하고 `commitAsync(currentOffsets, null)` 호출

→ **효과**: 큰 배치 처리 도중 장애/리밸런스가 발생해도, 이미 처리한 메시지를 다시 읽지 않아도 됨

# 4.7 리밸런스 리스너

## 1. **필요성**

- 리밸런스 시 파티션을 잃기 전에 **오프셋 커밋, 리소스 정리(DB 연결, 파일 핸들 등)** 필요:
    - 컨슈머가 갑자기 죽으면, **처리 중이던 메시지들의 offset 커밋이 누락**될 수 있음 → 중복 처리 발생
    - DB/파일 핸들, 네트워크 연결 등을 정리하지 않으면 **리소스 누수** 발생
    - 리밸런스 시에는 내가 가지고 있던 파티션이 다른 컨슈머로 넘어가기 때문에, 그 전에 **상태/오프셋을 정리**해야 다른 컨슈머가 이어받을 수 있음
- 이를 위해 **ConsumerRebalanceListener**를 `subscribe()`에 등록해 특정 시점에 사용자 코드를 실행 가능

## 2. **세 가지 콜백 메서드**

### **`onPartitionsAssigned()`**

- 새 파티션 할당 직후, 컨슈머가 메시지를 읽기 시작하기 전에 호출 (컨슈머 시작 준비: 상태 로드, seek 등)
- 컨슈머가 그룹에 문제없이 조인하려면 여기서 수행되는 모든 준비 작업은 `max.poll.timeout.ms`안에 완료되어야 함

### **`onPartitionsRevoked()`**

- 컨슈머가 할당받았던 파티션이 할당 해제될 때 호출
- 여기서 `commitSync()`를 호출해 마지막 처리 오프셋 저장
- 조급한 리밸런스 알고리즘 사용 경우:
    - 컨슈머가 메시지 읽기를 멈춘 뒤에, 그리고 리밸런스가 시작되기 전에 호출
- 협력적 리밸런스 알고리즘 사용 경우:
    - 리밸런스가 완료될 때, 컨슈머에서 할당 해제되어야 할 파티션들에 대해서만 호출

### **`onPartitionsLost()`**

- 협력적 리밸런스(cooperative)에서만, 예외적으로 파티션이 곧바로 다른 컨슈머에 넘어간 경우 호출
- 리소스 정리 필요
- 이 메서드를 구현하지 않았을 경우, `onPartitionsRevoked()`가 대신 호출

### 3. **주의사항 (Cooperative Rebalancing)**

- `onPartitionsAssigned()` : 리밸런싱이 발생할 때마다 호출 (새 파티션 없으면 빈 리스트)
- `onPartitionsRevoked()` : 일반적인 리밸런스 상황에서 호출되지만, 파티션이 특정 컨슈머에서 해제될 때만 호출(빈 리스트가 주어지는 경우는 없음)
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

## +) 활용예시

### 1. **장애 복구 (Recovery)**

- 상황: 컨슈머 애플리케이션이 DB에 쓰다가 장애 발생 → DB 트랜잭션은 롤백됐는데 offset은 이미 commit 됐을 수도 있음.
- 문제: 그대로 두면 **DB에는 데이터가 없는데 Kafka는 이미 읽었다고 offset이 앞으로 가 있음 → 데이터 유실**.
- 해결: `seek(partition, 특정 offset)`을 사용해 **장애 직전/안전한 지점부터 다시 읽기**.
    - 예: offset 2000까지 commit됐지만 실제 DB에는 1900까지만 저장됨 → `seek(partition, 1900)`으로 되돌려 읽음.

---

### 2. **특정 시점부터 재처리 (Reprocessing)**

- 상황:
    - 어제 배포한 코드에 버그가 있어서 잘못된 데이터가 DB에 들어감.
    - 오늘 그 버그를 고쳤는데, 어제 처리했던 데이터들을 다시 DB에 반영하고 싶음.
- 해결:
    - 어제 12시 기준 offset/timestamp를 찾음 (`offsetsForTimes()`)
    - `seek(partition, offsetAtTimestamp)`로 이동 → 어제 12시부터 다시 소비해서 **재처리 파이프라인 실행**.

즉, Kafka는 로그를 계속 보관하니까, **잘못된 처리를 되돌리고 다시 재처리할 수 있는 타임머신**처럼 쓸 수 있음.

---

### 3. **늦은 컨슈머가 빠르게 따라잡기 (Catch-up Consumer)**

- 상황: 신규 컨슈머 그룹이 토픽에 붙었는데, 과거 데이터가 너무 많음 (예: 수억 건).
- 그런데 이 컨슈머는 **실시간 데이터만** 필요하고, 예전 데이터는 안 봐도 됨.
- 해결:
    - `seekToEnd()`로 바로 끝으로 점프 → 최신 오프셋부터 읽기 시작.
    - → 과거 쌓인 데이터를 다 읽느라 몇 시간/며칠 기다릴 필요 없음.

반대로 **백필/분석용 컨슈머**는 `seekToBeginning()`으로 옛날 것까지 모두 읽음.

---

### 4. 그 외 자주 쓰이는 케이스

- **데이터 검증/디버깅**: 운영 중 특정 메시지가 의심될 때, 그 offset으로 직접 jump 해서 해당 레코드 확인
- **부분 집계 리셋**: 집계 컨슈머가 잘못 집계했을 때, 특정 offset 지점부터만 다시 돌려보고 싶을 때
- **DLQ(Dead Letter Queue) 재처리**: 문제가 있던 메시지(실패해서 다른 토픽에 쌓아둔 것)를 offset 기준으로 다시 처리

# 4.9 폴링 루프를 벗어나는 방법

컨슈머는 보통 무한 루프(poll loop)로 동작 → 종료 시 별도 처리 필요

## 안전한 종료 방법

- 다른 스레드에서 **`consumer.wakeup()` 호출** → 현재 `poll()`이 **WakeupException** 발생하며 루프 탈출
    - 여기서 `consumer.wakeup()`은 다른 스레드에서 호출해줄 때만 안전하게 작동하는 유일한 컨슈머 메서드
- 메인 스레드에서 컨슈머 루프를 돌고 있다면 `ShutdownHook` 사용 가능
- wakeup을 호출하던 대기중이던 poll()이 WakeupException을 발생시키며 중단되거나, 대기중이 아닐 경우네는 다음 번에 처음으로 poll()가 호출될 때 예외 발생
- 종료 시 반드시 **`consumer.close()`** 호출 → 오프셋 커밋 + 그룹 코디네이터에 “떠난다” 알림 → 빠른 리밸런스 유도
- 보통 **ShutdownHook**에 `consumer.wakeup()` 넣어서 Ctrl+C 등 종료 시 깨끗하게 마무리

→ **poll timeout이 길면** wakeup을 활용, 짧으면 boolean 플래그로 루프 제어 가능

# 4.10 디시리얼라이저

## SerDes

- 같은 데이터 타입의 시리얼라이저와 디시리얼라이저를 묶어 놓은 클래스
- `org.apache.kafka.common.serialization.Serdes`
- 직접 시리얼라이저, 디시리얼라이저 이름을 지정할 필요 없이 아래와 같이 원하는 객체를 얻어올 수 있음
    
    ```java
    Serializer<String> serializer = Serdes.String().serializer();
    // org.apache.kafka.common.serialization.StringSerializer 리턴
    DeSerializer<String> deserializer = Serdes.String().deserializer();
    // org.apache.kafka.common.deserialization.StringDeSerializer 리턴
    ```
    
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

컨슈머는 토픽을 구독하거나 스스로 파티션을 할당하거나 양자 택일

### **방법:**

- `subscribe()` 대신 `assign()`을 사용해 직접 파티션을 지정
- 오프셋 커밋은 여전히 가능하지만, 자동 리밸런스는 없음
    
    ```java
    Duration timeout = Duration.ofMillis(100);
    List<PartitionInfo> partitionInfos = null;
    partitionInfos = consumer.partitionsFor("topic");
    if (partitionInfos != null) {
    	for (PartitionInfo partition : partitionInfos)
    		partitions.add(
    			new TopicPartition(partition.topic(), partition.partition())
    			);
    		}
    		
    consumer.assign(partitions);
    	while (true) {
    		ConsumerRecords<String, String> records = consumer.poll(timeout);
    		for (ConsumerRecord<String, String> record: records) {
    			System.out.printf("topic = %s, partition = %s, offset = %d,
    				customer = %s, country = %s\n",
    				record.topic(), record.partition(), record.offset(),
    				record.key(), record.value());
    		}
    		consumer.commitSync();
    	}
    }
    ```
    
- 코드 예시: `consumer.partitionsFor("topic")` → 파티션 리스트 조회 → `consumer.assign(partitions)`

### **특징 & 주의점:**

- 리밸런스 없음 → 단순하고 안정적
- 하지만 **토픽에 파티션이 늘어나도 자동 반영되지 않음** → 주기적으로 `partitionsFor()`를 확인하거나 애플리케이션 재시작 필요
