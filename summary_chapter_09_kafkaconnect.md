# 1️⃣ Kafka Connect 실행 환경 준비 (실무 기준, S3 Sink 준비 포함)

## 전체 흐름
Kafka Connect 분산 모드로 S3 Sink Connector를 실행하기 위한 기본 환경을 구축합니다. 이 과정은 다음 단계(S3 Sink Connector 등록 및 실행)의 기반이 됩니다.

## 1) Kafka Connect 바이너리 설치

### Confluent Platform 기준

```bash
# 다운로드 및 압축 해제
wget https://packages.confluent.io/archive/7.5/confluent-community-7.5.0.tar.gz
tar -xzf confluent-community-7.5.0.tar.gz
cd confluent-7.5.0
```

**명령어 설명**: Confluent Community 버전을 다운로드하고 압축 해제하여 Kafka Connect 실행 환경을 준비

### 주요 파일 위치

| 파일/폴더 | 역할 |
|-----------|------|
| `bin/connect-distributed` | 분산 모드 실행 스크립트 |
| `bin/connect-standalone` | 단일 모드 실행 스크립트 (테스트용) |
| `config/connect-distributed.properties` | 분산 모드 환경 설정 파일 |
| `config/connect-standalone.properties` | 단일 모드 환경 설정 파일 |

## 2) S3 Sink Connector 설치

### Confluent Hub에서 설치

```bash
confluent-hub install confluentinc/kafka-connect-s3:latest
```

**명령어 설명**: Confluent Hub에서 공식 S3 Sink Connector를 다운로드하고 설치

### 설치 경로

```
/usr/share/confluent-hub-components/confluentinc-kafka-connect-s3/lib/
```

**중요**: 이 위치를 **plugin.path**에 등록해야 Connect가 읽을 수 있음

> **엔지니어 독백**: "plugin.path에 안 넣으면 'ClassNotFoundException' 뜨면서 Connector가 실행 안 된다. 여러 커넥터가 있다면 쉼표로 경로 나열 가능."

## 3) plugin.path 설정

Connect가 참조하는 커넥터 위치 지정

**예시 (connect-distributed.properties 내):**

```properties
plugin.path=/usr/share/java,/usr/share/confluent-hub-components/confluentinc-kafka-connect-s3/lib
```

**설명**: 기본 Java 라이브러리와 S3 Connector 경로를 함께 등록

## 4) Connect 환경 설정 파일 예시 (config/connect-distributed.properties)

```properties
# Kafka 브로커 연결
bootstrap.servers=broker1:9092,broker2:9092

# 플러그인 경로
plugin.path=/usr/share/java,/usr/share/confluent-hub-components/confluentinc-kafka-connect-s3/lib

# REST API 포트
rest.port=8083

# 상태/오프셋/설정 저장 토픽
offset.storage.topic=connect-offsets
config.storage.topic=connect-configs
status.storage.topic=connect-status

# 로그 및 컨버터 설정
key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
internal.key.converter=org.apache.kafka.connect.json.JsonConverter
internal.value.converter=org.apache.kafka.connect.json.JsonConverter

# 병렬 처리 수
tasks.max=10
```

### 커스터마이징 포인트

- `plugin.path`: S3 Sink Connector 경로 추가
- `tasks.max`: 병렬 처리 수 조절 (데이터 처리량에 따라 조정)
- `rest.port`: REST API 접근 포트 (기본 8083)

## ✅ 확인 포인트 (1단계 완료 조건)

1. **Kafka Connect 바이너리 설치 및 `bin` 스크립트 확인** (`connect-distributed`, `connect-standalone`)
2. **S3 Sink Connector JAR 설치 확인** (`/usr/share/confluent-hub-components/.../lib`)
3. **`plugin.path`에 S3 Connector 경로 등록**
4. **`connect-distributed.properties` 환경 설정 완료**

> **엔지니어 독백**: "이 단계에서 환경이 완전히 준비되어야 이후 단계(S3 Sink Connector 등록, 실행)가 문제 없이 진행된다. 파일 위치, plugin.path, Kafka 브로커 주소까지 꼭 확인."

# 2️⃣ Kafka Connect 실행 (분산 모드 기준)

## 전체 흐름
환경 설정이 완료된 Kafka Connect를 실제로 실행하여 REST API를 통해 Connector를 관리할 수 있는 상태로 만듭니다. 이 단계 완료 후 S3 Sink Connector 등록이 가능해집니다.

## 1) 실행 방법

### (1) 바이너리로 실행

```bash
# 분산 모드
cd /path/to/confluent-7.5.0
bin/connect-distributed config/connect-distributed.properties
```

**설명**: 
- Connect 프로세스가 백그라운드에서 실행
- REST API(기본 8083 포트) 통해 Connector 등록/제어 가능

### (2) Docker 실행 (테스트/빠른 배포용)

```bash
docker run -d --name connect \
  -p 8083:8083 \
  -e CONNECT_BOOTSTRAP_SERVERS=broker1:9092 \
  -e CONNECT_REST_ADVERTISED_HOST_NAME=connect \
  -e CONNECT_PLUGIN_PATH=/usr/share/java,/usr/share/confluent-hub-components/confluentinc-kafka-connect-s3/lib \
  confluentinc/cp-kafka-connect:latest
```

**명령어 설명**: Docker 컨테이너로 Connect를 실행하며, 환경변수로 주요 설정을 전달

## 2) 확인 포인트

### 1. 프로세스 확인

```bash
ps aux | grep connect
```

**명령어 설명**: Connect 프로세스가 정상적으로 실행 중인지 확인

### 2. REST API 확인

```bash
curl http://localhost:8083/
# {"version":"x.x.x","commit":"xxxx"} 나오면 정상
```

**명령어 설명**: REST API가 응답하는지 확인하여 Connect 서비스 정상 동작 검증

### 3. 로그 확인

```bash
tail -f logs/connect.log
# 플러그인 로드 성공, Kafka 브로커 연결 확인
```

**명령어 설명**: 실시간으로 로그를 모니터링하여 시작 과정에서 발생하는 이슈를 즉시 파악

> **엔지니어 독백**: "기동 후 로그에 Plugin load, Kafka connection 메시지가 나오면 성공. 여기서 막히면 plugin.path, Kafka 브로커 주소, 포트 확인 필수."

## 3) 커스터마이징 포인트

- **REST 포트 변경** (`rest.port`) - 기본 8083에서 다른 포트로 변경 가능
- **로그 경로, 로그 레벨 변경** - 디버깅 및 운영 환경에 맞춤
- **JVM 옵션 추가** (메모리, GC 튜닝) - 대용량 데이터 처리 시 성능 최적화
- **분산 모드 토픽 설정** (offset/config/status) - 클러스터 환경에서 상태 관리

## ✅ 2단계 완료 조건

1. **Connect 프로세스가 정상 기동**
2. **REST API 응답 확인**
3. **플러그인(S3 Sink) 로드 확인**
4. **Kafka 브로커 연결 정상**

# 🎯 실무 시나리오: 고객 로그 → S3 저장

## 목표
- 웹/앱에서 발생하는 고객 로그를 실시간으로 수집
- Kafka Connect S3 Sink Connector로 바로 S3 버킷에 저장
- JSON 포맷, IAM Role 인증, 파일 사이즈/폴더 구조 커스터마이징

---

# 3️⃣ 환경 준비 & 커스텀 포인트

## 전체 흐름
실무에서 고객 로그를 S3에 저장하기 위한 전체 환경을 준비합니다. AWS 권한 설정부터 Kafka Topic 준비까지 실제 운영 환경에서 필요한 모든 요소를 준비합니다.

## 작업 내용

### 1. Kafka Connect 실행 환경 준비
- Docker or 바이너리 설치
- S3 Sink Connector JAR 다운로드 (Confluent Hub) → `plugin.path`에 배치

### 2. AWS S3 버킷 준비
- IAM Role 부여: S3 PutObject 권한
- 버킷 이름/폴더 구조 설계 (`s3://customer-logs/yyyy/mm/dd/`)

### 3. 로컬 테스트용 Kafka Topic 준비
- 로그 임시 저장용 (실제 운영에서는 Connect Source 없이 수집 가능)

> **엔지니어 독백**: "먼저 Connect 환경과 플러그인, S3 권한 확보가 필수. IAM Role을 사용하면 키 관리 편하고, S3 버킷 구조는 나중에 쿼리/ETL 작업 편하게 설계."

---

# 4️⃣ Kafka Connect 환경 설정

## 커스텀 포인트
- `bootstrap.servers` → Kafka 브로커 연결
- `plugin.path` → S3 Sink Connector 위치 지정
- REST 포트, 로그 경로, JVM 옵션 설정

## 실무 예시

```properties
bootstrap.servers=broker1:9092,broker2:9092
plugin.path=/usr/share/java
rest.port=8083
# distributed 모드인 경우 offset/config/status 토픽 추가
```

> **엔지니어 독백**: "환경이 틀리면 Connector가 아예 안 띄워진다. 분산 모드라면 offset/config/status 토픽도 반드시 설정."

---

# 5️⃣ S3 Sink Connector 설정 작성

## 커스텀 포인트
- 저장할 Kafka Topic 지정 (실제 로그가 Topic에 들어오거나, Source 없이 Direct Push 가능)
- S3 버킷 이름, region, 포맷(JSON)
- `flush.size` → 몇 개 로그마다 S3에 파일 저장
- `rotate.interval.ms` → 시간 단위 파일 분할
- `partitioner.class` → S3 폴더 구조 커스터마이징
- 오류 처리 정책 (`errors.tolerance`, DLQ)

## 실무 JSON 예시

```json
{
  "name": "customer-logs-s3-sink",
  "config": {
    "connector.class": "io.confluent.connect.s3.S3SinkConnector",
    "tasks.max": "3",
    "topics": "customer-logs",
    "s3.bucket.name": "customer-logs",
    "s3.region": "ap-northeast-2",
    "storage.class": "io.confluent.connect.s3.storage.S3Storage",
    "format.class": "io.confluent.connect.s3.format.json.JsonFormat",
    "flush.size": "500",
    "rotate.interval.ms": "300000",
    "partitioner.class": "io.confluent.connect.storage.partitioner.TimeBasedPartitioner",
    "path.format": "'year'=YYYY/'month'=MM/'day'=dd",
    "locale": "en",
    "timezone": "Asia/Seoul",
    "errors.tolerance": "all",
    "errors.deadletterqueue.topic.name": "dlq-customer-logs",
    "errors.deadletterqueue.context.headers.enable": "true"
  }
}
```

**주요 설정 설명**:
- `flush.size`: 500개 로그마다 S3 파일 저장
- `rotate.interval.ms`: 5분(300000ms)마다 파일 분할
- `path.format`: S3에 `year=2025/month=09/day=24` 구조로 저장
- `errors.tolerance`: 모든 에러 허용하고 DLQ로 전송

> **엔지니어 독백**: "flush.size, rotate.interval, partitioner 조합이 파일 사이즈와 S3 요청 수를 결정한다. 너무 작으면 S3 PUT 폭주, 너무 크면 실시간 분석 지연. DLQ는 필수."

---

# 6️⃣ Connector 등록

## 작업 내용
REST API로 S3 Sink Connector 등록

```bash
curl -X POST -H "Content-Type: application/json" \
     --data @customer-logs-s3.json \
     http://localhost:8083/connectors
```

**명령어 설명**: JSON 설정 파일을 사용해 S3 Sink Connector를 Connect 클러스터에 등록

> **엔지니어 독백**: "등록 후 바로 상태 확인. 실패하면 IAM Role, Topic 존재 여부, 플러그인 로딩 문제를 확인. 운영에서는 tasks.max 수, 재시도 정책 확인 필수."

---

# 7️⃣ 모니터링 & 검증

## 검증 포인트
- Kafka Topic 메시지 확인 → S3 파일 생성 확인
- S3 파일 구조, JSON 스키마 확인
- Connector 로그 확인 → 실패/재시도 여부 점검
- DLQ 존재 여부 확인

## 실무 명령 예시

```bash
# Connector 상태 확인
curl http://localhost:8083/connectors/customer-logs-s3-sink/status

# S3 확인
aws s3 ls s3://customer-logs/year=2025/month=09/day=24/ --recursive
```

**명령어 설명**: 
- 첫 번째: Connector가 정상 실행 중인지, Task 상태는 어떤지 확인
- 두 번째: 실제 S3에 파일이 생성되었는지, 폴더 구조가 올바른지 확인

> **엔지니어 독백**: "테스트 데이터 몇 개 보내서 S3에 잘 올라오는지 확인. 파일 구조, 로그 스키마, DLQ 상태까지 확인해야 실제 운영 준비 완료."

## ✅ 전체 시나리오 완료 조건

1. **Kafka Connect 환경 정상 기동**
2. **S3 Sink Connector 등록 성공**
3. **테스트 로그 → S3 저장 확인**
4. **DLQ 설정 및 에러 핸들링 동작 확인**
5. **S3 파일 구조 및 JSON 포맷 검증**
