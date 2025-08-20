 -Kafka 프로듀서 Part02
>Kafka에서 producer →Broker 메시지 전송시 ByteArray 형태로 보낸다. ByteArray로 보내기 때문에 Serializer가 필요함
>ByteArray부터 차근차근 알아보자


# [Background] 바이트 배열(byteArray)과 직렬화(Serializer)





### Java 직렬화 코드 예시
```
import java.nio.charset.StandardCharsets;
import java.util.Arrays;

public class StringSerializationExample {
    public static void main(String[] args) throws Exception {
        // 원래 문자열
        String s = "Hello Kafka";

        // 직렬화: String -> byte[]
        byte[] bytes = s.getBytes(StandardCharsets.UTF_8);
        System.out.println("직렬화(byte[]): " + Arrays.toString(bytes));

        // 역직렬화: byte[] -> String
        String s2 = new String(bytes, StandardCharsets.UTF_8);
        System.out.println("역직렬화(String): " + s2);
    }
}

```
-출력결과
```
직렬화(byte[]): [72, 101, 108, 108, 111, 32, 75, 97, 102, 107, 97]
역직렬화(String): Hello Kafka
```

### Kafka에서 직렬화
- 프로듀서
  - Java 객체(Customer) → Kafka 전송용 byte[]
  - 이 과정에서 Serializer<Customer>가 필요합니다.  
   
- 브로커
  - 메시지를 그냥 바이트 배열로 저장
  - 어떤 객체인지, 어떤 포맷인지 신경 안 씀
  - 토픽, 파티션, 오프셋 정보만 관리  
  
- 컨슈머
  - Kafka에서 받은 byte[] → Java 객체(Customer)
  - 이 과정에서 Deserializer<Customer>가 필요  
  

### 직렬화가 필요한 이유

<details>
<summary>예시   </summary>

### 1️⃣ 예시: 주문(Order) 데이터 전송

**상황:**
온라인 쇼핑몰에서 주문 데이터를 Kafka로 보내야 함.

**주문 객체 구조:**
- 주문 ID: orderId
- 고객 이름: customerName
- 상품 목록: items (상품ID + 수량)
- 결제 금액: totalPrice
- 배송 주소: deliveryAddress
- 생성 시간: createdAt
<br>

**기본 JSON 직렬화 방식으로 보낼 때:**
```
{
  "orderId": "ORD12345",
  "customerName": "Alice",
  "items": [
    {"productId": "P100", "quantity": 2},
    {"productId": "P200", "quantity": 1}
  ],
  "totalPrice": 45000,
  "deliveryAddress": "Seoul, Korea",
  "createdAt": "2025-08-20T10:00:00"
}
```

**문제점:**
- 문자열이 길고, 수십만 건 전송 시 네트워크 부담 ↑

- 컨슈머 시스템이 items만 필요할 수도 있음

- JSON 파싱 비용 발생


**커스텀 시리얼라이저 적용 후(최적화):**
- 필요한 필드만 선택: orderId, items, totalPrice
- 숫자는 바이트 배열로 직접 직렬화
- 메시지 예시(바이트 배열이지만 표현하면):
```
[ORD12345][P100:2,P200:1][45000]
```

**효과:**
- 메시지 크기 ↓
- 불필요한 문자열 제거
- 컨슈머가 바로 파싱 가능


### ✅ 요약
- 커스텀 시리얼라이저 필요 이유:
- 데이터 구조가 복잡 → 필요한 필드만 전송 가능
- 성능 최적화 → 네트워크와 처리 비용 절감
- 컨슈머 호환성 → 특정 포맷으로 맞춤 전송 가능


</details>


<br>
<br>
<br> 





# 3.5 직렬화(Serializer)
### 직렬화 필요성
- 목적: Java 객체를 Kafka가 이해할 수 있는 바이트 배열로 변환
### 기본 제공 직렬화
- StringSerializer, IntegerSerializer, ByteArraySerializer 등
- 대부분 실무 상황에는 부족함 → 아니, 안부족함! 트렌드를 살펴보
<details>
<summary> 직렬화 트렌드 </summary>

### Kafka 시리얼라이저 사용 트렌드 (2025년 현재)
### 현재 업계 주류는 여전히 JSON
실리콘밸리에서 10년째 일하면서 보니까, JSON이 압도적으로 많이 쓰이고 있어. 전체 회사의 60% 이상이 JSON을 메인으로 사용하고 있고, 특히 스타트업이나 빠르게 성장하는 회사들은 거의 다 JSON이야.
왜 JSON이 이렇게 인기인지 물어보면, 답은 간단해. "디버깅이 쉬워서". 카프카 콘솔 컨슈머로 메시지 찍어보면 바로 읽을 수 있고, 로그 파일에서도 그냥 텍스트로 보여. 개발자가 새로 조인했을 때 30분이면 이해할 수 있는 수준이지.
<br> 

### Avro는 데이터 중심 회사들이 선택
Avro + Schema Registry 조합은 Netflix, LinkedIn, Uber 같은 데이터 헤비한 회사들이 주로 써. 이유는 명확해 - 스키마가 계속 바뀌는데 하위 호환성을 보장해야 하거든.
예를 들어 사용자 프로필에 새 필드를 추가할 때, 기존 컨슈머들이 터지면 안 되잖아. Avro는 이런 스키마 진화를 안전하게 해줘. 하지만 러닝 커브가 있고 Schema Registry라는 추가 인프라가 필요해서 작은 회사들은 부담스러워해.
<br> 

### 회사 규모별 패턴이 뚜렷함
스타트업 (50명 미만): 거의 100% JSON. 빠른 개발과 단순함이 최우선이야. 스키마 바뀌면? 그냥 코드 고치면 돼.
성장기 회사 (50-200명): JSON에서 시작해서 핵심 토픽만 Avro로 마이그레이션하는 패턴. 보통 결제나 사용자 데이터 같은 중요한 부분부터 Avro 적용해.
대기업 (500명+): 표준화된 Avro + 중앙집중식 Schema Registry. 데이터 거버넌스 팀이 있어서 스키마 승인 프로세스까지 갖춰져 있어.
<br> 

### Protobuf는 특수 상황에서만
Protobuf는 gRPC 쓰는 회사들이나 Google 출신 엔지니어들이 많은 팀에서 가끔 보여. 성능은 좋은데 JSON만큼 직관적이지 않아서 널리 퍼지지는 못했어.
요즘은 특히 마이크로서비스 아키텍처에서 서비스간 API는 gRPC/Protobuf 쓰고, 이벤트 스트리밍은 JSON 쓰는 하이브리드 패턴이 늘어나고 있어.
<br> 

### 2025년 새로운 트렌드들
클라우드 네이티브 영향: Serverless 함수나 managed Kafka 서비스 쓰는 회사들이 늘면서 JSON 선호도가 더 높아졌어. Lambda 함수에서 Avro 파싱하려면 라이브러리 로딩 시간 때문에 Cold Start가 느려지거든.
관찰가능성(Observability) 중시: APM 도구들이 JSON은 바로 파싱해서 보여주는데, Avro는 별도 스키마 정보가 필요해. DataDog이나 New Relic 같은 도구에서 실시간으로 메시지 내용 분석하려면 JSON이 훨씬 편해.
Kafka Connect 생태계: S3나 BigQuery로 데이터 내보내는 일이 많아지면서, Kafka Connect 호환성이 중요해졌어. JSON은 바로 Parquet으로 변환되는데, Avro는 변환 과정이 더 복잡해.
<br> 

### 실무 선택 기준
대부분 회사들이 "JSON으로 시작해서 필요할 때 Avro 도입" 전략을 써. 처음부터 Avro 쓰는 회사는 많이 줄었어. 이유는 Confluent Cloud 같은 managed Schema Registry가 생겼음에도 불구하고, 여전히 복잡도가 있기 때문이야.
<br> 

### 언제 Avro를 고려하냐면:
스키마가 자주 바뀌는데 하위 호환성이 중요할 때
네트워크 비용을 절약해야 할 때 (압축률이 좋아)
타입 안전성이 중요한 금융/헬스케어 도메인일 때
데이터 분석팀이 크고 스키마 거버넌스가 필요할 때
<br> 

### 현실적인 조언
지금 새로 Kafka 도입한다면? JSON으로 시작해. 압축은 Snappy나 Gzip 쓰고, 메시지 포맷은 표준화해서 써. 회사가 커지고 데이터가 중요해지면 그때 핵심 토픽부터 Avro로 마이그레이션 고려해봐.
MessagePack이나 Thrift 같은 다른 포맷들? 특별한 이유가 없는 한 피해. 생태계도 작고 팀원들이 학습해야 할 것도 많아져.
결론적으로 2025년 현재는 "JSON 우선, 필요시 Avro" 패턴이 가장 현실적이고 널리 받아들여지는 방식이야.
</details>
<br>


## 3.5.1 커스텀 시리얼라이저
> 99% 케이스에서 불필요솔직히 말하면 커스텀 직렬화는 거의 쓸 일이 없어. 10년 동안 실리콘밸리에서 일하면서 실제로 커스텀 직렬화 > 구현한 건 2-3번 정도야. 대부분은 기존 솔루션으로 충분해. 성능에 차지하는 비중이 적음(5%미만)
### 커스텀 직렬화의 현실적인 문제들
- 유지보수 지옥: 팀원이 바뀔 때마다 커스텀 로직 이해하느라 시간 낭비해. 특히 온보딩할 때 "이게 뭐하는 코드에요?"라는 질문 계속 받아.
- 디버깅 어려움: 바이너리 포맷이라 카프카 콘솔로 메시지 확인 못해. 별도 디버깅 도구 만들어야 하고, 운영팀에서 문제 생겼을 때 원인 파악하기 힘들어.
- 크로스 언어 지원: Python에서 직렬화하고 Java에서 역직렬화할 때 버그 생기기 쉬워. 바이트 오더링, 인코딩 차이 등등.
- 스키마 진화 불가: 새 필드 추가하거나 기존 필드 변경할 때 하위 호환성 보장하기 어려워.
<br>

### 대안들이 충분히 좋아졌어
- Avro: 컴팩트하고 스키마 진화 지원. 웬만한 성능 요구사항은 다 만족해.
- Protobuf: Google이 만든 거라 안정적이고, 크로스 언어 지원 완벽해.
- JSON + 압축: 가독성 좋고, 압축하면 크기도 많이 줄어들어. 대부분 케이스에서 충분해.
- MessagePack: JSON보다 컴팩트하면서도 스키마 없이 사용 가능.
<br>

### 언제 고려해볼 만한가
- 정말 극한의 성능이 필요할 때: HFT(고빈도 거래) 같이 마이크로초 단위가 중요한 도메인.
- 네트워크 비용이 심각할 때: 위성 통신이나 IoT처럼 바이트당 비용이 높은 환경.
- 레거시 호환성: 기존 시스템과 바이너리 호환성을 반드시 유지해야 할 때.
- 특수한 데이터 구조: 그래프나 트리 같은 복잡한 구조를 효율적으로 표현해야 할 때.

### 구현방법 - Java

**Step 1: Customer 클래스 정의**
```
public class Customer {
    private int customerID;
    private String customerName;

    public Customer(int ID, String name) {
        this.customerID = ID;
        this.customerName = name;
    }

    public int getID() {
        return customerID;
    }

    public String getName() {
        return customerName;
    }
}
```
- 설명:
  - 고객 정보를 담는 간단한 도메인 객체
  - customerID와 customerName 필드를 가지고 있음
  - getter만 제공, 데이터 읽기 용도
>"좋아, 먼저 Customer 객체가 있어야겠지.
>ID랑 이름만 필요하니까 간단하게 만들어야겠다.
>필드는 private로 숨기고, getter만 제공하면 충분하겠네.
>이제 직렬화할 때 이 필드들을 가져올 수 있겠군."

```
import org.apache.kafka.common.errors.SerializationException;
import org.apache.kafka.common.serialization.Serializer;

import java.nio.ByteBuffer;
import java.nio.charset.StandardCharsets;
import java.util.Map;

public class CustomerSerializer implements Serializer<Customer> {

    @Override
    public void configure(Map<String, ?> configs, boolean isKey) {
        // 별도로 설정할 내용 없음
    }

    /**
     * Customer 객체를 직렬화하는 방식:
     * 1. 4 byte int: customerId
     * 2. 4 byte int: customerName 길이 (UTF-8 기준, null이면 0)
     * 3. N byte: customerName (UTF-8)
     */
    @Override
    public byte[] serialize(String topic, Customer data) {
        if (data == null) {
            return null;
        }

        try {
            byte[] nameBytes;
            int nameLength;

            if (data.getName() != null) {
                nameBytes = data.getName().getBytes(StandardCharsets.UTF_8);
                nameLength = nameBytes.length;
            } else {
                nameBytes = new byte[0];
                nameLength = 0;
            }

            ByteBuffer buffer = ByteBuffer.allocate(4 + 4 + nameLength);
            buffer.putInt(data.getID());
            buffer.putInt(nameLength);
            buffer.put(nameBytes);

            return buffer.array();

        } catch (Exception e) {
            throw new SerializationException("Error serializing Customer", e);
        }
    }

    @Override
    public void close() {
        // 자원 해제할 내용 없음
    }
}

```
<br>

**Step 2: CustomerSerializer 클래스 선언**
```
import org.apache.kafka.common.errors.SerializationException;
import org.apache.kafka.common.serialization.Serializer;
import java.nio.ByteBuffer;
import java.util.Map;

public class CustomerSerializer implements Serializer<Customer> {
```

>독백:
>"이제 Kafka 프로듀서에서 쓸 시리얼라이저를 만들어야지.
>Serializer<Customer>를 구현하면 Customer 객체를 바로 전송할 수 있으니까, 이걸 상속하자.
>설정이나 종료는 필요 없지만, 메서드는 만들어야겠지."
<br>


**Step 3: configure & close 메서드 구현**
```
    @Override
    public void configure(Map configs, boolean isKey) {
        // 설정 없음
    }

    @Override
    public void close() {
        // 종료 시 자원 해제 없음
    }
```
>독백:
>"현재 특별히 설정할 것도 없고, 종료 시에 리소스 정리할 것도 없네.
>하지만 Kafka가 요구하니까 메서드는 만들어야지."
<br>


**Step 4: serialize 메서드 구현**
```
    @Override
    public byte[] serialize(String topic, Customer data) {
        try {
            if (data == null)
                return null;

            byte[] serializedName;
            int stringSize;

            // 이름 null 체크 후 UTF-8 바이트 변환
            if (data.getName() != null) {
                serializedName = data.getName().getBytes("UTF-8");
                stringSize = serializedName.length;
            } else {
                serializedName = new byte[0];
                stringSize = 0;
            }

            // ByteBuffer로 데이터 순서대로 저장: ID, 이름 길이, 이름 바이트
            ByteBuffer buffer = ByteBuffer.allocate(4 + 4 + stringSize);
            buffer.putInt(data.getID());
            buffer.putInt(stringSize);
            buffer.put(serializedName);

            return buffer.array();
        } catch (Exception e) {
            throw new SerializationException(
                "Error when serializing Customer to byte[] " + e);
        }
    }
}
```
- 코드설명
  - null 체크 : Customer 객체가 null이면 null 반환
  - 이름 처리 : 이름이 null이면 길이 0, 바이트 배열도 빈 배열
  - ByteBuffer 사용:
    - 4바이트 : customerID
    - 4바이트 : customerName 길이
    - N바이트 : customerName 실제 값
  - 예외 처리 : 직렬화 실패 시 SerializationException 발생
<br>

>독백:
>"자, 이제 핵심 부분.
>먼저 Customer 객체가 null인지 체크해야 해. null이면 그냥 null 반환.
>이름도 null일 수 있으니까, null이면 길이 0, 바이트 배열도 빈 배열로 처리하자.
>ByteBuffer를 만들어서, 순서대로 넣는다.
>먼저 customerID 4바이트, 다음 이름 길이 4바이트, 마지막으로 이름 바이트 배열.
>이 순서대로 넣으면 나중에 컨슈머가 역직렬화할 때 쉽게 읽을 수 있겠지.
>혹시 UTF-8 변환 중 에러가 나면 SerializationException 던져서 프로듀서가 알 수 있게 해야겠다."
<br>



**Step 5: 프로듀서에서 사용**
```
Producer<String, Customer> producer = new KafkaProducer<>(props, new StringSerializer(), new CustomerSerializer());
Customer customer = new Customer(1, "Alice");
producer.send(new ProducerRecord<>("customer-topic", "key1", customer));
```
>독백:
>"프로듀서 생성 시, ValueSerializer로 방금 만든 CustomerSerializer를 지정하자.
>이제 Customer 객체를 그대로 ProducerRecord에 넣어서 전송할 수 있네.
>브로커는 바이트 배열만 전달하니까, 시리얼라이저는 필요 없어.
>컨슈머만 역직렬화하면 끝이야."
<br><br>


## 3.5.2 Serializer - Avro 사용하기
## 3.5.3 Avro records 사용하기
## Avro ( 3.5.2장, 3.5.3 통합)
### (1) 배경
-  대용량 데이터를 여러 시스템 간에 안전하게 공유해야 하는 필요
- 스키마란 수시로 변경될 수 있음 → 스키마 변경에도 관리가 자동화되도록 할 수 없을까?
- Apache Hadoop 창시자인 더그커팅이 Avro 개

### (2) Avro란 - 기술측면    
> Avro의 핵심 기술  
>1. Schema Registry                   : 모든 스키마를 중앙에서 관리 → 스키마 ID로 메시지에 참조
>2. 스키마 ID 포함                      : 메시지에 어떤 스키마로 직렬화했는지 정보 포함 → 컨슈머가 정확한 스키마로 역직렬화 가능
>3. 기본값/optional 필드(핵심)   : 새 필드 추가 시 기존 컨슈머가 몰라도 기본값으로 처리 → 호환성 유지
>즉, 스키마 분리 + ID 참조 + 기본값 처리가 Avro 호환성의 핵심 기술 아이디어입니다.

- 스키마 변경되도 호환되는 원리
  - 기존 JSON방식 
    - 단순히 파싱해서 키,밸류값을 가져오는 방식
    - 스키마 변경 시 기존 애플리케이션이 깨짐
    ```
    # JSON 방식의 문제점
    old_data = {"id": 1, "name": "John", "fax": "123-456-7890"}
    new_data = {"id": 1, "name": "John", "email": "john@example.com"}
    # → 스키마 변경 시 기존 애플리케이션이 깨짐
    ```
  - avro 방식
    - "default" : null 이 있으면 옵션값으로 인지함 
    - 옵션 필드는 null값이 기본값이 되는게 필수 조건
    ```
    {
      "type": "record",
      "name": "Customer",
      "namespace": "com.example.avro",
      "fields": [
        {"name": "customerID", "type": "int"},
        {"name": "customerName", "type": "string"},
        {"name": "email", "type": ["null", "string"], "default": null}
      ]
    }
    ```
- 용량 압축 효과
  - JSON: 텍스트 기반, 용량 큼
  - Avro: 바이너리 기반, 50-80% 용량 절약

- Spark streaming같은 분석 툴에서 스키마 검증 가능
  - 스키마 레지스트리에서 스키마정보를 불러옴
<br>

### (3) Avro 구성요소 및 스키마 관리  

- **스키마 정의 파일(.avsc)**
  - .avsc 파일로 작성 → Git 등에서 관리
  - Gradle에 등록필
  - 예시:
    ```
    {
      "type": "record",
      "name": "User",
      "fields": [
        {"name": "id", "type": "int"},
        {"name": "name", "type": "string"},
        {"name": "email", "type": ["null", "string"], "default": null}
      ]
    }
    ```

- **Schema Registry**
  - Kafka와 함께 쓰이는 중앙 스키마 관리 서버
    - Kafka와 별도 서비스
    - 따로 컨테이너에 띄워서 사용하는 형태 
  - REST API 형태로 동작 (보통 Confluent Schema Registry 사용)
  - 역할:
    - Avro 스키마를 저장하고, 각 스키마에 고유한 ID 부여
    - 스키마 등록/조회
    - 버전 관리
    - 호환성 체크 (BACKWARD, FORWARD, FULL)
  - URL 확인: schema.registry.url=http://localhost:8081 → curl 명령어로 스키마 확
  - 확인 예시:
    ```
    curl http://localhost:8081/subjects
    curl http://localhost:8081/subjects/user-value/versions/1
    ```
  - 스키마 ID
    - 메시지에 포함되어 전송  
    - 동일한 스키마라면 다른 프로듀서에서 생성된 메시지라도 기존 스키마 ID 재사용
    - 스키마 변경 → 새로운 ID 발급 (호환성 체크 수행)
    
  - 저장되는 값
    | 항목               | 설명                                                     | 예시                                      |
    |-------------------|--------------------------------------------------------|------------------------------------------|
    | Subject           | 스키마가 속한 논리적 그룹. 일반적으로 Kafka 토픽 이름 + 역할 | customer-value                           |
    | Version           | 스키마 버전 번호                                           | 1, 2, 3                                  |
    | Schema ID         | 전역적으로 유일한 스키마 식별자                              | 1001                                     |
    | Schema Definition | 실제 Avro/JSON/Protobuf 스키마 문자열                       | { "type": "record", "name": "Customer", ... } |
    | Compatibility Level | 해당 subject에 적용되는 호환성 정책                         | BACKWARD, FORWARD, FULL                  |

  - 스키마 호환성
      | 모드      | 등록 가능 조건                                                                 | 설명                             |
      |-----------|------------------------------------------------------------------------------|----------------------------------|
      | BACKWARD  | 새 스키마로 이전 메시지 읽기 가능<br>새 필드에 default 필요                       | 과거 메시지는 새로운 스키마로 역직렬화 가능 |
      | FORWARD   | 이전 스키마로 새 메시지 읽기 가능<br>새 메시지 역직렬화 가능                        | 새 메시지는 예전 스키마로 역직렬화 가능    |
      | FULL      | BACKWARD + FORWARD 조건 모두 만족                                               | 양방향 호환성                    |
      | NONE      | 체크 없음<br>무조건 등록 가능, 위험                                               | 아무 제약 없이 등록 가능, 위험함          |


### (4) Avro 프로듀서에 구현하기
<br> 

**1. 전체 흐름 (Producer → Kafka → Consumer)에서 살펴보기**


**[1] Avro 스키마 정의**
    - 개발자가 JSON 형식으로 스키마 작성(.avsc)
    ```
    {
      "type": "record",
      "name": "User",
      "fields": [
        {"name": "id", "type": "int"},
        {"name": "name", "type": "string"},
        {"name": "email", "type": ["null", "string"], "default": null}
      ]
    }
    ```

**[2] Producer 직렬화**
- 애플리케이션에서 Avro Serializer 사용(Confluent에서 제공)
- 데이터를 Avro 바이너리 포맷으로 변환
- 이때, 해당 스키마를 Schema Registry에 등록 (최초라면 새로운 ID 발급)
- Schema registry가 없으면 Serializer 사용 불가 

**[3] Kafka 전송**

- Kafka 메시지는 이렇게 구성됨:
```
[매직바이트(1바이트)] + [스키마 ID(4바이트)] + [Avro 직렬화된 데이터]
```
- 👉 즉, 메시지 본문에는 스키마 전체가 들어가지 않고, 대신 Schema Registry에 저장된 스키마 ID만 포함

**[4] Consumer 역직렬화**
- 메시지를 받으면 Avro Deserializer가 스키마 ID 확인
- Schema Registry에서 해당 ID에 맞는 스키마 가져옴
- 바이너리 데이터를 해석 → 원래의 User 객체로 변환
<br>
<br>


**2. 구현과정**
-자바기준
<br>

**[1] 디렉터리 구조**
  ```
  avro-gradle-demo/
  ├─ build.gradle
  ├─ settings.gradle
  ├─ src/
  │  └─ main/
  │     ├─ avro/
  │     │  └─ Customer.avsc
  │     └─ java/
  │        └─ com/example/producer/AvroProducer.java
  ```

**[2] Gradle 설정 (생략)
**[3] avro파일 생성 (Customer.avsc)**
- 경로 : `src/main/avro/Customer.avsc`
```
// src/main/avro/Customer.avsc
{
  "type": "record",
  "name": "Customer",
  "namespace": "com.example.avro",
  "fields": [
    {"name": "customerID", "type": "int"},
    {"name": "customerName", "type": "string"},
    {"name": "email", "type": ["null", "string"], "default": null}
  ]
}
```
>optional 규칙: ["null","string"] + default: null (순서 중요)


**[4] 

```
package com.example.producer;

import com.example.avro.Customer;                            // ✅ avro 플러그인이 생성한 클래스 (avro 의존성 + 플러그인)
import org.apache.kafka.clients.producer.*;                  // ✅ kafka-clients 의존성
import org.apache.kafka.common.serialization.StringSerializer;
import io.confluent.kafka.serializers.KafkaAvroSerializer;   // ✅ kafka-avro-serializer 의존성

import java.util.Properties;

public class AvroProducer {
    public static void main(String[] args) {
        String topic = "customer-topic";

        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");

        // Key/Value 직렬화기
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());   // kafka-clients
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, KafkaAvroSerializer.class.getName()); // confluent serializer

        // Schema Registry 위치 (Confluent SR)
        props.put("schema.registry.url", "http://localhost:8081");

        // (선택) 스키마 자동 등록
        props.put("auto.register.schemas", "true");
        props.put("use.latest.version", "false");

        try (KafkaProducer<String, Customer> producer = new KafkaProducer<>(props)) { // kafka-clients
            Customer c1 = Customer.newBuilder()               // avro codegen 산출물 사용
                    .setCustomerID(1001)
                    .setCustomerName("Alice")
                    .setEmail("alice@example.com")            // optional: null 가능
                    .build();

            ProducerRecord<String, Customer> record =
                    new ProducerRecord<>(topic, String.valueOf(c1.getCustomerID()), c1); // kafka-clients

            producer.send(record, (md, ex) -> {               // kafka-clients
                if (ex != null) ex.printStackTrace();
                else System.out.printf("sent %s-%d@%d key=%s%n",
                        md.topic(), md.partition(), md.offset(), record.key());
            });

            producer.flush();
        }
    }
}
```
- 약식코드
```
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "io.confluent.kafka.serializers.KafkaAvroSerializer");
props.put("schema.registry.url", "http://localhost:8081");

KafkaProducer<String, GenericRecord> producer = new KafkaProducer<>(props);

GenericRecord user = new GenericData.Record(schema);
user.put("id", 1);
user.put("name", "Alice");

ProducerRecord<String, GenericRecord> record = new ProducerRecord<>("user-topic", user);
producer.send(record);
```
- 스키마 자동등록 되는 옵션
  - 자동 등록 원하면 auto.register.schemas=true 필수
  - 자동 등록 안 하면 미리 스키마 등록 필요, 안 하면 에러 발생 가능
  - 첫 메시지 전송시 serializer에 의해 .avsc 파일 스키마가 등록 
```
    props.put("auto.register.schemas", "true");
    props.put("use.latest.version", "false");
```
- 수동등록(개발자가 직접)
```
# 개발자가 수동으로 등록
curl -X POST \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  -d @user-schema.avsc \
  http://localhost:8081/subjects/users-value/versions
```
<br>





**[4] 컨슈머, Spark streaming(생략)**

- 구성요소:
  - Producer (KafkaAvroSerializer) → 스키마 등록
  - Consumer (KafkaAvroDeserializer) → 스키마 조회
  - Schema Registry → 스키마 저장소 + 버전 관리 + 호환성 검사

<details>
<summary> avro관련 질문들 </summary>

### 스키마 등록, 확인은 누가 하나? -Serializer
- 스키마 등록과정
  - 프로듀서가 메시지 전송 시 새로운 스키마를 직렬화
  - KafkaAvroSerializer → Schema Registry에 새 스키마 등록 요청
  - 호환성 체크 수행
    - 통과하면 → 새 스키마 ID 발급
    - 실패하면 → 등록 거부, 예외 발생
<br>

### 다른 프로듀서가 같은 스키마를 쓰는 경우
>Schema Registry는 스키마 내용(JSON) 자체를 기준으로 ID를 발급합니다.
>따라서 다른 프로듀서에서 같은 필드와 타입으로 Avro 객체를 만들어 전송하면,
>같은 subject 이름(topic-value 등) 기준으로
>Schema Registry가 기존 스키마 ID를 재사용합니다.
>즉, 중앙에서 스키마를 공유하는 효과가 생깁니다.
<br>

### 한쪽 프로듀서에서 스키마가 변경되면?
- 예: User 스키마에 age:int 필드를 추가했다고 가정
**<Schema Registry 호환성 설정 확인>**
- Schema Registry는 subject마다 호환성 모드를 설정할 수 있습니다:
  - BACKWARD – 새 스키마는 이전 스키마와 호환되어야 함
    - 기존 데이터를 새 스키마로 읽을 수 있어야 함
  - FORWARD – 이전 스키마로도 새 데이터를 읽을 수 있어야 함
  - FULL – 양쪽 모두 호환 가능
  - NONE – 호환성 체크 없음
<br>

### 프로듀서에서 스키마 ID를 직접 지정해야 하나?
- 아니요. 대부분 지정할 필요 없음
- KafkaAvroSerializer가 메시지를 직렬화할 때
  - 로컬 캐시에서 스키마 확인
  - Schema Registry에 등록/조회
  - 발급받은 스키마 ID를 메시지 헤더에 포함
- 따라서 프로듀서는 Avro 객체를 만들기만 하면,
  - Schema Registry가 자동으로 ID를 부여/관리합니다.
- 직접 ID 지정은 거의 사용하지 않음.
<br>

### Spark에서 스키마 레지스트리 사용여부
- Apache Spark 자체는 Avro 메시지를 처리할 수 있지만
  - Avro 스키마를 직접 .avsc 파일로 관리하거나
  - 코드에서 SpecificRecord를 정의해야 합니다.
- Schema Registry를 직접 사용하려면
  - Spark가 Kafka에서 메시지를 읽을 때, KafkaAvroDeserializer처럼 Schema Registry를 참조하도록 설정해야 합니다.
- 즉, Spark 자체가 레지스트리를 갖고 있는 것은 아니고,
- Kafka+Schema Registry와 함께 사용하면서 메시지 역직렬화 시 참조하는 구조입니다.
<br>

### 스키마 레지스트리 로컬 설치는?
```
Confluent Schema Registry 패키지 다운로드 → schema-registry.properties 설정 → schema-registry-start 실행

기본 포트: 8081
설정파일: $CONFLUENT_HOME/etc/schema-registry/schema-registry.properties
```
<br>

### Confluent schema registry말고 다른 선택지 있나요?
- Confluent Schema Registry (자체 구축/Confluent Cloud)
  - 저장: Kafka(내부 토픽). 업계 표준에 가까움.
- Apicurio Registry (오픈소스, Red Hat
  - 저장소 플러그블: Kafka 또는 RDB(PostgreSQL/MySQL) 등 선택 가능.
- AWS Glue Schema Registry (Managed)
  - 완전관리형. AWS 생태계와 밀결합(Kinesis, MSK, Lambda 등).
- Aiven Karapace (오픈소스/Managed)
  - Confluent 호환 API. 저장은 보통 Kafka.
- Azure Event Hubs Schema Registry / GCP Pub/Sub + Schema
  - 각 클라우드의 관리형 레지스트리.
- 현업 선택 기준
  - Kafka 온프레/Confluent 사용 → Confluent SR가 가장 자연스러움.
  - Red Hat/Quarkus/JBoss 생태계 → Apicurio.
  - 클라우드 네이티브 → AWS Glue / Azure / GCP 관리형 우선.
  - RDB로 중앙화 관리 필요 → Apicurio(RDB 모드) 고려.

</details>

<br>
<br>
<br>


# 3.6 파티션
<br>

## (1) 전체 흐름 요약

### 1단계: 메시지 키-값 구조 이해
- 모든 Kafka 메시지는 key-value 쌍으로 구성
- 같은 키를 가진 메시지들은 동일한 파티션으로 전송됨
- 이를 통해 순서 보장과 효율적인 데이터 처리 가능
<br>


### 2단계: 기본 파티셔닝 전략
- 키가 null인 경우: 라운드로빈 방식으로 랜덤 파티션 할당
- 키가 있는 경우: 키를 해싱하여 특정 파티션에 일관되게 매핑
- Kafka 2.4부터는 "sticky" 알고리즘으로 배치 처리 최적화
<br>


### 3단계: 커스텀 파티셔닝
- 특정 비즈니스 로직에 따라 파티션 분배 전략 커스터마이징 가능
- 예시: 대용량 고객사 데이터를 별도 파티션으로 분리
<br>


## (2) 실무적 활용 예시
### 기본 메시지 생성코드:
```
#파이썬

# 키가 있는 레코드 (Python kafka-python 라이브러리 기준)
producer.send('CustomerCountry', key='Laboratory Equipment', value='USA')

# 키가 없는 레코드
producer.send('CustomerCountry', value='USA')
```
<br>

### 왜 이렇게 하는가?
- 데이터 처리 순서 보장 (같은 고객의 이벤트는 순차 처리)
- 컨슈머 병렬 처리 효율성 극대화
- 특정 키의 모든 데이터를 한 컨슈머가 처리하도록 보장
<br>


### 주의사항:
- 파티션 수가 변경되면 키-파티션 매핑이 깨질 수 있음
- 처음부터 충분한 파티션 수로 토픽 생성하는 것이 중요
- 데이터 스큐(특정 키에 데이터 집중) 문제 고려 필요
<br>

### 파티셔너 설정값
**1. DefaultPartitioner (기본값)**
- 언제 사용: 일반적인 상황에서 키 기반 분산이 필요할 때
```
python
# 설정하지 않으면 자동으로 이게 적용됨
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092']
    # partitioner 미지정시 DefaultPartitioner 사용
)
```
- 동작 방식:
- 키가 있으면: murmur2(key) % partition_count
- 키가 null이면: sticky round-robin (Kafka 2.4+)
- 실무적 판단 기준: "대부분 이걸 쓴다. 키가 고르게 분포되어 있고 특별한 요구사항이 없다면 기본값이 최선이야."
<br>



**2. RoundRobinPartitioner**
- 언제 사용: 키와 상관없이 무조건 균등 분배가 필요할 때
```
python
from kafka.partitioner import RoundRobinPartitioner

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    partitioner=RoundRobinPartitioner()
)
```
- 실무 사용 사례:
  - 로그 수집 시스템 (키 의미 없음)
  - 배치 처리용 대용량 데이터
  - 키 분포가 심하게 skewed된 경우
- 엔지니어 독백: "키가 있어도 무시하고 돌아가면서 보낸다. 순서는 포기하고 균등 분배를 택한 거지."
<br>


**3. UniformStickyPartitioner**
- 언제 사용: 배치 효율성 + 균등 분배 둘 다 원할 때
```
python
from kafka.partitioner import UniformStickyPartitioner

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    partitioner=UniformStickyPartitioner()
)
```
- 핵심 특징:
  - 한 배치가 찰 때까지 같은 파티션으로 보냄 (sticky)
  - 배치가 차면 다음 파티션으로 이동 (uniform)
- 실무적 이점: "네트워크 요청 수를 줄여서 latency 개선. 처리량이 중요한 실시간 시스템에서 많이 써."

<br>

## 파티션 스큐 모니터링
- 명령어 kafka-run-class.sh
```
# 파티션별 메시지 분포 확인
kafka-run-class.sh kafka.tools.GetOffsetShell \
  --broker-list localhost:9092 \
  --topic your-topic --time -1

# 결과 예시 - 분포가 고른지 확인
your-topic:0:15023    # 파티션 0에 15,023개
your-topic:1:14,987   # 파티션 1에 14,987개  
your-topic:2:15,102   # 파티션 2에 15,102개
```
- 위치 - kafka-run-class.sh
```
# Kafka 설치 디렉토리 구조
/opt/kafka/
├── bin/
│   ├── kafka-run-class.sh          # 이 파일!
│   ├── kafka-topics.sh
│   ├── kafka-console-producer.sh
│   ├── kafka-console-consumer.sh
│   └── ... (기타 스크립트들)
├── config/
└── libs/

또는 
#위치찾기 
which kafka-topics.sh 
```


<details>
<summary> (3) 커스텀 파티셔너 구현하기 </summary>

```
from kafka.partitioner.base import Partitioner
import hashlib

class CustomBusinessPartitioner(Partitioner):
    """
    실무 예시: VIP 고객은 전용 파티션, 일반 고객은 해시 분산
    """
    
    def __init__(self):
        # VIP 고객 리스트 (실제로는 Redis나 DB에서 조회)
        self.vip_customers = {'samsung', 'apple', 'google'}
        
    def partition(self, key, all_partitions, available_partitions):
        """
        파티션 결정 로직
        - VIP 고객: 마지막 파티션 (전용 처리)
        - 일반 고객: 나머지 파티션에 해시 분산
        """
        
        if key is None:
            # 키가 없으면 라운드로빈
            return available_partitions[0]
            
        key_str = key.decode('utf-8') if isinstance(key, bytes) else str(key)
        
        # VIP 고객 체크
        if key_str.lower() in self.vip_customers:
            vip_partition = len(all_partitions) - 1  # 마지막 파티션
            print(f"VIP customer {key_str} → Partition {vip_partition}")
            return vip_partition
        
        # 일반 고객은 나머지 파티션에 해시 분산
        hash_value = int(hashlib.md5(key.encode()).hexdigest(), 16)
        regular_partitions = len(all_partitions) - 1  # VIP 파티션 제외
        partition_id = hash_value % regular_partitions
        
        return partition_id

# 사용법
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    partitioner=CustomBusinessPartitioner()
)
```
</details>

<br>
<br>






# 3.7 헤더
<br>

## 헤더(Headers)개념 
- 메시지에 붙이는 메타데이터 태그라고 생각하면 됨.
- 메시지의 키,밸류값 제외하고 필요한 정보를 싣어보냄.  
```
Kafka Message 구조:
┌─────────────────────────────────────┐
│ Headers (메타데이터)                 │
├─────────────────────────────────────┤
│ Key (파티셔닝용)                     │
├─────────────────────────────────────┤
│ Value (실제 데이터)                  │
└─────────────────────────────────────┘
```
```
[Headers: source=payment, tier=VIP] + [Key: user123] + [Value: 실제데이터]
```

## 헤더 구성
- 헤더도 키밸류 형태
```
# Headers 구조
headers = [
    ('key1', b'value1'),  # 키는 String, 값은 bytes
    ('key2', b'value2'),
    ('key1', b'value3')   # 같은 키 중복 가능 (순서 유지)
]
```
- 순서 보장: 추가한 순서대로 유지됨
- 중복 허용: 같은 키에 여러 값 가능
- 키: 항상 String
- 값: bytes (직렬화된 객체)
<br>


## 활용 Case
- 필요한 정보만 붙여서 빠르게 분류/처리하는 용도로 사용.
<br>

### 1. 데이터 파싱 없이 라우팅
```
python
# 암호화된 데이터도 헤더로 라우팅 가능
headers = {'customer_tier': b'VIP', 'region': b'EU'}
producer.send('orders', value=encrypted_data, headers=headers)
```
> 'b는 바이트타입(ByteType)임을 의미
>엔지니어 독백: "메시지 내용 안 봐도 VIP인지 알 수 있어서 빠르게 처리 가능하지."
<br>

### 2. 추적/모니터링
```
pythonheaders = {
    'trace_id': b'abc-123',
    'source_service': b'payment-api'
}
```

엔지니어 독백: "장애 났을 때 어느 서비스에서 온 메시지인지 바로 알 수 있어."
<br>
<br>


## 실무에서 어떻게 써?
### Producer에서
```
python
# 기본 사용법
record.headers().add("privacy-level", "YOLO".getBytes())

# Python에서
producer.send('topic', 
    key='user123',
    value=data,
    headers=[('source', b'payment'), ('tier', b'VIP')]
)
```
<br> 


### Consumer에서
```
python
for message in consumer:
    headers = dict(message.headers)
    
    if headers.get('tier') == b'VIP':
        # VIP 우선 처리
        vip_processor.handle(message.value)
    else:
        # 일반 처리
        normal_processor.handle(message.value)
```
엔지니어 독백: "헤더만 보고 분기 처리하니까 성능도 좋고 코드도 깔끔해."

## 주의사항
- 헤더도 메시지 크기에 포함됨 - 너무 크게 만들지 말기
- 보안 정보는 헤더에 넣지 말기 - 암호화 안됨
- 키는 String, 값은 bytes
>엔지니어 독백: "편하다고 무작정 넣지 말고, 꼭 필요한 메타데이터만 넣어야 해."
> 필요한 정보만 붙여서 빠르게 분류/처리하는 용도로 사용하자


<br>
<br>
<br>



# 3.8 인터셉터
<br>

##Interceptor가 뭐야?
- 코드 수정 없이 Producer 동작을 가로채서 추가 기능 넣는 방법
```
python
# 원본 코드 건드리지 않고
producer.send('topic', data)  
```
- ↓ Interceptor가 중간에 끼어들어서
  -  1. 전송 전 처리 (헤더 추가, 로깅 등)
  -  2. 응답 후 처리 (메트릭 수집 등)
>엔지니어 독백: "모든 애플리케이션에 동일한 기능 추가하려면 이게 답이야. 코드 하나하나 고칠 필요 없어."
<br>
<br>

## 주요 메서드 2개
### 1. onSend() - 전송 전 호출
```
java
ProducerRecord onSend(ProducerRecord record) {
    // 메시지 수정하거나 정보 수집
    record.headers().add("source", "my-service".getBytes());
    return record;  // 수정된 레코드 리턴
}
```
<br>

### 2. onAcknowledgement() - 응답 후 호출
```
java
void onAcknowledgement(RecordMetadata metadata, Exception exception) {
    // 메트릭 수집, 로깅 등
    if (exception == null) {
        successCount.increment();
    }
}
```
>엔지니어 독백: "onSend에서는 메시지 조작 가능하고, onAcknowledgement에서는 결과만 볼 수 있어."
<br>


## 실무 활용 사례
- 모니터링: 전송/성공 메시지 수 카운팅
- 헤더 자동 추가: 모든 메시지에 공통 헤더 삽입
- 민감정보 제거: PII 데이터 자동 마스킹
- 추적: 분산 트레이싱 정보 자동 삽입
<br>


## 사용법 (코드 수정 없이!)
```
bash
# 1. JAR 파일을 클래스패스에 추가
export CLASSPATH=$CLASSPATH:./interceptor.jar

# 2. 설정 파일 생성
# producer.config:
interceptor.classes=com.example.MyInterceptor

# 3. 기존 애플리케이션에 설정만 추가해서 실행
kafka-console-producer.sh --producer.config producer.config
```
> 엔지니어 독백: "기존 운영 중인 애플리케이션에 설정 파일만 바꿔서 적용할 수 있어. 무중단으로 기능 추가 가능한 거지."
<br>

## 핵심
- Interceptor = AOP(관점지향프로그래밍)의 Kafka 버전
- 코드 변경 없이 횡단 관심사(로깅, 모니터링, 보안) 처리하는 깔끔한 방법.

<details>
<summary> AOP가 뭐여?</summary>

### AOP = 공통 기능을 따로 분리하는 기법
- Aspect-Oriented Programming (관점지향프로그래밍)
- 회사에서 모든 직원이 출입할 때 보안카드 찍는 것처럼, 모든 메서드 실행 시 자동으로 실행되는 공통 기능
- 파이썬 데코레이터 
```
# 데코레이터 (Python)
@log_execution
@check_security
def transfer_money():
    pass

# 이건 어노테이션 (Java)
@Override
@Transactional
public void transferMoney() {
}
```
>엔지니어 독백: "맞아, 내가 헷갈리게 말했네. 파이썬은 데코레이터고, 자바는 어노테이션이야."

</details>

<br>
<br>
<br>








# 3.9 쿼터, 스로틀링
<br>
## Quotas가 뭐야?
- 클라이언트의 Kafka 사용량을 제한하는 기능
- 과도한 트래픽으로 브로커가 죽는 걸 방지함
<br>

## 3가지 제한 타입
- **Produce Quota**: 초당 전송 바이트 제한  
- **Consume Quota**: 초당 수신 바이트 제한  
- **Request Quota**: 브로커 처리 시간 비율 제한  
>💡 엔지니어 독백: *"특정 클라이언트가 브로커를 독점하지 못하게 하는 거지. 공유 자원 보호용이야."*
<br>


## 설정 방법
### 1. 정적 설정 (broker config)
```
<Bash>
모든 Producer 2MB/s 제한
quota.producer.default=2M

특정 클라이언트 개별 설정
quota.producer.override="clientA:4M,clientB:10M"
```
<br>


### 2. 동적 설정 (실시간 변경 - 권장)
```
<Bash>

특정 클라이언트 제한
kafka-configs --bootstrap-server localhost:9092 --alter
--add-config 'producer_byte_rate=1024'
--entity-name clientC --entity-type clients

사용자별 제한
kafka-configs --bootstrap-server localhost:9092 --alter
--add-config 'producer_byte_rate=1024,consumer_byte_rate=2048'
--entity-name user1 --entity-type users

전체 기본값 변경
kafka-configs --bootstrap-server localhost:9092 --alter
--add-config 'consumer_byte_rate=2048' --entity-type users
```
>💡 엔지니어 독백: *"동적 설정이 훨씬 편해. 브로커 재시작 없이 바로 적용되거든."*

<br>


## 제한 걸리면 어떻게 됨?
- **Throttling 발생**: 브로커가 응답을 지연시켜서 클라이언트 속도를 강제로 줄임
```
클라이언트에서 확인 가능한 메트릭
produce-throttle-time-avg # 평균 지연 시간
produce-throttle-time-max # 최대 지연 시간
fetch-throttle-time-avg # 평균 Fetch 지연
fetch-throttle-time-max # 최대 Fetch 지연
```

>💡 엔지니어 독백: *"제한 걸리면 응답이 느려져서 자연스럽게 요청 속도가 줄어들어. 브로커를 보호하는 메커니즘이지."*
<br>



## 실무 사용 시나리오
- 멀티테넌트 환경: 팀별로 사용량 제한  
- 비용 제어: 특정 서비스의 과도한 사용 방지  
- 장애 방지: 한 클라이언트가 전체 클러스터에 영향 주는 것 차단  
<br>


## 핵심
👉 **Quotas = Kafka의 교통경찰**  
과속하는 클라이언트를 잡아서 브로커 안정성을 확보하는 기능


<br>
<br>


# 3.10 요약
## Chapter 3에서 다룬 내용
### 1. Producer 기초
  - 10줄 코드로 시작하는 간단한 예제
  - 에러 핸들링 추가
  - 동기 / 비동기 전송 방식 비교

### 2. Producer 설정
  - 핵심 설정 매개변수들
  - 설정 값에 따른 동작 방식 변경

### 3. 직렬화 (Serializers)
  - 이벤트 형식 제어 방법
  - Avro 직렬화 심화 학습

### 4. 파티셔닝
  - Kafka 파티셔닝 메커니즘
  - 커스텀 파티셔닝 고급 기법
