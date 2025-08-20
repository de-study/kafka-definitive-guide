# 3장 Kafka 프로듀서

# Introduction
- 클라이언트 API = 앱이 카프카와 메시지를 주고받거나 처리할 수 있게 해주는 라이브러리
- 분류
  - **공식 클라이언트**                    : 카프카가 직접 제공하고, 안정성과 호환성을 보장하는 라이브러리
    - 예: Java, Python(kafka-python, confluent-kafka-python), Go, C# 등
  - **비공식/서드파티 클라이언트**  :  외부 개발자가 만든 라이브러리로, 공식 문서에는 없지만 사용 가능

- 필수 클라이언트 API
  - Producer API: 메시지 보내기
  - Consumer API: 메시지 받기
  - Streams API: 실시간 데이터 처리
  - Admin API: 토픽/클러스터 관리
  - 실무 예시:
    - 쇼핑몰 앱에서 주문 발생 → Producer API로 카프카에 발행
    - 배송 서비스 → Consumer API로 주문 읽음
    - 데이터팀 → Streams API로 실시간 집계
    - 운영팀 → Admin API로 토픽 관리

- 배울내용
  -  Kafka 프로듀서를 사용하는 방법 -> 클라이언트API 에서 Producer를 사용하는방법
  - KafkaProducer 및 ProducerRecord 객체를 생성하는 방법
  -  레코드를 Kafka로 전송하는 방법과 Kafka가 반환할 수 있는 오류를 처리하는 방법
  - 프로듀서 동작을 제어하는 가장 중요한 설정 옵션
  - 다양한 파티셔닝 방법과 직렬화기(serializers)를 사용하는 방법
  - 사용자 정의 직렬화기 및 파티셔너를 작성하는 방법
<br><br>



# 3.1 프로듀서 개요
## (1) 메시지 전송과정
**`1.메시지 수신 → 2. 직렬화 → 3. 파티션 결정 → 4. 버퍼 저장 → 5. 배치 생성 → 6. 네트워크 전송 → 7. 응답 처리`**
<br> 
<img width="888" height="534" alt="image" src="https://github.com/user-attachments/assets/1bbb8da8-8cfb-4bfc-9f10-0fdc9a3598a2" />

<br>  


### Step 1. 메시지 수신 및 검증

- **목적:** 잘못된 메시지나 존재하지 않는 토픽으로 인한 오류를 사전에 방지하기 위함
- **작업 흐름:**
  - 애플리케이션이 `producer.send()` 호출 -> ProducerRecord객체 생성  
  - 토픽명 유효성 검사
  - 브로커에서 토픽 메타데이터 조회 (파티션 수, 리더 브로커 위치)
  - 타임아웃 설정에 따라 메타데이터 대기
- 필수 요소:
  - Topic (토픽명)
  - Value (메시지 내용)
- 선택 요소:
  - Key (파티셔닝 기준)
  - Partition (명시적 파티션 지정) - key없으면 라운드로
  - Timestamp (메시지 시점)
  - Headers (추가 메타데이터)
- 실무 예시: 신용카드 거래는 transaction_id를 키로, 웹 클릭은 키 없이 전송
---

### Step 2. 직렬화 (Serialization)

- **목적:** 파이썬 객체를 네트워크 전송 가능한 바이트 배열로 변환
- **내부 처리:**
  - Key Serializer 적용: 문자열 → UTF-8 바이트
  - Value Serializer 적용: 딕셔너리 → JSON → UTF-8 바이트
  - 직렬화 실패 시 즉시 예외 발생

---

### Step 3. 파티션 결정 (Partitioning)

- **목적:** 메시지의 파티션 할당으로 로드 분산 및 순서 보장
- **결정 로직:**
  - 명시적 파티션 지정 → 해당 파티션 사용
  - 키가 있으면 → 키 해시값으로 파티션 결정 (같은 키 = 같은 파티션)
  - 키가 없으면 → 라운드로빈(순차 분배) 방식
- 실무 활용: 같은 사용자 이벤트는 같은 파티션으로 (순서 유지)
---

### Step 4. RecordAccumulator에 버퍼링

- **목적:** 여러 메시지를 모아 배치 처리로 네트워크 효율성 향상
- **내부 구조:**
  - 파티션별로 별도 큐 유지
  - 각 파티션마다 RecordBatch 생성
  - 메모리 풀에서 버퍼 할당
  - buffer_memory 한계 도달 시 블로킹 또는 예외 발생

---

### Step 5. 배치 생성 및 관리

- **목적:** 개별 메시지를 효율적인 전송 단위(배치)로 그룹화
- **배치 생성 조건:**
  - batch_size 도달 (예: 16KB)
  - linger_ms 시간 경과 (예: 10ms)
  - 버퍼 메모리 부족
  - flush() 명시적 호출
- 메모리 구조: 파티션별로 별도 RecordBatch 생성 및 관리
---

### Step 6. Sender Thread의 네트워크 전송

- **목적:** 메인 스레드 블로킹 없이 백그라운드로 비동기 전송
- **Sender Thread 작업 흐름:**
  - RecordAccumulator에서 준비된 배치 수집
  - 브로커별로 요청 그룹화
  - NetworkClient를 통해 TCP 연결로 전송
  - `acks` 설정에 따라 응답 대기

---

### Step 7. 응답 처리 및 재시도

- **목적:** 전송 실패 감지 및 데이터 손실 방지
- **응답 처리 방식:**
  - 성공: Future 완료, 메타데이터(RecordMetadata 객체) 반환 (offset, partition, timestamp) 
  - 실패: retries 횟수만큼 재시도
  - 최종 실패: Exception으로 Future 완료
- 주요 예외 타입(오류처리):
  - BufferExhaustedException: 프로듀서 버퍼 가득참
  - TimeoutException: 브로커 응답 시간 초과
  - InterruptException: 전송 스레드 중단
  - SerializationException: 직렬화 실패
  - 재시도 메커니즘: retries 설정에 따른 자동 재시도
---

### 메모리 관리 구조

- **프로듀서 내부 메모리:**
  - 파티션별 RecordAccumulator 큐
  - 각 큐 내 RecordBatch 단위로 메시지 그룹핑 및 버퍼 관리

---

### 실제 동작 시나리오

1. **첫 번째 메시지:** 새로운 RecordBatch 생성, 버퍼에 저장
2. **두 번째 메시지:** 같은 파티션이면 기존 배치에 추가
3. **세 번째 메시지:** 배치 크기 or 시간 조건 만족 시 Sender로 전달
4. **백그라운드:** Sender가 브로커로 배치 전송
5. **응답:** 브로커에서 확인 후 Future 완료

---

### 에러 발생 지점별 처리

> 엔지니어 독백:  
> "프로듀서에서 에러가 나면 어느 단계에서 발생했는지 파악해야 한다.  
> 로그를 보면 대부분 직렬화, 버퍼 오버플로우, 네트워크 타임아웃 중 하나다."

| 단계         | 예외 타입                    | 주요 원인                               | 예외 발생 위치              |
|--------------|-----------------------------|----------------------------------------|--------------------------|
| 직렬화       | SerializationError          | JSON 변환 불가능 객체                  | 애플리케이션 스레드       |
| 버퍼링       | KafkaTimeoutError           | buffer_memory 부족, max_block_ms 초과  | RecordAccumulator 메모리 할당 시 |
| 네트워크     | TimeoutError, BrokerNotAvailableError | 브로커 장애, 네트워크 지연        | Sender 스레드            |

---
### 사용 사례별 설정 전략
- **고신뢰성 시나리오 (신용카드 거래)**
  - acks=all: 모든 복제본 확인
  - retries=MAX: 최대 재시도
  - enable.idempotence=true: 중복 방지
  - 동기 전송 사용
- **고성능 시나리오 (웹 클릭)**
  - acks=1: 리더만 확인
  -batch.size 크게 설정
  - linger.ms 적절히 설정
  - 비동기 전송 사용
 
<br><br>  











# 3.2 프로듀서 생성
<br> 



## (1) Producer생성 - 코드
`Properties 객체 생성(props)` → `필수 설정 추가(props.put)` → `KafkaProducer 인스턴스 생성(producer)` → `producer.send() 전송`


```
#자바
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import java.util.Properties;

Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092"); // 브로커 주소
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer"); // 키 직렬화
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer"); // 값 직렬화

props.put("acks", "all"); // 데이터 보장 수준
props.put("retries", 3); // 전송 실패 시 재시도 횟수
props.put("linger.ms", 1); // 메시지 배치 지연 시간

KafkaProducer<String, String> producer = new KafkaProducer<>(props);

producer.send(new ProducerRecord<>("my-topic", "key1", "Hello Kafka!"));
producer.close();
```




## (2) 기본 인자값 설정
### `bootstrap.servers`
- 목적: Kafka 클러스터와의 초기 연결을 위한 브로커 목록 지정
  - 모든 브로커를 포함할 필요 없음 (초기 연결 후 자동으로 전체 클러스터 정보 획득)
  - 최소 2개 이상 권장 (브로커 장애 시 연결 가능성 확보)
 
```
bootstrap_servers=[
    'kafka-broker-1:9092',
    'kafka-broker-2:9092', 
    'kafka-broker-3:9092'
]

# 클라우드 환경 (AWS MSK 예시)
bootstrap_servers=[
    'b-1.mycluster.abc123.c2.kafka.us-east-1.amazonaws.com:9092',
    'b-2.mycluster.abc123.c2.kafka.us-east-1.amazonaws.com:9092'
]

# 일반 연결 (PLAINTEXT)
bootstrap_servers=['kafka1:9092', 'kafka2:9092']

# SSL 연결(9093) - 암호화
bootstrap_servers=['kafka1:9093', 'kafka2:9093']

# SASL 인증(9094) - 사용자/패스워드 인증
bootstrap_servers=['kafka1:9094', 'kafka2:9094']
```



### `key.serializer`
- 목적: 메시지 키를 바이트 배열로 변환하는 직렬화 클래스 지정
  - Kafka 브로커는 바이트 배열만 처리 가능
  - 내장 직렬화 클래스 제공: StringSerializer, IntegerSerializer 등
  - 키를 사용하지 않아도 설정 필수 (VoidSerializer 사용 가능)



### `value.serializer`
- 목적: 메시지 값을 바이트 배열로 변환하는 직렬화 클래스 지정
  - key.serializer와 동일한 방식으로 동작



<details>
<summary>Serializer 종류</summary>
# Kafka 내장 직렬화 클래스 (Built-in Serializers)

## 1. 기본 데이터 타입용

| 번호 | 클래스 | Java 경로 | 용도 | 변환 예시 |
|------|-------|-----------|------|-----------|
| 1 | StringSerializer | `org.apache.kafka.common.serialization.StringSerializer` | 문자열 데이터 | `"Hello World"` → 바이트 배열 |
| 2 | ByteArraySerializer | `org.apache.kafka.common.serialization.ByteArraySerializer` | 이미 바이트 배열인 데이터 | `byte[]` → 바이트 배열 (변환 없음) |
| 3 | ByteBufferSerializer | `org.apache.kafka.common.serialization.ByteBufferSerializer` | ByteBuffer 객체 | `ByteBuffer` → 바이트 배열 |
| 4 | BytesSerializer | `org.apache.kafka.common.serialization.BytesSerializer` | Kafka Bytes 객체 | `Bytes` → 바이트 배열 |

## 2. 숫자 타입용

| 번호 | 클래스 | Java 경로 | 용도 | 변환 예시 |
|------|-------|-----------|------|-----------|
| 5 | ShortSerializer | `org.apache.kafka.common.serialization.ShortSerializer` | 16비트 정수 (Short) | `12345` → 2바이트 배열 |
| 6 | IntegerSerializer | `org.apache.kafka.common.serialization.IntegerSerializer` | 32비트 정수 (Integer) | `1234567` → 4바이트 배열 |
| 7 | LongSerializer | `org.apache.kafka.common.serialization.LongSerializer` | 64비트 정수 (Long) | `123456789L` → 8바이트 배열 |
| 8 | FloatSerializer | `org.apache.kafka.common.serialization.FloatSerializer` | 32비트 부동소수점 | `123.45f` → 4바이트 배열 |
| 9 | DoubleSerializer | `org.apache.kafka.common.serialization.DoubleSerializer` | 64비트 부동소수점 | `123.456789` → 8바이트 배열 |

## 3. 특수 목적용

| 번호 | 클래스 | Java 경로 | 용도 | 변환 예시 |
|------|-------|-----------|------|-----------|
| 10 | VoidSerializer | `org.apache.kafka.common.serialization.VoidSerializer` | 키나 값을 사용하지 않을 때 | `null` → `null` |
| 11 | UUIDSerializer | `org.apache.kafka.common.serialization.UUIDSerializer` | UUID 객체 | `UUID.randomUUID()` → 바이트 배열 |

---

**엔지니어 독백:**
"내장 직렬화만으로도 90%의 경우는 해결된다. StringSerializer가 가장 범용적이고, 특별한 요구사항이 있을 때만 다른 걸 고려하면 된다. 성능이 중요하면 숫자 타입 직렬화를, 호환성이 중요하면 JSON이나 Avro를 선택하는 게 좋다."

**<정리>**
- 기본: StringSerializer (가장 범용적)
- 숫자: IntegerSerializer, LongSerializer 등
- 바이너리: ByteArraySerializer
- 키 없음: VoidSerializer
- 고급: JSON, Avro, Protobuf (별도 라이브러리)
</details>
<br>  







## (2) 메시지 전송방식  
<3가지 메시지 전송 방식에 따라 코드가 달라짐>



### 1. Fire-and-forget (전송 후 망각)
- 특징: 메시지 전송 후 성공/실패 여부 확인하지 않음
- 장점: 가장 빠른 성능
- 단점: 재시도 불가능한 오류나 타임아웃 시 메시지 손실 가능
- 용도: 성능이 중요하고 일부 메시지 손실이 허용되는 경우
```
#파이썬
# 그냥 send()만 호출하고 결과 무시
producer.send('my_topic', value={'data': 'example'})
producer.send('my_topic', value={'data': 'example2'})
producer.send('my_topic', value={'data': 'example3'})

# 결과를 받지도 않고 대기하지도 않음
print("메시지 전송 완료") # 실제로는 아직 전송 중일 수 있음
```
```
//자바  
KafkaProducer<String, String> producer = new KafkaProducer<>(props);

ProducerRecord<String, String> record = new ProducerRecord<>("topic", "key", "value");

// 전송 후 결과 확인하지 않음
producer.send(record);

// 다음 작업 바로 진행
System.out.println("메시지 전송 완료 처리 없이 진행");
```





### 2. Synchronous send (동기 전송)
- 특징: send() 메서드가 Future 객체 반환 → get()으로 결과 대기
- 장점: 전송 성공/실패를 확실히 확인 가능
- 단점: 각 메시지마다 응답 대기로 성능 저하
- 용도: 신뢰성이 중요한 중요한 메시지
```
#파이썬
try:
    future = producer.send('my_topic', value={'data': 'important'})
    record_metadata = future.get(timeout=10)  # 10초 대기
    print(f"전송 성공: offset={record_metadata.offset}")
except Exception as e:
    print(f"전송 실패: {e}")
```
```
//자바
KafkaProducer<String, String> producer = new KafkaProducer<>(props);

ProducerRecord<String, String> record = new ProducerRecord<>("topic", "key", "value");

try {
    // send() 결과를 기다림 → 브로커 응답 확인
    RecordMetadata metadata = producer.send(record).get();
    System.out.printf("전송 성공, 파티션=%d, 오프셋=%d%n", metadata.partition(), metadata.offset());
} catch (InterruptedException | ExecutionException e) {
    e.printStackTrace();
    System.out.println("메시지 전송 실패");
}
```




### 3. Asynchronous send (비동기 전송)
- 특징: send() 메서드에 콜백 함수 전달 → 브로커 응답 시 콜백 실행
- 장점: 성능과 신뢰성의 균형
- 단점: 콜백 함수 구현 필요
- 용도: 대부분의 실무 상황에서 권장



```
# 파이썬  
def callback(metadata):
    print(f"성공: {metadata.offset}")

def errback(error):
    print(f"실패: {error}")

future = producer.send('topic', value=data)
future.add_callback(callback)
future.add_errback(errback)
```




```
// 자바   
KafkaProducer<String, String> producer = new KafkaProducer<>(props);

ProducerRecord<String, String> record = new ProducerRecord<>("topic", "key", "value");

// 비동기 전송 + 콜백
producer.send(record, (metadata, exception) -> {
    if (exception != null) {
        // 전송 실패 처리
        System.out.println("메시지 전송 실패: " + exception.getMessage());
    } else {
        // 전송 성공 처리
        System.out.printf("전송 성공, 파티션=%d, 오프셋=%d%n", metadata.partition(), metadata.offset());
    }
});

System.out.println("send() 호출 후 바로 다음 작업 진행");
```



### 핵심 특징
- 프로듀서 본질: Kafka 프로듀서는 본질적으로 비동기 방식으로 동작
- 제어 방식: 대부분의 동작은 설정 프로퍼티를 통해 제어
- 스레드 안전성: 하나의 프로듀서 객체를 여러 스레드에서 공유 가능

**엔지니어 독백:**
 "프로듀서 생성은 단순하지만, 3가지 전송 방식의 트레이드오프를 이해하는 게 핵심이다. 대부분의 경우 비동기 전송이 성능과 신뢰성의 최적 균형점이다."
<br><br> 













# 3.3 카프카로 메시지 전달하기
## Fire-and-forget (가장 간단)
- 특징:
  - 전송 후 결과 확인 안함
  - 성능 최고, 신뢰성 최저
  - 메시지 손실 가능성 있음
  - 예외: 전송 전 오류만 감지 (SerializationException, BufferExhaustedException 등)
 


## 3.3.1 메시지 전송 - 동기적
- 특징:
  - Future.get()으로 결과 대기
  - 전송 성공/실패 확실히 확인
  - 성능 저하: 각 메시지마다 대기 (10ms 네트워크 시 100개 메시지 = 1초)
  - 프로덕션에서 거의 안씀


## 3.3.2 메시지 전송 - 비동기적
- 특징:
  - 성능과 신뢰성의 균형
  - 콜백으로 성공/실패 처리
  - 주의: 콜백은 프로듀서 메인 스레드에서 실행 (빠르게 처리해야 함)


### 에러 처리
- 재시도 가능한 에러 (Retriable)
  - 연결 오류, 리더 없음 등
  - 프로듀서가 자동 재시도
    <details>
    <summary>예시</summary>
    
    **네트워크/연결 관련**
    - TimeoutException: 브로커 응답 시간 초과
    - NetworkException: 네트워크 연결 장애
    - DisconnectException: 브로커와 연결 끊김
    
    **리더십/메타데이터 관련**
    - NotLeaderForPartitionError: 해당 파티션의 리더가 아님
    - LeaderNotAvailableError: 파티션 리더를 찾을 수 없음
    - UnknownTopicOrPartitionError: 토픽/파티션 정보 없음 (일시적)
    
    **브로커 상태 관련**
    - BrokerNotAvailableError: 브로커 일시 불가
    - RequestTimedOutError: 요청 처리 시간 초과
    - OffsetOutOfRangeError: 오프셋 범위 벗어남
    
    **동작: 프로듀서가 retries 설정만큼 자동 재시도**
    
    </details>
<br>  

- 재시도 불가능한 에러 (Non-retriable)
  - 메시지 크기 초과 등
  - 즉시 예외 반환
    <details>
    <summary>예시</summary>
    
    **메시지 크기/형식 문제**
    - MessageSizeTooLargeError: 메시지 크기 한계 초과
    RecordTooLargeError: 레코드 크기 초과
    SerializationException: 직렬화 실패
    
    **인증/권한 문제**
    - TopicAuthorizationError: 토픽 접근 권한 없음
    - ClusterAuthorizationError: 클러스터 접근 권한 없음
    - SaslAuthenticationError: SASL 인증 실패
    
    **설정/구성 오류**
    - InvalidTopicError: 잘못된 토픽명
    - InvalidConfigurationError: 잘못된 설정
    - UnsupportedVersionError: 지원하지 않는 프로토콜 버전
    
    **리소스 부족**
    BufferExhaustedException: 프로듀서 버퍼 부족
    InterruptException: 스레드 중단
    
    **동작: 즉시 예외 발생, 재시도 안함**
    </details>

### 권장: 대부분 상황에서 비동기 전송 사용
<br><br>  



# 3.4 프로듀서 설정

> Kafka 프로듀서 설정을 배우는 건 단순히 옵션 외우기가 아니야.  
> ‘내 서비스에서 가장 중요한 건 무엇인가?’ 그걸 기준으로 잡는 거지.  
> 트래픽이 미친 듯이 몰리는 상황에서 메시지 하나라도 잃으면 안 되는지,  
> 아니면 약간의 데이터 유실은 괜찮으니 밀리초 단위의 지연을 줄이는 게 중요한지.  
> 
> 또 하나는 **운영 관점**에서 보는 습관이 필요해.  
> 설정은 코드 안에서 끝나는 게 아니라, 클러스터 복구 시간, 네트워크 병목, 디스크 IO, 팀 SLA 같은 현실적인 제약 조건을 반영해야 하거든.  
> 결국 튜닝은 **‘내 시스템이 감당할 수 있는 리스크의 수준’을 정하는 일**이야.  

```
// Kafka Producer 설정 예시 (Java)
// 신뢰성 + 성능 균형 잡힌 설정
Properties props = new Properties();
props.put("bootstrap.servers", "broker1:9092,broker2:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

//프로듀서 식별자
props.put("client.id", "order-service-producer");

// 신뢰성 관련
props.put("acks", "all"); // durability 보장
props.put("enable.idempotence", "true"); // 중복 방지, 순서 보장
props.put("delivery.timeout.ms", 120000); // 최대 재시도 시간 (2분)

// 성능 관련
props.put("linger.ms", 5); // batching 효과
props.put("batch.size", 32768); // 32KB
props.put("compression.type", "snappy"); // 네트워크/스토리지 최적화

// 안정성 관련
props.put("buffer.memory", 67108864); // 64MB 버퍼
props.put("max.in.flight.requests.per.connection", 5);
props.put("retries", Integer.MAX_VALUE); // 사실상 무한 재시도
props.put("request.timeout.ms", 30000); // 각 요청 단위 타임아웃
```
<br>

### 🚀 엔지니어의 독백 (Kafka Producer 튜닝을 배우며)

<details>
<summary>펼쳐보기</summary>

### ✅ Kafka Producer 설정을 배우며 고려해야 할 기준과 목표

#### 목적 / 기준
- **안정성 vs 성능 트레이드오프**
  - 금융 트랜잭션 → `acks=all`, `enable.idempotence=true`
  - 로그/모니터링 → `acks=0` 또는 `acks=1`, 성능 우선
- **메시지 손실 허용 여부** → SLA 기준으로 판단  
- **네트워크 환경** (동일 DC vs 멀티 리전)  

#### 중점 고려사항
- **신뢰성**
  - `acks`, `delivery.timeout.ms`, `enable.idempotence`
- **성능/Throughput**
  - `batch.size`, `linger.ms`, `compression.type`
- **리소스 관리**
  - `buffer.memory`, `max.in.flight.requests.per.connection`
- **운영 편의성**
  - `client.id` (로그 추적, 모니터링 편리)

#### 필요 역량
- 분산 시스템 기본 지식 (리더-팔로워, 복제, 장애 복구)  
- 네트워크/스토리지 병목 이해  
- SLA 기준에 따른 설계 경험  
- 모니터링/알람 구축 (latency, retries, throughput 확인)  

#### 실무 목표
- 장애 발생 시 **서비스 다운 없이 데이터 유실 최소화**  
- **적절한 리소스 비용**으로 SLA 충족  
- 운영팀이 장애 상황을 쉽게 트러블슈팅 할 수 있는 **설정과 로그 구조** 만들기  

</details>
<br>

## 식별자
###  (1) `client.id`
- 역할 : 클라이언트 → 프로듀서(혹은 컨슈머) 인스턴스를 구분하는 라벨 (Kafka-서버, 나머지 애들은다 병ㅍ클라이언트)
```
#파이썬  
producer = KafkaProducer(
    client_id='order-service-producer'  # 문제 발생 시 쉬운 식별
)
```
```
#브로커 로그에 찍히는 로그에서 확인가능
[2025-08-19 10:32:45,123] INFO [ReplicaFetcher replicaId=2, leaderId=1, clientId=order-service-producer] 
Successfully sent record to topic=orders partition=0 offset=12345 (kafka.server.ReplicaFetcherThread)
```

**엔지니어 독백:**
> "client.id를 제대로 안 정하면 장애 시 'IP 104.27.155.134에서 인증 실패'라고 나와서 누구 서비스인지 모른다. 'order-validation-service 인> 증 실패'라고 나오면 바로 담당자 연락 가능하다."
<br>  
<br>  


## 신뢰성(Durability)
### (2) `acks`
- 의미: 프로듀서가 메시지를 “성공적으로 보냈다”고 판단하기 위해, 몇 개의 **복제본(replica)**이 메시지를 받아야 하는지 지정.
- 작동 과정 기준 이해:
```
01.프로듀서가 메시지를 브로커 리더(Leader)에게 보냄.
02.리더가 메시지를 자기 디스크에 기록하고, 동기화된 팔로워(Follower) 복제본으로 전송.

 → acks 값에 따라 프로듀서가 성공 응답(success) 받는 시점이 달라짐.
```
- 옵션별 의미:
  - **`acks=0`** → 프로듀서는 즉시 성공으로 간주 
    - 브로커가 메시지를 받았는지 확인 안 함
    - 속도 최우선, 손실 가능.

  - **`acks=1`** → 리더 자신의 디스크에 기록 완료 시 성공
    - 팔로워 아직 복제 안 됐어도 프로듀서는 OK
    - 속도와 일부 안정성 균형.

  - **`acks=all`** (또는 **`acks=-1`**) → 리더 + 모든 in-sync 팔로워가 기록 완료해야 성공.
    - 가장 안전, 브로커 장애 시에도 데이터 보존 가능, 지연 ↑
- 실제 메시지 손실 가능성을 acks 숫자가 결정한다고 보면 됨.
<br>  


### (3) `enable.idempotence=true`
- 의미: True, "같은 메시지를 여러 번 보내도 한 번만 저장해줘"
- 영향: 중복 메시지 완전 제거, 순서 보장
- 적용: 결제, 주문, 계좌 이체

**엔지니어 독백: ** "결제 시스템에서는 무조건 켜야 한다. 네트워크 장애로 재시도할 때 중복 결제 방지는 필수다."


<br>
<br> 



## 메시지 전송 시간제어
-Kafka 2.1 버전 기준(최신아님)

<details>

<summary>프로듀서 send() 동작 순서(Async기준)</summary>

### (1) 메시지 준비
```
record = ProducerRecord("topic-logs", key, value)
```
- 메시지 객체 생성
- 아직 브로커와는 상관 없음
### (2) send() 호출 - 버퍼에 넣기  
```
future = producer.send(record, callback=my_callback)  
```
- 메시지를 프로듀서 내부 배치 큐에 넣음  
- send() 반환 시점 = 메시지가 큐에 들어가고, 브로커 전송 준비가 끝난 순간(_브로커로 보낸거아님_)  
- 이때 호출-반환까지 스레드가 블록됨 (짧은 시간)  
```
🚀 버퍼와 배치의 차이  
01. 버퍼(Buffer)  
- send()가 호출되면 메시지가 프로듀서 내부 메모리 공간에 들어감  
- 버퍼는 그냥 대기 공간  

02. 배치(Batch)  
- 같은 파티션으로 갈 메시지들을 모아서 하나로 묶는 것
- 예를 들어: 파티션 A로 5개, 파티션 B로 3개 메시지가 있다면
                   파티션 A 배치 = 5개 묶음
                   파티션 B 배치 = 3개 묶음

#배치가 꽉 찼거나, linger.ms 시간이 지나야 실제로 브로커로 전송됨
```
### (3) 배치 전송 시작  
- 프로듀서 이벤트 루프가 배치 단위로 브로커 전송  
- 네트워크 I/O 시작  
### (4) 브로커 처리  
- 브로커가 메시지를 받고, 복제/ACK 처리   
- 성공/실패 결정  
- 동기 호출(synchronous)은 여기서 send() 반환 → send()호출부터~반환까지 블로킹됨  
### (5) 콜백 호출  
- 브로커 처리 완료 후 이벤트 루프가 콜백 실행  
- 이때 호출 스레드는 블록되지 않고 다른 일 수행 가능  

---
</details>

<br>
<br>

<img width="559" height="274" alt="image" src="https://github.com/user-attachments/assets/734e5f7d-bf7e-4b9e-b250-8001fc856f9a" />


### (4) `max.block.ms`
의미        :   "send() 호출했을 때 몇 초까지 기다릴지"
실제의미 :   "버퍼에 빈 공간이 생길 때까지 몇 초 기다릴지"
- 영향:
  - 설정값이 짧으면 BufferExhaustedException 빈발
  - 길면 API 응답 지연  
- Buffer에러 발생시 카프카에서 자동 재시도
```
producer = KafkaProducer(
    retries=3,                    # Kafka가 자동 재시도
    retry_backoff_ms=1000,
    delivery_timeout_ms=120000
)

# 이런 에러들은 Kafka가 알아서 재시도
- TimeoutException
- NotLeaderForPartitionError  
- LeaderNotAvailableError
- BrokerNotAvailableError

producer.send('topic', message)  # 실패 시 Kafka가 자동으로 3번 재시도
```
- 요구상황에 따른 설정값 범위
```
# 웹 API 서비스 (응답성 중요)
max_block_ms=1000~5000  # 1-5초

# 배치 처리 (안정성 중요)  
max_block_ms=30000~60000  # 30-60초

# 실시간 스트리밍
max_block_ms=100~1000  # 0.1-1초
```

**엔지니어 독백:** "웹 요청 처리할 때는 3초 넘으면 사용자가 짜증난다. 배치에서는 1분까지도 괜찮다."
 "에러 상황에 따라 버퍼메모리조정가능"

<br>  








### (5) `delivery.timeout.ms`
- 의미  :  메시지가 전송 → 브로커 기록 → 성공/실패 확인까지 최대 시간 (재시도 포함).
- 적용 기준: 브로커 장애 복구 시간 + 여유분

**엔지니어 독백:** "우리 클러스터는 브로커 재시작이 보통 30초 걸리니까 120초로 설정했다. 4배 여유분이면 충분하다."
<br>  



### (6) `request.timeout.ms `
 - 의미 : 브로커에게 "메타데이터 요청, 메시지 전송 요청" 등을 보낼 때 응답 대기 최대 시간 (개별 요청 기준).
 - 왜 이게 중요한가?
  → 네트워크가 느리거나 브로커가 바쁠 때, 무한정 기다리면 앱이 멈춰버림
 - 적용 기준: 네트워크 지연시간(ping) × 100
   - 실시간 10초 , 배치ETL 60
 - 실무 판단 기준

**엔지니어 독백:** "음... 우리 Kafka 클러스터가 AWS us-west-2에 있고, 앱도 같은 리전이니까... 보통 네트워크 RTT가 1-2ms 정도야. 근데 브로커가 바쁠 때나 GC 일어날 때를 고려하면... 15초 정도가 적당할 것 같은데?"
<br>


### (7) `retries and retry.backoff.ms`
- retries: 전송 실패 시 재시도 횟수
- retry.backoff.ms: 재시도 간격
- 실무 팁: 무한정으로 주고 사용안함. 횟수보다 시간으로 제어하는 게 직관적

**엔지니어 독백: ** "**`delivery_timeout_ms`**가 진짜 컨트롤러야. 2분 안에서 계속 재시도하다가 시간 끝나면 포기하는 거지. retries=무한으로 두고 시간으로만 제어하는 게 직관적이야."
```
# 권장하지 않음 (레거시)
producer = KafkaProducer(
    retries=3,
    retry_backoff_ms=1000
)

# 권장 방식
producer = KafkaProducer(
    delivery_timeout_ms=120000,  # 시간으로 제어
    retries=2147483647          # 사실상 무한
)
```
<br>




### (8) `linger.ms`
- 의미: 메시지를 모아서 배치 전송할 때 기다리는 시간
- 영향: 지연시간 vs 처리량 트레이드오프  

**엔지니어 독백1:** "10-100ms가 스위트스팟이다. 사용자는 못 느끼는데 처리량이 10배 늘어난다."

**엔지니어 독백2:** "linger_ms=0일 때 1000 TPS, linger_ms=10일 때 10000 TPS 나왔어. 10ms 지연으로 10배 처리량... 이게 바로 스위트스팟이야."

**엔지니어 독백3:** "우리 서비스가 초당 5만 개 이벤트 발생하면, 최소 50,000 TPS는 처리해야 해. linger_ms 튜닝이 바로 이 TPS를 올리는 핵심이야."
**TPS = Transactions Per Second (초당 트랜잭션 수)**
```
#높을수록 좋은성능
저사양: 1,000 TPS
일반적: 10,000 TPS
고성능: 100,000 TPS
```
```
# 1. 실시간 시스템 (결제, 알림)
linger_ms = 0        # 즉시 전송

# 2. 준실시간 (사용자 액션)  
linger_ms = 5        # 5ms 대기

# 3. 일반 로깅/이벤트
linger_ms = 10       # 10ms 대기 (스위트스팟), 99% 상황에서 최적

# 4. 배치 ETL
linger_ms = 100      # 100ms 대기
```
<br>



### (9) buffer.memory
- 의미     :  프로듀서내부에서 브로커로 전송 대기중인 메시지가 사용하는 메모리 총량
- 설정값  :  초당 메시지 수(TPS) × 메시지 크기 × 예상 지연시간(초) x (버퍼링계수)
  - 실무 팁 : 예상치의 2-3배로 설정 
  - 일반서비스(64MB) : 1,000 × 500 × 0.1 × 3 = 150,000 bytes = 0.14MB
  - 실시간 스트리밍(128MB) : 100,000 × 1,024 × 0.05 × 3 = 15,360,000 bytes = 14.6MB
  - 고부하서비스(256MB) : (tps 50000 tr/s) x (message size 2KB) x ( latency 0.5ms ) x 3 = 150MB ?
 - AWS 인스턴스 : "EC2 인스턴스 메모리의 1-5% 정도를(1.6%) Kafka buffer로 할당하는 게 일반적이야."

**엔지니어 독백:** "네트워크가 느리거나 브로커가 바쁘면 메시지들이 여기서 대기하게 돼. 이 공간이 가득 차면 send() 호출이 블록되거나 에러 발생해."
<br>



### (10) compression.type
- 의미 : 메시지 압축 방식 (none, snappy, gzip, lz4, zstd)
- 고려사항 :   CPU vs 네트워크 비용 → 네트워크 사용량 ↓, 전송 속도 ↑, CPU 사용량 고려
```
네트워크가 병목
→ gzip 또는 zstd 선택 (전송량 최소화)

CPU가 여유롭고 초저지연 중요
→ lz4 (실시간 로그 처리, IoT 이벤트)

균형형, 무난한 선택
→ snappy (CPU/네트워크 모두 중간 수준)

CPU도 부족하고, 압축 이득이 적은 데이터
→ none (소규모 메시지, 초고속 필요 시)
```
- 예시
  - 같은 데이터센터 내 처리 → lz4 (빠른 처리)
  - 리전 간 전송 (서울 ↔ 버지니아) → gzip이나 zstd (비용 절감)

**엔지니어 독백:** "Uber는 실시간 위치 데이터라 lz4 쓰고, Netflix는 로그 분석용이라 gzip 써. 우리도 용도에 맞게 선택해야지."
<br>






### (11) batch.size
- 의미 : 한 배치에 담을 메시지 최대 크기(Byte)
- 영향 : 크면 처리량 증가, 메모리 더 사용,  배치 사용 시 전송 효율 ↑, 네트워크 부하 ↓, latency 약간 ↑
- 적용 기준: 평균 메시지 크기의 10-20배
- 실무 팁: 배치가 안 차도 linger.ms 후 전송되니 지연 걱정 없음
- batch_size vs linger_ms 우선순위 → OR조건, 둘 중 하나라도 조건 만족하면 전송
  - 실시간 서비스 로그/이벤트 → 시간 위주 튜닝
  - 데이터 적재/분석 파이프라인 → 배치 위주 튜닝

**엔지니어 독백 : ** "실제로는 시간(linger_ms)이 먼저 발동하는 경우가 90% 이상이야. 배치가 가득 차기 전에 시간이 다 되는 거지. 그래서 배치 크기는 '최대 허용 크기'라고 보는 게 맞아."

<details>
<summary>예시상황</summary>

-상황: 실시간 사용자 이벤트 (1초에 100개, 각 메시지 200byte)
```
pythonproducer = KafkaProducer(
    batch_size=16384,    # 16KB = 대략 80개 메시지(여유있게)
    linger_ms=50         # 50ms 대기
)
```
"100개 메시지가 1초에 들어오니까... 500ms에 50개, 800ms에 80개가 쌓이겠네. 80개째에서 batch_size 조건 만족해서 전송될 거야. linger_ms=50ms는 이 경우엔 의미없고."
"만약 트래픽이 적어서 50ms에 10개만 쌓였다면? linger_ms=50이 먼저 발동해서 10개만 보내게 되지."
</details>
<br>



### (12) max.in.flight.requests.per.connection
- 의미 
  - 프로듀서 → 브로커로 보낸 요청(배치)이 응답을 기다리지 않고 동시에 outstanding 상태로 몇 개까지 있을 수 있는지 설정
  - 브로커 ACK 오기 전에도 몇 개나 더 보낼 수 있느냐를 정하는 값
```
# 예시 시나리오
<값이 1일 때>
  - 프로듀서가 배치1 보냄
  - 브로커 응답(ACK) 오기 전에는 배치2를 못 보냄
  - 항상 순서 보장 (배치1 → 배치2 → 배치3)
  - 하지만 처리량은 낮음
  - 실무 사용 예: 정확한 순서 보장 필요 (금융 거래 로그, 주문 이벤트)

<값이 5일 때>
  - 프로듀서가 배치1, 배치2, …, 배치5를 브로커에 동시에 전송
  - 브로커에서 응답이 오는 순서가 뒤섞일 수 있음
  - 예: 배치3이 ACK 먼저 오고, 배치2가 나중에 ACK
  - 재시도(retry) 상황이 생기면 순서가 바뀔 수 있음
  - 실무 사용 예: 순서가 조금 바뀌어도 괜찮은 로그, 모니터링 이벤트, 메트릭 전송 / 고처리량 최적화
```
- 실무 팁
  - 기본값은 5 (KafkaProducer) → 처리량 우선
  - 순서 보장 필요하면 1로 설정 →브로커 응답을 기다려야 다음 배치를 보낼 수 있음 → TPS는 떨어짐 (병렬성 감소)
  - retries > 0 이면서 max.in.flight.requests.per.connection > 1일 경우 순서 꼬임 가능
<details>
<summary>문제상황 예시 </summary>

### 배경
- max.in.flight.requests.per.connection > 1
  - 프로듀서는 브로커 ACK를 기다리지 않고 동시에 여러 배치를 전송 가능
  - 즉, 배치1, 배치2, 배치3을 동시에 날림

- retries > 0
  - 브로커나 네트워크 오류가 발생하면 실패한 배치만 재전송

### 문제 상황
[1] 배치1, 배치2, 배치3이 동시에 전송됨 (in-flight)
[2] 배치1 전송 실패 → 재시도
[3] 배치2, 배치3은 이미 성공 처리됨
[4] 배치1 재전송이 성공하면, 원래 순서보다 뒤에 들어가게 됨
즉, 재시도로 인해 순서가 뒤바뀌면서 순차적 순서 보장이 깨지는 것
후처리 - 보통은 그대로 사용, 필요시 분석할때 재정렬해서 사용
### 요약
- 순서 꼬임 가능 조건
  - 재시도를 허용 (**`retries > 0`**)
  - 동시에 여러 요청 가능 (**`max.in.flight.requests.per.connection > 1`**)
- 순서 보장을 확실히 하고 싶으면
  - **`max.in.flight.requests.per.connection = 1`**
  - 재시도 있어도 항상 하나씩 보내므로 순서 유지
</details>

- 정리
  - 1 → 순서 보장, 처리량 낮음
  - 1보다 큰 값 → 처리량 높음, 순서 꼬일 수 있음



### (14) `max.request.size`
- 의미       :  한 요청에 담을 수 있는 최대 메시지 크기.
- 고려사항 :  브로커의 설정값인 message.max.bytes와 맞춰야 거부되지 않음.
- 실무 팁    :   큰 파일은 S3에 저장하고 URL만 전송 ??
  - 큰 메시지(예: 수 MB 이상)**를 그대로 Kafka로 보내면 RecordTooLargeException 발생  
  - 메시지 자체는 S3 같은 외부 스토리지에 저장하고, Kafka에는 그 메시지를 참조하는 경로(URL, key 등)만 전송
  - Consumer는 메시지를 받아서 url을 읽고, S3에서 실제 이미지 다운로드 후 처리
- 설정값     : batch_size × 파티션 수 × 3배, 
  - 일반적인 웹서비스(2MB), ETL분석(16MB) 

**엔지니어 독백 :** "batch_size는 하나의 파티션에 대한 배치고, max_request_size는 **_모든 파티션 배치들을 합친 전체 요청 크기_**야. 보통 batch_size × 파티션 수 정도로 계산해."



<details>
<summary>RequestTooLargeException에러</summary>

### 원문 에러
```
org.apache.kafka.common.errors.RecordTooLargeException: 
The message is 2097152 bytes when serialized which is larger than 1048576, 
which is the value of the max.request.size configuration.
```
### 해석
- 예외 타입: RecordTooLargeException → 전송하려는 메시지가 너무 큼
- 메시지 크기: 2097152 bytes (약 2MB)
- 설정 제한: max.request.size = 1048576 bytes (약 1MB)
- 의미: 직렬화된 메시지가 설정한 최대 요청 크기보다 커서 전송 불가
즉, Kafka 프로듀서가 한 번에 브로커로 보낼 수 있는 메시지 크기보다 메시지가 커서 에러 발생한 상황입니다.

### 실무 해결 방법
- 메시지 크기 늘리기
  - 프로듀서 max.request.size 값을 메시지보다 조금 크게 설정
  - 브로커 설정도 확인 필요: message.max.bytes
```
producer = KafkaProducer(
    max_request_size=3 * 1024 * 1024  # 3MB
)
```
- 메시지 분할
  - 2MB 메시지를 작은 단위로 쪼개서 여러 번 전송
  - 예: 500KB씩 쪼개서 순서대로 보내고 Consumer에서 합치기
- 압축 활용
  - compression.type='gzip' 등으로 메시지 직렬화 후 크기 감소

**엔지니어 독백 : ** "에러 터지면 일단 max_request_size 늘려서 급한 불 끄고, 근본적으로는 메시지 구조를 바꿔야 해. 큰 데이터는 Kafka 밖에서 처리하는 게 정답이야."
</details>
<br>



### (15) `receive.buffer.bytes` , `send.buffer.bytes`
- 정의
  - receive.buffer.bytes
    - 브로커/컨슈머가 수신할 때 사용하는 소켓 버퍼 크기
    - 단위: 바이트
    - Kafka가 브로커로부터 데이터를 읽어올 때 TCP 레벨에서 한 번에 받을 수 있는 최대 크기를 의미
  - send.buffer.bytes
    - 프로듀서/브로커가 전송할 때 사용하는 소켓 버퍼 크기
    - 단위: 바이트
    - 한 번에 네트워크로 보낼 수 있는 데이터 양의 최대값
- 버퍼 개념 구분
  - 프로듀서 내부 버퍼 (`buffer.memory`) → JVM 메모리 안에서 메시지를 쌓아두는 공간
  - 네트워크 소켓 버퍼 (`send.buffer.bytes` , `receive.buffer.bytes`) → OS TCP 레벨에서 메시지를 전송하거나 받을 때 사용하는 메모리

**엔지니어 독백:** "이건 Kafka 레벨이 아니라 OS 레벨 설정이야. 네트워크가 느리거나 지연시간이 클 때 TCP 윈도우 크기를 조절해서 처리량을 높이는 거지."
- 설정방법  
  - 기본값 
    - 기본: 32KB~64KB 정도
    - 대부분 소규모 메시지/저TPS 환경에서는 기본값으로 충분
      ```
          'send_buffer_bytes': -1,        # OS 기본값 사용 (보통 64KB)
          'receive_buffer_bytes': -1,     # OS 기본값 사용 (보통 64KB)
      ```
  - 같은리전 (RTT 1-10ms)
    ```
    # AWS us-west-2 내부
    SAME_REGION_CONFIG = {
        'send_buffer_bytes': 131072,    # 128KB
        'receive_buffer_bytes': 65536,  # 64KB
        'reason': '적당한 지연시간, 약간의 버퍼링',
        'use_case': 'AWS 같은 AZ 간 통신'
    }
    ```
  - 크로스 리전 (RTT 50-100ms)
    ```
    # us-west-2 → us-east-1
    CROSS_REGION_CONFIG = {
        'send_buffer_bytes': 524288,    # 512KB
        'receive_buffer_bytes': 262144, # 256KB
        'reason': '높은 지연시간, 큰 윈도우 필요',
        'use_case': '미국 동서부 간 데이터 복제'
    }
    ```
<details>
<summary>RTT 개념</summary>

### RTT는(Round-Trip Time)
- 데이터 패킷이 출발지에서 목적지까지 갔다가 다시 돌아오는 데 걸리는 시간
- ping 결과값임

- 설명
  -단위: 보통 밀리초(ms)

- 예시
  - 로컬 데이터센터: RTT < 1ms → 거의 지연 없음
  - 같은 리전(AZ 간): RTT 1~10ms → 약간의 지연
  - 크로스 리전(미국 동서부): RTT 50~100ms → 눈에 띄는 지연
  - 글로벌/위성 통신: RTT 200ms+ → 매우 긴 지연
</details>

<br>

### Summary
<details>
<summary>설정값 항목 </summary>

1️⃣ client.id  

2️⃣ acks (메시지 기록 확인 기준)

3️⃣ 메시지 전송 시간 관련

  max.block.ms: send() 호출 시 버퍼 꽉 차면 최대 대기 시간.
  
  delivery.timeout.ms: 메시지가 전송 → 브로커 기록 → 성공/실패 확인까지 최대 시간 (재시도 포함).
  
  request.timeout.ms: 브로커 응답 대기 최대 시간 (개별 요청 기준).
  
  💡 팁: Kafka 2.1 이후엔 delivery.timeout.ms 중심으로 재시도/타임아웃 설정.



4️⃣ 재시도 관련

  retries: 전송 실패 시 재시도 횟수.
  
  retry.backoff.ms: 재시도 간 대기 시간.
  
  ⚠️ 직접 재시도 로직 만들 필요 없음. Kafka 클러스터 복구 시간에 맞춰 delivery.timeout.ms만 조절.



5️⃣ 배치 및 대기

  linger.ms: 메시지를 모아서 배치 전송할 때 기다리는 시간.
  
  batch.size: 한 배치에 담을 메시지 최대 크기(Byte).
  
  → 배치 사용 시 전송 효율 ↑, 네트워크 부하 ↓, latency 약간 ↑



6️⃣ 메모리

  buffer.memory: 프로듀서가 전송 대기 메시지 위해 쓰는 메모리 총량.
  
  max.in.flight.requests.per.connection: 응답 대기 없이 전송 가능한 배치 수 → 순서 보장/성능에 영향.


7️⃣ 압축

  compression.type: 메시지 압축 방식 (none, snappy, gzip, lz4, zstd).
  
  → 네트워크 사용량 ↓, 전송 속도 ↑, CPU 사용량 고려.



8️⃣ 메시지 크기

  max.request.size: 한 요청에 담을 수 있는 최대 메시지 크기.
  
  → 브로커 message.max.bytes와 맞춰야 거부되지 않음.



9️⃣ Idempotent (중복 방지)

  enable.idempotence=true → 메시지 중복 방지.
 
  원리: 메시지마다 시퀀스 번호 부여, 이미 받은 메시지는 브로커가 무시 → DuplicateSequenceException 발생.
  
  조건: acks=all, retries>0, max.in.flight.requests.per.connection ≤ 5
  
  장점: 순서 보장 + 중복 없는 안전한 전송.
</details>
<br><br> 
