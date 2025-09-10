# Kafka Exactly-Once Semantics 요약 (Chapter 8)

## 1. 개요

- 데이터 처리의 **'정확히 한 번(Exactly-Once)'** 보장을 Kafka에서 구현하는 방법을 다룸
- 데이터 스트림 집계(Aggregation), 상태 업데이트 등 중복 처리가 비즈니스 로직에 치명적 오류를 발생시키는 경우 필수적
- **'멱등성 프로듀서(Idempotent Producer)'**와 **'트랜잭션(Transactions)'** 두 가지 핵심 기능으로 달성

## 2. 역할별 핵심 요약

### 프로듀서 (Producer)

- **멱등성**: `enable.idempotence=true` 설정으로 네트워크 오류로 인한 자체 재시도 시 메시지 중복 방지
  - 메시지에 부여된 Producer ID(PID)와 Sequence Number를 브로커가 확인하여 중복 제거
- **트랜잭션**: `transactional.id` 설정으로 여러 토픽/파티션에 걸친 쓰기 작업과 오프셋 커밋을 하나의 원자적(atomic) 단위로 처리
  - `beginTransaction()`, `commitTransaction()` API 등으로 생명주기 관리
  - **'펜싱(fencing)'** 메커니즘으로 '좀비 인스턴스'의 데이터 처리 방지

### 브로커 (Broker)

- **중복 추적**: 프로듀서가 보낸 메시지의 (PID, Sequence Number)를 추적하여 중복 요청 거부
- **트랜잭션 관리**: '트랜잭션 코디네이터'가 트랜잭션의 시작, 커밋, 중단 등 모든 상태를 트랜잭션 로그(`__transaction_state` 토픽)에 기록 및 관리
- **데이터 격리**: 커밋된 트랜잭션의 데이터만 컨슈머에게 노출되도록 LSO(Last Stable Offset)를 관리하여 데이터 가시성 제어

### 컨슈머 (Consumer)

- **격리 수준 설정**: `isolation.level` 옵션으로 데이터 읽기 방식 결정
  - `read_uncommitted` (기본값): 커밋 여부와 상관없이 모든 메시지 읽기
  - `read_committed`: 성공적으로 커밋된 트랜잭션의 메시지만 읽기 (데이터 정합성 보장)
- **오프셋 커밋 방식**: 트랜잭션 사용 시 `commitSync`/`Async`를 직접 호출하지 않음
  - 프로듀서가 `sendOffsetsToTransaction()` API를 통해 트랜잭션의 일부로 오프셋을 커밋

---

## 상세 요약

### 1. 멱등성 프로듀서 (Idempotent Producer)

- **개념**
  - 동일한 작업을 여러 번 수행해도 결과가 한 번 수행한 것과 같은 **'멱등성'** 보장
  - 네트워크 오류로 인한 프로듀서의 내부 재시도 시 발생하는 메시지 중복 방지
- **동작 원리**
  - 프로듀서에 고유 ID (PID) 할당, 메시지별로 단조 증가하는 Sequence Number 부여
  - 브로커는 (PID, 파티션, 시퀀스 번호) 조합을 추적하여 중복 식별 및 거부
- **제한 사항**
  - 프로듀서의 내부 재시도로 인한 중복만 방지
  - 애플리케이션이 `producer.send()`를 두 번 호출하는 등 애플리케이션 레벨의 중복은 방지 불가
- **사용법**
  - 프로듀서 설정에 `enable.idempotence=true` 추가
  - 활성화 시 `acks=all` 자동 설정, 메시지 순서 보장

### 2. 트랜잭션 (Transactions)

- **개념**
  - 'consume-process-produce' 패턴의 데이터 처리 정확성 보장
  - 여러 파티션에 걸친 쓰기 작업과 컨슈머 오프셋 커밋을 하나의 원자적 단위로 묶어 **'전부 성공'** 또는 **'전부 실패'** 보장
- **해결하는 문제**
  - **애플리케이션 충돌**: 결과는 전송했으나 오프셋 커밋 전 다운된 경우, 재시작 시 중복 처리되는 문제 방지
  - **좀비 인스턴스**: 이전 인스턴스가 잠시 멈췄다 깨어나 이미 처리된 데이터를 중복 처리하는 문제 방지
- **동작 원리**
  - **좀비 펜싱**: `transactional.id`와 내부적으로 증가하는 'epoch' 번호를 조합하여 구버전 프로듀서(좀비)의 요청을 차단
  - **원자적 쓰기**: `beginTransaction()` → `send()` → `sendOffsetsToTransaction()` → `commitTransaction()` 순서로 진행, 실패 시 `abortTransaction()`으로 롤백
  - **컨슈머 격리**: `isolation.level=read_committed` 설정 필수
- **한계점**
  - Kafka 내부의 쓰기 작업에만 트랜잭션이 적용됨
  - 외부 시스템(DB, REST API, 이메일 발송 등) 연동 작업은 롤백되지 않음 (Outbox 패턴 등 별도 아키텍처 필요)
- **사용법: Producer/Consumer API**
  - **프로듀서 설정**
    - `transactional.id=고유ID` (필수, 재시작 시에도 유지되어야 함)
  - **컨슈머 설정**
    - `enable.auto.commit=false`
    - `isolation.level=read_committed`
  - **애플리케이션 로직**
    - `producer.initTransactions()`: 트랜잭션 초기화
    - `while (true)` 루프 내에서
      - `producer.beginTransaction()`: 트랜잭션 시작
      - 메시지 처리 (process) 및 결과 전송 (`producer.send`)
      - 오프셋 정보 전송 (`producer.sendOffsetsToTransaction`)
      - `producer.commitTransaction()`: 트랜잭션 커밋
    - `catch (Exception e)` 블록에서 `producer.abortTransaction()`: 예외 발생 시 롤백

### 3. 트랜잭션 ID와 펜싱 메커니즘

- **`transactional.id` 중요성**
  - 애플리케이션 인스턴스별로 고유해야 하며, 재시작해도 유지되어야 함
  - 잘못 할당 시 좀비 펜싱이 정상 동작하지 않아 데이터 정합성 문제 발생 가능
- **펜싱 메커니즘 발전 (Kafka 2.5+)**
  - **과거**: `transactional.id`와 파티션을 정적으로 매핑해야 하는 비효율적 구조
  - **현재**: 컨슈머 그룹 메타데이터를 트랜잭션에 포함시켜, 동적 파티션 할당(rebalance) 환경에서도 안전하게 트랜잭션 사용 가능

### 4. 트랜잭션 성능

- **프로듀서 오버헤드**
  - 트랜잭션 관련 API 호출로 인해 약간의 지연 시간(latency) 증가
  - 트랜잭션 당 메시지 수(배치 크기)를 늘리면 처리량(throughput)은 향상됨
- **컨슈머 영향**
  - `read_committed` 모드는 트랜잭션이 완료될 때까지 데이터 소비가 지연될 수 있음 (end-to-end latency 증가)
- **핵심**: Exactly-Once는 성능(특히 지연 시간)과 데이터 정합성을 맞바꾸는 트레이드오프 관계
  - 비즈니스 요구사항에 맞춰 배치 크기를 조절하는 튜닝이 필수적임
<br><br>



---
<br><br>



실습

## 1. 트랜잭션을 위한 환경 준비
### 브로커(서버) 측
- 특별히 명령어를 실행할 필요 없음. Kafka 0.11 이상이면 트랜잭션 지원됨.
- 내부적으로 트랜잭션 상태를 저장할 __transaction_state 토픽이 자동 생성됨.
- 다만 브로커 설정 예시:

```
# server.properties
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2
```
- `transaction.state.log.replication.factor` : 트랜잭션 상태 로그 복제 수
- `transaction.state.log.min.isr` : 트랜잭션 상태 로그 최소 ISR
- 의미: 브로커가 트랜잭션 상태를 안정적으로 저장하고, 장애 발생 시 복구 가능하게 설정

### 컨슈머 측
- isolation.level 설정 필요
```
# consumer.properties
isolation.level=read_committed
enable.auto.commit=false
```
- `isolation.level=read_committed` : 커밋된 트랜잭션 메시지만 읽도록 설정
- `enable.auto.commit=false` : 오프셋 커밋을 트랜잭션과 연계하도록 설정




## 2. 프로듀서 측 트랜잭션 설정
<br>

### 2-1. 프로듀서 생성 시 설정
- 프로듀서 코드/컨테이너에서 설정
```
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

// 트랜잭션 관련 설정
props.put("enable.idempotence", "true");                // 멱등성 활성화
props.put("transactional.id", "my-unique-transactional-id");  // 트랜잭션 ID 설정
```

의미:
- enable.idempotence=true : 프로듀서 재시도 시 중복 전송 방지
- transactional.id : 트랜잭션을 구분하는 고유 ID, 재시작 후에도 동일 ID를 사용해야 함
<br>

### 2-2. 트랜잭션 API 호출 순서
```
// 1. 트랜잭션 초기화
producer.initTransactions();  
// 브로커와 통신하여 해당 transactional.id 초기화, epoch 번호 관리 시작

try {
    // 2. 트랜잭션 시작
    producer.beginTransaction();
    // 이 시점부터 send되는 메시지는 트랜잭션에 묶임

    // 3. 메시지 전송
    producer.send(new ProducerRecord<>("my-topic", key, value));

    // 4. 컨슈머 오프셋 트랜잭션에 포함
    Map<TopicPartition, OffsetAndMetadata> offsets = ...
    producer.sendOffsetsToTransaction(offsets, consumerGroupId);

    // 5. 트랜잭션 커밋
    producer.commitTransaction();  
    // 성공하면 메시지와 오프셋 모두 커밋, 컨슈머가 읽을 수 있음

} catch (Exception e) {
    // 실패 시 트랜잭션 롤백
    producer.abortTransaction();
    // 메시지와 오프셋이 모두 롤백
}
```

**의미:** 
- `initTransactions()` : 트랜잭션 환경 초기화, 브로커와 epoch 번호 동기화
- `beginTransaction()` : 새 트랜잭션 시작
- `send()` : 메시지를 트랜잭션에 포함
- `sendOffsetsToTransaction()` : 읽은 컨슈머 오프셋까지 트랜잭션에 포함 → 중복 재처리 방지
- `commitTransaction()` : 성공 시 메시지+오프셋 동시 확정
- `abortTransaction()` : 실패 시 메시지+오프셋 동시 롤백
<br>

## 3. 실무 적용 위치
- 프로듀서 컨테이너/애플리케이션에서 코드 작성
- 컨슈머 컨테이너/애플리케이션에서 isolation.level 설정
- 브로커는 설정 파일(server.properties)만 확인, 트랜잭션 상태와 펜싱은 자동 처리
