# Chapter 5. 프로그램 내에서 코드로 카프카 관리하기
- 사용자 입력에 기반해서 새 토픽 작성하는 경우는 흔하다
- `Apache Kafka` 버전 0.11부터 프로그램적인 관리 기능 API 제공 목적으로 `AdminClient` 추가
    - 토픽 목록 조회, 생성, 삭제, 클러스터 상세 정보 확인, ACL 관리, 설정 변경 등의 기능
## 5.1 AdminClient 개요
### 5.1.1 비동기적이고 최종적 일관성 갖는 API
#### `AdminClient`를 이해할 때 가장 중요한 것은 비동기적 (Asynchronous) 으로 작동한다는 것
- 각 메서드는 요청을 클러스터 컨트롤러로 전송한 뒤 바로 1개 이상의 `Future` 객체 리턴
	> `AdminClient`에게 "토픽 A를 만들어줘!"라고 명령하면, `AdminClient`는 그 명령을 카프카에 전달하자마자 "응, 알겠어. 만드는 중이야!"라는 의미의 `접수증(Future 객체)`만 바로 돌려주고, 토픽이 실제로 다 만들어질 때까지 기다려주지 않는다. 
- 덕분에 애플리케이션은 토픽이 생성되는 동안 멈추지 않고 다른 중요한 작업을 계속 처리할 수 있다. 
	- 이는 시스템 전체의 효율과 응답성을 크게 높여준다.
- `Future` 객체는 아래의 역할을 수행하는 메서드 갖고 있다.
	- 비동기 작업 결과 확인
	- 비동기 작업 결과 취소
	- 비동기 작업 완료될 때까지 대기
	- 비동기 작업 완료되었을 때 실행할 함수 지정
- `AdminClient`는 `Future` 객체를 `Result` 객체 안에 감싸는데, `Result` 객체는 작업이 끝날때까지 대기하거나 작업 결과에 대해 일반적으로 뒤이어 쓰이는 작업 수행하는 헬퍼 메서드 갖고 있다.
#### 최종적 일관성 (Eventual Consistency)
- `AdminClient`를 사용해 **토픽 생성**을 요청하는 것은 **비동기** 작업
1. 이 요청은 클러스터의 리더인 **`클러스터 컨트롤러`**에게 전달
2. `컨트롤러`는 요청을 받자마자 "알겠다"는 의미의 **`Future` 객체**를 돌려주고, 실제 작업에 착수
3. `컨트롤러`는 클러스터의 상태 정보(메타데이터)를 변경하고, 이 변경 사항을 다른 모든 **`브로커`**에게 전파
4. 이 정보가 전파되는 데 걸리는 아주 짧은 시간 동안 클러스터는 **일시적으로 불일치 상태**
5. 전파가 완료되면 클러스터 전체가 새로운 토픽의 존재를 인지하게 되며, **최종적 일관성** 달성
	- 최종적으로 모든 브로커는 모든 토픽에 대해 알게 될 것이지만, 정확히 언제가 될 지에 대해서는 아무런 보장도 할 수 없다
### 5.1.2 옵션
- `AdminClient`의 각 메서드들은 특정한 `Options` 객체를 인수로 받는다
	- 브로커가 요청을 어떻게 처리할지에 대해 서로 다른 설정을 담는다
### 5.1.3 수평 구조
- 모든 어드민 작업은 `KafkaAdminClient`에 구현된 아파치 카프카 프로토콜 사용하여 이루어진다
- 어드민 작업 프로그램적으로 수행하는 방법 알아야 할 때 `JavaDoc` 문서에서 필요한 메서드 하나 찾아서 쓰기만 해도 되고, IDE에서 간편하게 자동 완성 해준다
### 5.1.4 추가 참고 사항
- 클러스터의 상태를 변경하는 모든 작업 (`create`, `delete`, `alter`)은 컨트롤러에 의해 수행된다.
- 대부분 어드민 작업은 `AdminClient`를 통해 수행되거나 주키퍼에 저장된 메타데이터를 직접 수정하는 방식으로 이루어진다.
	- **주키퍼를 직접 수정하는 것은 하지 말 것을 권장**
	- **가까운 시일 내 주키퍼 의존성 완전히 제거 된다**
- `AdminClient API`는 카프카 클러스터 내부 바뀌어도 남아 잇다.
## 5.2 AdminClient 사용법: 생성, 설정, 닫기
- `AdminClient` 사용하기 위해서는 가장 먼저 `AdminClient` 객체를 생성해야 한다
```java
// java
import java.time.Duration;
import java.util.Properties;
import org.apache.kafka.clients.admin.AdminClient;
import org.apache.kafka.clients.admin.AdminClientConfig;

public class AdminClientSample {
  public static void main(String[] args) {
    Properties props = new Properties();
    props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");

    AdminClient adminClient = AdminClient.create(props);

    // TODO: AdminClient 를 사용하여 필요한 작업 수행

    adminClient.close(Duration.ofSeconds(30));
  }
}
```
```python
# python (confluent-kafka-python 사용)
from confluent_kafka.admin import AdminClient

# confluent-kafka-python 에서는 설정 키가 Java의 설정 문자열과 거의 동일합니다.
# 'bootstrap_servers' (언더스코어)가 아닌 'bootstrap.servers' (점)를 사용합니다.
config = {
    'bootstrap.servers': 'localhost:9092'
}

try:
    # AdminClient 인스턴스 생성
    admin_client = AdminClient(config)

    # TODO: AdminClient를 사용하여 필요한 작업 수행
    print("AdminClient가 성공적으로 생성되었습니다.")

    # 예시: 클러스터의 토픽 목록 조회
    # list_topics()는 ClusterMetadata 객체를 반환합니다. timeout 설정이 가능합니다.
    cluster_metadata = admin_client.list_topics(timeout=10)
    
    print("\n클러스터의 토픽 목록:")
    # metadata.topics는 토픽 정보를 담은 딕셔너리입니다.
    for topic in cluster_metadata.topics.keys():
        print(f"- {topic}")

except Exception as e:
    print(f"오류 발생: {e}")
```
- `create` 메서드는 설정값 담고 있는  `Properties` 객체를 인수로 받는다
- 프로덕션 환경에서는 브로커 중 하나에 장애가 발생할 경우를 대비하여 최소 3개 이상의 브로커 지정해야 한다.
- `AdminClient`를 시작했으면 `close`로 닫아야 한다
	- `close`를 호출하면 아직 진행중인 작업 있을 수 있기 때문에 타임아웃 매개변수를 받는다
	- `timeout` 발생하면 클라이언트는 모든 진행 중인 작동 멈추고 모든 자원을 해제
	- `timeout` 없이 `close` 호출하면 얼마가 되었든 모든 진행 중인 작업이 완료될 때까지 대기한다
#### `Kafka-python`과 `confluent-kafka-python`의 `AdminClient` 차이점
- **설정 키 (Configuration Keys):** 
	- `kafka-python`에서는 `bootstrap_servers`와 같이 파이썬 스타일의 변수명(snake_case)을 사용했지만, `confluent-kafka-python`은 Java 클라이언트와 거의 동일한 **`bootstrap.servers`**처럼 점(`.`)으로 구분된 문자열 키를 사용
	- 이 방식 덕분에 Java나 다른 언어의 카프카 설정 예제를 거의 그대로 가져와 사용 가능
- **동작 방식:**
	- 내부적으로 C 라이브러리인 `librdkafka`를 사용하여 동작하므로 매우 빠르고 효율적. 
	- `list_topics()`와 같은 메서드는 요청을 보내고 응답을 기다린 후 결과를 반환하는 동기식 메서드처럼 보이지만, 내부적으로는 비동기 통신을 효율적으로 처리
- **리소스 관리 (`close`):** 
	- `kafka-python`의 `AdminClient`와 달리 `confluent-kafka-python`의 `AdminClient`는 **별도의 `close()` 메서드나 `with` 구문을 사용하지 않음.** 
	- 객체가 생성될 때 필요한 연결을 관리하고, 작업이 끝나면 파이썬의 가비지 컬렉터에 의해 자연스럽게 정리 → 코드가 더 간결해진다
### 5.2.1 `client.dns.lookup`
- 기본적으로 카프카는 부트스트랩 서버 설정에 포함된 호스트명을 기준으로 연결을 검증, 해석, 생성
	- 브로커로부터 호스트 정보를 받은 뒤부터는 `advertised.listeners` 설정에 있는 호스트명을 기준으로 연결
- 이 모델은 대부분의 경우 제대로 동작하지만, 2가지 유의할 점이 있다.
	- DNS 별칭 (alias) 를 사용할 경우
	- 2개 이상의 IP 주소로 연결되는 하나의 DNS 항목을 사용할 경우
#### 5.2.1.1 DNS 별칭 (Alias) 을 사용하는 경우
- 일정한 네이밍 컨벤션을 따르는 브로커들이 있다면, 모든 브로커들을 부트스트랩 서버 설정에 일일이 지정하는 것보다 **이 모든 브로커 전체를 가리킬 하나의 DNS alias 를 만들어 관리하는 것이 더 편리**
	- **어떤 브로커가 클라이언트와 처음으로 연결될지는 그리 중요하지 않기 때문에 부트스트래핑을 위해 `all-brokers.hostname.com` 사용 가능**
- 이는 매우 편리하나, **SASL(Simple Authentication and Security Layer) 을 사용하여 인증을할 때 문제 발생**
	- SASL 을 사용할 경우 클라이언트는 `all-brokers.hostname` 에 대해 인증을 하려고 하는데, 서버의 보안 주체 (principal) 는 `broker2.hostname.com` 이기 때문
	- 호스트명이 일치하지 않을 경우 악의적인 공격일 수도 있기 때문에 SASL은 인증을 거부하고 연결도 실패한다
- **`client.dns.lookup=resolve_canonical_bootstrap_servers_only`**
	- 클라이언트는 DNS alias 에 포함된 모든 브로커 이름을 일일이 부트스트랩 서버 목록에 넣어준 것과 동일하게 작동
#### 5.2.1.2 다수의 IP 주소로 연결되는 DNS 이름 사용하는 경우
- 네트워크 아키텍처에서 모든 브로커를 프록시나 로드 밸런서 뒤로 숨기는 것은 매우 흔하다
	- e.g. 외부로부터 연결을 허용하기 위해 로드 밸런서를 두어야 하는 쿠버네티스를 사용하는 경우
- 로드 밸런서가 단일 장애점이 되는 것을 원치 않기 때문에, `broker1.hostname.com`을 여러 개의 IP 로 연결하는 것
	- 이 IP 주소들은 모두 로드 밸런서로 연결되고 따라서 모든 트래픽이 동일한 브로커로 전달
	- IP 주소들은 시간이 지남에 따라 변경될 수도 있다
- 기본적으로 카프카 클라이언트는 해석된 첫 번째 호스트명으로 연결을 시도
	- **다시 말하면 해석된 IP 주소가 사용 불가능일 경우 브로커가 정상임에도 불구하고 클라이언트는 연결에 실패할 수도 있다**
- **`client.dns.lookup=use_all_dns_ips`**
	- **클라이언트가 로드 밸런싱 계층의 고가용성을 충분히 활용할 수 있도록 해당 부분 설정**
### 5.2.2 `request.timeout.ms`
- `request.timeout.ms`은 애플리케이션이 `AdminClient`의 응답을 기다릴 수 있는 시간의 최대값을 정의한다
	- 클라이언트가 재시도가 가능한 에러를 받고 재시도하는 시간 포함
	- 기본 값은 `120s`
	- 각각의 `AdminClient` 메서드는 해당 메서드에만 해당하는 타임아웃 값을 포함하는 `Options` 객체를 받는다.
- **만일 애플리케이션에서 `AdminClient` 작업이 주요 경로 (critical path) 상에 있을 경우, 타임아웃 값을 낮게 잡아준 뒤 제 시간에 리턴되지 않는 응답은 조금 다른 방식으로 처리**해야 할 수도 있다.
	- 서비스가 시작될 때 특정한 토픽이 존재하는지 확인
	- 브로커가 응답하는데 30s 이상 걸릴 경우, 확인 작업을 건너뛰거나 일단 서버 기동을 계속한 뒤 나중에 토픽의 존재 확인
## 5.3 필수적인 토픽 관리 기능
- `AdminClient`의 가장 흔한 활용 사례: 토픽 관리
	- 토픽 목록 조회, 상세 내역 조회, 생성 및 삭제
```java
// java
// 클러스터에 있는 토픽 목록 조회
ListTopicsResult topics = adminClient.listTopics(); // Future 객체들을 ListTopicsResult 객체 리턴
topics.names().get().forEach(System.out::println);
```
```Python
# python
from confluent_kafka.admin import AdminClient

config = {
    'bootstrap.servers': 'localhost:9092'
}

try:
    admin_client = AdminClient(config)

    # 클러스터에 있는 토픽 메타데이터 조회 (timeout: 10초)
    # Java의 ListTopicsResult와 달리, ClusterMetadata 객체를 직접 반환합니다.
    metadata = admin_client.list_topics(timeout=10)

    print("클러스터 토픽 목록:")
    
    # metadata.topics는 토픽 이름(key)과 토픽 메타데이터(value)를 담은 딕셔너리입니다.
    # 딕셔너리의 key들을 순회하며 토픽 이름을 출력합니다.
    for topic in metadata.topics:
        print(topic)

except Exception as e:
    print(f"오류 발생: {e}")
```
- **`adminClient.listTopics()` 는 `Future` 객체들을 감싸고 있는 `ListTopicsResult` 객체를 리턴**
- 특정 토픽이 존재하는지 확인하는 방법 중 하나는 모든 토픽의 목록을 받은 후 원하는 토픽이 그 안에 있는지 확인
	- **큰 클러스터에서 이 방법은 비효율적**일 수 있다
	- 때로는 단순히 토픽의 존재 여부 뿐 아니라 **해당 토픽이 필요한 만큼의 파티션과 레플리카키를 갖고 있는지 확인하는 등 그 이상의 정보가 필요**할 수 있다.
- 예를 들어 카프카 커넥트와 컨플루언트의 스키마 레지스트리는 설정을 저장하기 위해 카프카 토픽을 사용하는데, 이들은 처음 시작 시 아래 조건을 만족하는 설정 토픽이 있는지 확인
	- 하나의 파티션을 갖는다. 이것은 설정 변경에 온전한 순서 부여 위해 필요하다.
	- 가용성을 보장하기 위해 3개의 레플리카를 갖는다
	- 오래된 설정값도 계속해서 저장되도록 토픽에 압착 설정이 되어 있다.
#### 토픽 목록 조회 및 생성
```java
// (이 코드를 실행하기 위한 AdminClient 객체 'admin'과 상수들이 미리 선언되어 있다고 가정.)

// 목표: 데모 토픽이 존재하는지 확인하고, 만약 없다면 생성합니다.
// 많은 카프카 애플리케이션은 특정 토픽이 반드시 존재해야 하므로,
// 프로그램을 시작할 때 이처럼 토픽의 존재 여부를 미리 확인하고 생성하는 로직을 넣어 안정성을 높입니다.
DescribeTopicsResult demoTopic = admin.describeTopics(TOPIC_LIST);
try {
	topicDescription = demoTopic.values().get(TOPIC_NAME).get();
	System.out.println("데모 토픽의 상세 정보:" + topicDescription);

	if (topicDescription.partitions().size() != NUM_PARTITIONS) {
		System.out.println("토픽의 파티션 수가 올바르지 않습니다.");
		//System.exit(-1);
	}
} catch (ExecutionException e) {
	// 예외 처리: '토픽을 찾을 수 없다'는 특정 예외 외에는 모두 심각한 문제로 간주하여 조기에 종료합니다.
	// 만약 네트워크 오류나 인증 실패 등 다른 예외가 발생하면, 프로그램을 즉시 중단시키는 것이 안전합니다.
	if (! (e.getCause() instanceof UnknownTopicOrPartitionException)) {
		e.printStackTrace();
		throw e;
	}

	// 여기까지 코드가 실행되었다면, 'try' 블록에서 발생한 예외가 'UnknownTopicOrPartitionException'이었음을 의미합니다.
	// 즉, 토픽이 존재하지 않는 것이 확실하므로 이제부터 토픽을 생성하는 작업을 시작합니다.
	System.out.println("토픽 " + TOPIC_NAME +
			" 이(가) 존재하지 않습니다. 지금부터 생성을 시작합니다.");
	
	// NewTopic 객체 생성 시 파티션 수와 복제본 수는 선택 사항(optional)입니다.
	// 만약 이 값들을 코드에 명시하지 않으면, 카프카 서버(브로커)에 설정된 기본값 (num.partitions, default.replication.factor)이 자동으로 사용됩니다.
	CreateTopicsResult newTopic = admin.createTopics(Collections.singletonList(
			new NewTopic(TOPIC_NAME, NUM_PARTITIONS, REP_FACTOR)));

	// 테스트용 코드: 아래 코드의 주석을 해제하면 토픽이 브로커의 기본 파티션 수로 생성되는 것을 테스트해 볼 수 있습니다.
	// 이 경우, 아래의 파티션 수 검증 로직에서 에러가 발생할 수 있습니다.
	//CreateTopicsResult newTopic = admin.createTopics(Collections.singletonList(new NewTopic(TOPIC_NAME, Optional.empty(), Optional.empty())));

	// 검증: 토픽이 올바르게 생성되었는지 최종적으로 확인합니다.
	// 토픽 생성을 '요청'한 후에, 그 결과가 정말 우리가 원했던 대로 처리되었는지 검증하는 단계입니다.
	if (newTopic.numPartitions(TOPIC_NAME).get() != NUM_PARTITIONS) {
		System.out.println("토픽이 잘못된 파티션 수로 생성되었습니다. 프로그램을 종료합니다.");
		// System.exit(-1);
	}
}
```
1. **정확한 설정을 갖는 토픽이 존재하는지 확인**하려면 확인하려는 토픽의 목록을 인자로 넣어서 **`describeTopics()` 메서드를 호출**함
	- 리턴되는 **`DescribeTopicsResult` 객체 안에는 토픽 이름을 key 로, 토픽에 대한 상세 정보를 담는 Future 객체를 value 로 하는 맵**이 들어있음

2. **`Future` 가 완료될 때까지 기다린다면 `get()` 을 사용하여 원하는 결과물 (여기선 `TopicDescription`) 을 얻을 수 있음**
	- 하지만 서버가 요청을 제대로 처리하지 못할수도 있음
		- e.g.) 토픽이 존재하지 않은 경우 서버가 상세 정보를 응답으로 보내줄 수도 없음  
			- 이 경우 서버는 에러를 리턴할 것이고, `Future` 는 _ExecutionException_ 을 발생
	- 예외의 `cause` 에 들어있는 것이 실제 서버가 리턴한 실제 에러
	- 여기선 토픽이 존재하지 않을 경우를 처리하고 싶은 것 (= 토픽이 존재하지 않으면 토픽 생성) 이므로 _ExecutionException_ 예외를 처리해주어야 한다.

3. **토픽이 존재할 경우 `Future` 객체는 토픽에 속한 모든 파티션의 목록을 담은 `TopicDescription` 을 리턴**
	- **`TopicDescription` 엔 파티션별로 어느 브로커가 리더이고, 어디에 레플리카가 있고, ISR(In-Sync Replica) 가 무엇인지까지 포함**
	- 주의할 점은 토픽의 설정은 포함되지 않는다는 점임

4. **모든 `AdminClient` 의 result 객체는 카프카가 에러 응답을 보낼 경우 _ExecutionException_ 을 발생**
	- `AdminClient` 가 리턴한 객체가 `Future` 객체를 포함하고, `Future` 객체는 다시 예외를 포함하고 있기 때문임
	- **카프카가 리턴한 에러를 열어보려면 항상 _ExecutionException_ 의 `cause` 를 확인**

5. **토픽이 존재하지 않을 경우 새로운 토픽 생성**  
	- 토픽 생성 시 토픽의 이름만 필수이고, 파티션 수과 레플리카수는 선택사항
	- 만일 이 값들을 지정하지 않으면 카프카 브로커에 설정된 기본값 사용

6. **토픽이 정상적으로 생성되었는지 확인**  
	- 여기서는 파티션의 수를 확인하고 있음  
		- `CreateTopic` 의 결과물을 확인하기 위해 `get()` 을 다시 호출하고 있기 때문에 이 메서드가 예외를 발생시킬 수 있음  
	- 이 경우 _TopicExistsException_ 이 발생하는 것이 보통이며, 이것을 처리해야 한다.
		- 보통은 설정을 확인하기 위해 토픽 상세 내역을 조회함으로써 처리함
#### 토픽 삭제
```java
// 토픽이 완전히 삭제되었는지 확인합니다.
// 그러나 토픽 삭제는 비동기(async) 방식으로 동작하기 때문에,
// 이 코드를 실행하는 시점에는 토픽이 아직 남아있을 가능성이 있습니다.
try {
	// 이 코드는 토픽 정보를 얻는 것이 목적이 아니라,
	// 토픽이 존재하지 않을 때 발생하는 예외(Exception)를 받기 위한 것입니다.
	demoTopic.values().get(TOPIC_NAME).get(); 
	
	// 만약 이 라인까지 코드가 실행된다면, 예외가 발생하지 않은 것이므로 토픽이 아직 삭제되지 않은 것입니다.
	System.out.println("토픽 " + TOPIC_NAME + " 이(가) 아직 남아있습니다.");
} catch (ExecutionException e) {
	// `.get()` 호출 시 ExecutionException이 발생했다면 (특히 원인이 UnknownTopicOrPartitionException일 경우),
	// 토픽을 찾지 못했다는 뜻이므로 성공적으로 삭제된 것입니다.
	System.out.println("토픽 " + TOPIC_NAME + " 이(가) 삭제되었습니다.");
}
```
- 삭제할 토픽 리스트를 인자로 하여 `deleteTopics()` 메서드를 호출한 뒤 `get()` 을 호출해서 작업이 끝날 때까지 기다린다.
	- **카프카에서 삭제된 토픽은 복구가 불가능하기 때문에 데이터 유실**이 발생할 수 있으므로 토픽을 삭제할 때는 특별히 주의
#### 예외 상황: 커프카가 응답할 때까지 서버 스레드가 블록되는 것을 원하지 않을 때 (비동기적)
- 지금까지 모두 서로 다른 **`AdminClient` 메서드가 리턴하는 `Future` 객체에 블로킹 방식으로 동작하는 `get()` 메서드 호출**
- 하지만 예외 케이스가 하나 있는데 바로 많은 어드민 요청을 처리할 것으로 예상되는 서버를 개발하는 경우
	- 이 경우 카프카가 응답할 때까지 기다리는 동안 서버 스레드가 블록되는 것보다는, 사용자로부터 계속해서 요청을 받고, 카프카로 요청을 보내고, 카프카가 응답하면 그 때 클라이언트로 응답을 보내는 것이 더욱합리적
- 이럴 때 `KafkaFuture`의 융통성은 도움이 된다
```java
// Vert.x를 사용하여 HTTP 서버를 생성
vertx.createHttpServer()
    // 들어오는 모든 HTTP 요청을 처리할 핸들러를 등록
    // request 객체에는 요청에 대한 모든 정보가 담겨 있다.
    .requestHandler(request -> {
        // HTTP 요청에서 'topic' 파라미터 값을 추출. (예: http://localhost:8080/?topic=demo-topic)
        String topic = request.getParam("topic");
        // HTTP 요청에서 'timeout' 파라미터 값을 추출 (예: &timeout=5000)
        String timeout = request.getParam("timeout");

        // timeout 값을 정수(int)로 변환. 값이 없거나 숫자가 아니면 기본값으로 1000ms를 사용
        int timeoutMs = NumberUtils.toInt(timeout, 1000);

        // AdminClient를 사용하여 특정 토픽의 상세 정보 조회를 '요청'. 이 작업은 비동기적으로 시작
        DescribeTopicsResult demoTopic = admin.describeTopics(
                // describeTopics는 토픽 이름의 리스트를 인자로 받으므로, 파라미터로 받은 topic 하나만 담은 리스트를 생성
                Collections.singletonList(topic),
                // 이 특정 요청에 대한 타임아웃 값을 옵션으로 지정
                new DescribeTopicsOptions().timeoutMs(timeoutMs));

        // describeTopics의 결과(KafkaFuture)가 완료되었을 때 실행될 콜백 함수를 등록
        // 이 .whenComplete() 메서드 덕분에 Vert.x의 이벤트 루프를 막지 않고(non-blocking) 비동기적으로 결과를 처리
        demoTopic.values().get(topic).whenComplete(
                // 콜백 함수는 BiConsumer 인터페이스를 구현하며, 성공 결과(topicDescription)와 실패 결과(throwable)를 인자로 받는다
                // 둘 중 하나는 항상 null.
                new KafkaFuture.BiConsumer<TopicDescription, Throwable>() {
                    @Override
                    public void accept(final TopicDescription topicDescription,
                                       final Throwable throwable) {
                        // 만약 throwable 객체가 null이 아니라면, 작업 중 오류가 발생한 것
                        if (throwable != null) {
                            System.out.println("got exception");
                            // 오류가 발생했을 경우, 클라이언트에게 에러 메시지를 담아 HTTP 응답을 전송
                            request.response().end("Error trying to describe topic "
                                    + topic + " due to " + throwable.getMessage());
                        } else {
                            // 작업이 성공했을 경우, 받아온 TopicDescription 객체를 문자열로 변환하여 클라이언트에게 HTTP 응답으로 보낸다
                            request.response().end(topicDescription.toString());
                        }
                    }
                });
    })
    // 마지막으로, 생성된 HTTP 서버를 8080 포트에서 실행(listen)
    .listen(8080);
```
- 여기에서 중요한 것은, **카프카 클러스터가 응답할 때까지 기다리지 않는다는 것이다 (완전한 비동기 처리)**
	- **Kafka AdminClient 호출:** `describeTopics`는 요청만 보내고 바로 다음 코드로 넘어간다
## 5.4 설정 관리
- **설정 관리는 `ConfigResource` 객체 사용**
- 설정 가능한 자원
	- 브로커
	- 브로커 로그
	- 토픽
- 브로커와 브로커 로깅 설정은 `kafka-configs.sh` 혹은 다른 카프카 관리 툴을 사용하는 것이 보통이지만, 애플리케이션에서 사용하는 토픽의 설정을 확인하거나 수정하는 것은 상당히 빈번
- 예를 들어 **많은 애플리케이션들은 정확한 작동을 위해 압착 설정이 된 토픽을 사용**
	-  경우 애플리케이션이 주기적으로 해당 토픽에 실제로 압착 설정이 되어있는지를 확인한 후 (보존 기한 기본값보다 짧은 주기로 하는 것이 안전함), 설정이 안되어 있다면 설정을 교정해주는 것이 합리적
```java
import org.apache.kafka.clients.admin.*;
import org.apache.kafka.common.config.ConfigResource;
import org.apache.kafka.common.config.TopicConfig;

import java.util.*;
import java.util.concurrent.ExecutionException;

// (이 코드를 실행하기 위한 AdminClient 객체 'admin'과 상수 TOPIC_NAME이 미리 선언되어 있다고 가정합니다.)
// AdminClient admin = ...;
// String TOPIC_NAME = "my-compacted-topic";

// --- 코드 시작 ---

// 목표: 특정 토픽의 정책을 '압축(compact)'으로 설정합니다.
// 로그 압축(Log Compaction)은 최신 key 값만 유지하여 토픽의 전체 크기를 관리하는 Kafka의 주요 기능 중 하나입니다.

// 1. 설정을 조회하거나 변경할 대상을 지정하는 객체를 생성합니다.
// 여기서는 'TOPIC' 타입의 'TOPIC_NAME'이라는 이름을 가진 리소스를 지정합니다.
ConfigResource configResource = new ConfigResource(ConfigResource.Type.TOPIC, TOPIC_NAME);

// 2. 지정된 리소스(토픽)의 현재 설정 정보를 비동기적으로 요청합니다.
DescribeConfigsResult configsResult = admin.describeConfigs(Collections.singleton(configResource));

// 3. 요청이 완료될 때까지 기다린 후(blocking), 해당 토픽의 설정(Config) 객체를 가져옵니다.
Config configs = configsResult.all().get().get(configResource);

// (디버깅용) 브로커의 기본값이 아닌, 사용자가 특별히 설정한 값들만 출력합니다.
System.out.println("--- Non-default configs for topic " + TOPIC_NAME + " ---");
configs.entries().stream()
        .filter(entry -> !entry.isDefault()) // 모든 설정 항목(entries) 중에서, 기본값(isDefault)이 아닌 항목만 필터링합니다.
        .forEach(System.out::println);
System.out.println("----------------------------------------------------");

// 4. 토픽의 정책이 이미 '압축(compact)'으로 설정되어 있는지 확인합니다.
// 먼저, 우리가 원하는 설정값인 'cleanup.policy=compact'에 해당하는 ConfigEntry 객체를 생성합니다.
ConfigEntry compaction = new ConfigEntry(TopicConfig.CLEANUP_POLICY_CONFIG, TopicConfig.CLEANUP_POLICY_COMPACT);

// 현재 토픽의 설정 목록에 위에서 정의한 '압축' 설정이 포함되어 있지 않다면, 설정을 변경하는 로직을 실행합니다.
if (!configs.entries().contains(compaction)) {
    System.out.println("Topic " + TOPIC_NAME + " is not compacted. Altering config now...");
    
    // 5. 토픽이 압축 상태가 아니라면, 압축하도록 설정을 변경합니다.
    
    // 변경할 설정 작업(Operation)들을 담을 컬렉션을 생성합니다.
    Collection<AlterConfigOp> configOp = new ArrayList<>();
    // 'cleanup.policy=compact' 설정을 'SET'(설정)하는 작업을 추가합니다.
    configOp.add(new AlterConfigOp(compaction, AlterConfigOp.OpType.SET));

    // 어떤 리소스에 어떤 변경을 적용할지를 담는 Map을 생성합니다.
    Map<ConfigResource, Collection<AlterConfigOp>> alterConfigs = new HashMap<>();
    // 우리가 지정한 토픽 리소스에 방금 만든 설정 변경 작업을 연결합니다.
    alterConfigs.put(configResource, configOp);

    // AdminClient를 통해 설정 변경을 요청하고, 작업이 완료될 때까지 기다립니다.
    // incrementalAlterConfigs는 다른 설정은 그대로 두고 지정된 항목만 변경/추가합니다.
    admin.incrementalAlterConfigs(alterConfigs).all().get();
    
    System.out.println("Topic " + TOPIC_NAME + " is now compacted.");

} else {
    // 6. '압축' 설정이 이미 적용되어 있는 경우에 해당합니다.
    System.out.println("Topic " + TOPIC_NAME + " is already a compacted topic.");
}
```
1. **`ConfigResource` 로 특정한 토픽의 설정을 확인**
	- 하나의 요청에 여러 개의 서로 다른 유형의 자원을 지정할 수 있음

2. **`describeConfigs()` 의 결과물인 `DescribeConfigsResult` 는 `ConfigResource` 를 key 로, 설정값의 모음을 value 로 갖는 맵**임
	- 각 설정 항목은 해당 값이 기본값에서 변경되었는지 확인할 수 있게 해주는 `isDefault()` 메서드 제공  
	- **토픽 설정이 기본값이 아닌 것으로 취급되는 경우는 2가지 경우**가 있음
	- 사용자가 토픽의 설정값을 기본값이 아닌 것으로 잡아준 경우
	- 브로커 단위 설정이 수정된 상태에서 토픽이 생성되어서 기본값이 아닌 값을 브로커 설정으로부터 상속받은 경우

3. **설정을 변경하기 위해 변경하고자 하는 `ConfigResource` 를 key 로, 바꾸고자 하는 설정값 모음을 value 로 하는 맵을 지정**
	- 각각의 설정 변경 작업은 설정 항목과 작업 유형으로 이루어짐  
	- 위에서 설정 항목은 설정의 이름과 설정값 의미 (여기서는 `cleanup.policy` 가 설정의 이름이고, `compact` 가 설정값)  
	- **카프카에서는 4가지 형태의 설정 변경이 가능**함 **설정값을 잡아주는 `SET` / 현재 설정값을 삭제하고 기본값으로 되돌리는 `DELETE` / `APPEND` / `SUBSTRACT`**  
		- `APPEND` 와 `SUBSTRACT` 는 목록 형태의 설정에만 사용 가능하며, 이걸 사용하면 전체 목록을 주고받을 필요 없이 필요한 설정만 추가/삭제 가능
## 5.5 컨슈머 그룹 관리
- **카프카는 이전의 데이터를 읽어서 처리한 것과 완전히 동일한 순서로 데이터를 재처리**할 수 있다.
	-  컨슈머 API 를 사용하여 처리 지점을 되돌려서 오래된 메시지를 다시 토픽으로부터 읽어올 수 있다
	- 이렇게 API 사용한다는 것 자체가 애플리케이션에 데이터 재처리 기능을 미리 구현해놓았다는 의미
- `AdminClient` 를 사용하여 프로그램적으로 컨슈머 그룹과 이 그룹들이 커밋한 오프셋을 조회,수정하는 방법 확인 예정
### 5.5.1 컨슈머 그룹 살펴보기
- 컨슈머 그룹 살펴보고 변경하고 싶다면, 컨슈머 그룹의 목록을 가장 먼저 조회해야 한다
```java
// 컨슈머 그룹 조회
adminClient.listConsumerGroups().valid().get().forEach(System.out::println);
```
- **`valid()` 와 `get()` 메서드를 호출함으로써 리턴되는 모음은 클러스터가 에러없이 리턴한 컨슈머 그룹만을 포함**한다는 것에 주의해야 한다
	- 이 과정에서 발생한 에러가 예외의 형태로 발생하지는 않는데, `errors()` 메서드를 사용하여 모든 예외를 가져올 수 있다
```java
// 특정 컨슈머 그룹에 대한 상세한 정보
ConsumerGroupDescription groupDescription = admin
		.describeConsumerGroups(CONSUMER_GRP_LIST)
		.describedGroups().get(CONSUMER_GROUP).get();
System.out.println("Description of group " + CONSUMER_GROUP
		+ ":" + groupDescription);
```
- **`ConsumerGroupDescription` 는 아래와 같이 해당 그룹에 대한 상세한 정보**를 가진다.
	- **이러한 정보들은 트러블 슈팅을 할 때 매우 유용**
		- 그룹 멤버와 멤버별 식별자
		- 호스트명
		- 멤버별로 할당된 파티션
		- 할당 알고리즘
		- 그룹 코디네이터의 호스트명
- **컨슈머 그룹이 읽고 있는 각 파티션에 대해 마지막으로 커밋된 오프셋 값이 무엇인지, 최신 메시지에서 얼마나 뒤떨어졌는지 (=lag) 에 대한 정보는 없다.**
#### `AdminClient`를 사용해서 커밋 정보 얻어오는 방법
```java
// 1. 특정 컨슈머 그룹이 커밋한 오프셋 정보를 가져옵니다.
//    이 정보는 "컨슈머 그룹이 각 파티션을 어디까지 읽었는지"를 나타냅니다.
Map<TopicPartition, OffsetAndMetadata> offsets =
        admin.listConsumerGroupOffsets(CONSUMER_GROUP) // 비동기적으로 오프셋 조회를 요청합니다.
                .partitionsToOffsetAndMetadata().get(); // 요청이 완료될 때까지 기다린 후, 결과를 Map 형태로 받습니다.

// 2. 각 파티션의 다양한 시점의 오프셋을 조회하기 위한 요청 Map들을 생성합니다.
Map<TopicPartition, OffsetSpec> requestLatestOffsets = new HashMap<>(); // 가장 최신 오프셋 (Log End Offset)
Map<TopicPartition, OffsetSpec> requestEarliestOffsets = new HashMap<>(); // 가장 오래된 오프셋 (Log Start Offset)
Map<TopicPartition, OffsetSpec> requestOlderOffsets = new HashMap<>(); // 특정 시간 기준의 오프셋

// 특정 시간(여기서는 2시간 전)을 기준으로 오프셋을 조회하기 위한 시간 객체를 생성합니다.
DateTime resetTo = new DateTime().minusHours(2);

// 3. 위에서 조회한 '커밋된 오프셋이 있는 모든 파티션'에 대해,
//    1) 가장 최신 오프셋(latest), 2) 가장 오래된 오프셋(earliest), 3) 2시간 전 시점의 오프셋을 조회하기 위한 요청을 준비합니다.
//    (참고) 이 예제 코드에서는 2시간 전 오프셋(requestOlderOffsets)을 실제로 사용하지는 않지만,
//    컨슈머 오프셋을 특정 시간으로 리셋하는 다른 기능에서 활용할 수 있습니다.
for (TopicPartition tp : offsets.keySet()) {
    // 현재 파티션(tp)의 '가장 최신' 오프셋을 요청하는 명세(OffsetSpec)를 추가합니다.
    requestLatestOffsets.put(tp, OffsetSpec.latest());
    // 현재 파티션(tp)의 '가장 오래된' 오프셋을 요청하는 명세를 추가합니다.
    requestEarliestOffsets.put(tp, OffsetSpec.earliest());
    // 현재 파티션(tp)의 '2시간 전' 시점의 오프셋을 요청하는 명세를 추가합니다.
    requestOlderOffsets.put(tp, OffsetSpec.forTimestamp(resetTo.getMillis()));
}

// 4. 준비된 요청 중 '가장 최신 오프셋' 요청 Map을 사용하여 실제 오프셋 정보를 가져오고, 완료될 때까지 기다립니다.
//    이 정보는 "각 파티션의 맨 끝이 어디인지"를 나타냅니다.
Map<TopicPartition, ListOffsetsResult.ListOffsetsResultInfo> latestOffsets =
        admin.listOffsets(requestLatestOffsets).all().get();

// 5. 이제 각 파티션별로 '커밋된 오프셋'과 '최신 오프셋'을 비교하여 랙(lag)을 계산하고 출력합니다.
for (Map.Entry<TopicPartition, OffsetAndMetadata> e : offsets.entrySet()) {
    String topic = e.getKey().topic();
    int partition = e.getKey().partition();
    // 컨슈머 그룹이 마지막으로 읽고 커밋한 위치(오프셋)
    long committedOffset = e.getValue().offset();
    // 해당 파티션에 가장 마지막으로 저장된 메시지의 위치(오프셋)
    long latestOffset = latestOffsets.get(e.getKey()).offset();
    
    // '최신 오프셋 - 커밋된 오프셋'을 계산하여 컨슈머 그룹이 얼마나 뒤처져 있는지(랙) 출력합니다.
    System.out.println("컨슈머 그룹 " + CONSUMER_GROUP
            + "은(는) " + topic + " 토픽 " + partition + " 파티션에 "
            + committedOffset + " 오프셋을 커밋했습니다. "
            + "해당 파티션의 최신 오프셋은 " + latestOffset + " 이므로, "
            + "컨슈머 그룹은 " + (latestOffset - committedOffset) + "개의 레코드만큼 뒤처져 있습니다 (lag).");
}
```
- 컨슈머 그룹이 사용중인 모든 토픽 파티션을 key 로, 각각의 토픽 파티션에 대해 마지막으로 커밋된 오프셋을 value 로 하는 맵 조회  
	- `describeConsumerGroups()` 와 달리 `listConsumerGroupOffsets()` 은 컨슈머 그룹의 모음이 아닌 하나의 컨슈머 그룹을 받음 
-  컨슈머 그룹에서 커밋한 모든 토픽의 파티션에 대해 마지막 메시지의 오프셋, 가장 오래된 메시지의 오프셋, 2시간 전의 오프셋 조회  
	- `OffsetSpec`
		- `latest()`: 해당 파티션의 마지막 레코드의 오프셋 조회
		- `earliest()`: 해당 파티션의 첫 번째 레코드의 오프셋 조회
		- `forTimestamp()`: 해당 파티션의 지정된 시각 이후에 쓰여진 레코드의 오프셋 조회
-  모든 파티션을 반복하여 각각의 파티션에 대해 마지막으로 커밋된 오프셋, 파티션의 마지막 오프셋, 둘 사이의 lag 출력
### 5.5.2 컨슈머 그룹 수정하기
- `AdminClient` 는 컨슈머 그룹을 수정하기 위한 메서드 또한 갖고 있다
	- 그룹 삭제
	- 멤버 제외
	- 커밋된 오프셋 삭제 및 변경 (가장 유용)
#### 오프셋 변경 기능
- 오프셋 삭제는 컨슈머를 맨 처음부터 실행시키는 가장 간단한 방법으로 보일수도 있지만, 이것은 컨슈머 설정에 의존
- 명시적으로 커밋된 오프셋을 맨 앞으로 변경하면 컨슈머는 토픽의 맨 앞에서부터 처리를 시작
	- 즉, **컨슈머가 리셋 된다**
- **오프셋 토픽의 오프셋 값을 변경한다고 해도 컨슈머 그룹에 변경 여부는 전달되지 않는다는 점을 명심하자**
	- 컨슈머 그룹은 컨슈머가 새로운 파티션을 할당받거나, 새로 시작할 때만 오프셋 토픽에 저장된 값을 읽어올 뿐이다
	- 컨슈머가 모르는 오프셋 변경을 방지하지 위해 카프카에서는 현재 작업이 돌아가고 있는 컨슈머 그룹에 대한 오프셋을 수정하는 것을 허용하지 않는다
- **상태를 가지고 있는 컨슈머 애플리케이션에서 오프셋을 리셋하고, 해당 컨슈머 그릅이 토픽의 맨 처음부터 처리를 시작하도록 할 경우 저장된 상태가 깨질 수 있다는 점도 명심하자**
- **이런 이유로 상태 저장소를 적절히 변경해 줄 필요가 있다**
	- 개발 환경이라면 상태 저장소를 완전히 삭제 → 다음 입력 토픽의 시작점으로 오프셋 리섹하면 된다
```java
// 1. 컨슈머 그룹의 오프셋을 토픽의 가장 처음(beginning)으로 리셋합니다.
//    (참고) `requestOlderOffsets` 맵을 사용하면 2시간 전 시점으로도 리셋을 시도해 볼 수 있습니다.

// '가장 오래된 오프셋' 요청을 실행하여, 각 파티션의 시작 오프셋 정보를 가져옵니다.
Map<TopicPartition, ListOffsetsResult.ListOffsetsResultInfo> earliestOffsets =
        admin.listOffsets(requestEarliestOffsets).all().get();

// 2. 오프셋을 변경(alter)하기 위한 요청 Map을 생성합니다.
//    이 Map은 최종적으로 alterConsumerGroupOffsets 메서드에 전달됩니다.
Map<TopicPartition, OffsetAndMetadata> resetOffsets = new HashMap<>();

// 3. 조회된 '가장 오래된 오프셋' 정보를 바탕으로, 오프셋 리셋 요청 Map을 채웁니다.
for (Map.Entry<TopicPartition, ListOffsetsResult.ListOffsetsResultInfo> e :
        earliestOffsets.entrySet()) {
    // 어떤 파티션의 오프셋을 몇 번으로 리셋할 것인지 로그를 출력합니다.
    System.out.println("토픽-파티션 " + e.getKey() + "의 오프셋을 " + e.getValue().offset() + "(으)로 리셋합니다.");
    // 현재 파티션(e.getKey())을 새로운 오프셋 정보(new OffsetAndMetadata)와 함께 리셋 요청 Map에 추가합니다.
    resetOffsets.put(e.getKey(), new OffsetAndMetadata(e.getValue().offset()));
}

// 4. AdminClient를 사용하여 최종적으로 컨슈머 그룹의 오프셋 변경을 요청합니다.
try {
    // alterConsumerGroupOffsets 메서드를 호출하고, 작업이 완료될 때까지 기다립니다.
    admin.alterConsumerGroupOffsets(CONSUMER_GROUP, resetOffsets).all().get();
    System.out.println("컨슈머 그룹 " + CONSUMER_GROUP + "의 오프셋이 성공적으로 리셋되었습니다.");

} catch (ExecutionException e) {
    // 오프셋 변경 작업 중 발생할 수 있는 예외를 처리합니다.
    System.out.println("컨슈머 그룹 " + CONSUMER_GROUP + "의 커밋된 오프셋을 수정하는 데 실패했습니다. 오류: " + e.getMessage());
    
    // 만약 발생한 예외의 원인이 'UnknownMemberIdException'이라면, 이는 매우 흔한 경우입니다.
    if (e.getCause() instanceof UnknownMemberIdException)
        // 이 오류는 보통 컨슈머 그룹이 현재 실행 중(active)일 때 발생합니다.
        // 오프셋을 강제로 변경하려면, 해당 컨슈머 그룹의 모든 컨슈머를 중지해야 합니다.
        System.out.println("해당 컨슈머 그룹이 현재 활성 상태인지 확인하세요.");
}
```
1. 맨 앞의 오프셋부터 처리를 시작하도록 컨슈머 그룹을 리셋하기 위해 **맨 앞 오프셋의 값 조회**
2.  반복문을 이용하여 `listOffsets()` 가 리턴한 `ListOffsetsResultInfo` 의 맵 객체를 **`alterConsumerGroupOffsets()` 를 호출하는데 필요한 `OffsetAndMetadata` 의 맵 객체로 변환**
3. **`alterConsumerGroupOffsets()` 를 호출한 뒤 `get()` 을 호출하여 `Future` 객체가 작업을 성공적으로 완료할 때까지 기다림**
4. **`alterConsumerGroupOffsets()` 가 실패하는 가장 흔한 이유 중 하나는 컨슈머 그룹을 미리 정지시키지 않아서이다**  
	- 특정 컨슈머 그룹을 정지시키는 어드민 명령은 없기 때문에 컨슈머 애플리케이션을 정지시키는 것 외에는 방법이 없음  
	- 만일 컨슈머 그룹이 여전히 돌아가고 있는 중이라면, 컨슈머 코디네이터 입장에서는 컨슈머 그룹에 대한 오프셋 변경 시도를 그룹의 멤버가 아닌 클라이언트가 시도하려는 것으로 간주함  
	- 이 경우 `UnknownMemberIdException` 예외가 발생
## 5.6 클러스터 메타데이터
- 애플리케이션이 연결된 클러스터에 대한 정보를 명시적으로 읽어와야 하는 경우는 드물다.\
- 카프카 클라이언트는 이러한 정보를 추상화 한다
```java
// Who are the brokers? Who is the controller?
DescribeClusterResult cluster = admin.describeCluster();

System.out.println("Connected to cluster " + cluster.clusterId().get());
System.out.println("The brokers in the cluster are:");
cluster.nodes().get().forEach(node -> System.out.println("    * " + node));
System.out.println("The controller is: " + cluster.controller().get());

```
## 5.7 고급 어드민 작업
- 잘 쓰이지도 않고 위험하기도 하지만 필요할 때 사용하면 매우 유용한 메서드
### 5.7.1 토픽에 파티션 추가하기
- **대체로 토픽의 파티션 수는 토픽이 생성될 때 결정**
	- 토픽에 파티션을 추가하는 경우는 드물기도 하고 위험하다
	- 현재 파티션이 처리할 수 있는 최대 처리량까지 부하가 차서 파티션 추가 외에는 선택지가 없는 경우도 존재한다
- **토픽에 파티션을 추가해야 한다면 파티션을 추가함으로써 토픽을 읽고 있는 애플리케이션들이 깨지지는 않을지 확인**해보아야 한다.
	- `createPartitions()` 메서드를 이용하여 지정된 토픽들에 파티션을 추가 가능
	- 여러 토픽을 한 번에 확장할 경우 **일부 토픽은 성공하고, 나머지는 실패할 수도 있다는 점을 주의**해야 한다.
-  **파티션을 확장하기 전에 토픽 상세 정보를 확인하여 몇 개의 파티션을 갖고 있는지 확인**해야 한다.
	- 토픽의 파티션을 확장할 때는 새로 추가될 파티션 수가 아닌 파티션이 추가된 뒤의 파티션 수를 지정해야 하기 때문
```java
// add partitions
Map<String, NewPartitions> newPartitions = new HashMap<>();
newPartitions.put(TOPIC_NAME, NewPartitions.increaseTo(NUM_PARTITIONS+2));
try {
	admin.createPartitions(newPartitions).all().get();
} catch (ExecutionException e) {
	if (e.getCause() instanceof InvalidPartitionsException) {
		System.out.printf("Couldn't modify number of partitions in topic: " + e.getMessage());
	}
}
```
### 5.7.2 토픽에서 레코드 삭제하기
- 카프카는 토픽에 대해 데이터 보존 정책을 설정할 수 있도록 되어있지만, 법적인 요구 조건을 보장할 수 있을 수준의 기능이 구현되어 있지 않다
- **`deleteRecords()` 메서드는 호출 시점을 기준으로 지정된 오프셋보다 더 오래된 모든 레코드에 삭제 표시를 함으로써 컨슈머가 접근할 수 없도록 한다.**
	- 삭제된 레코드의 오프셋 중 가장 큰 값을 리턴하므로 의도한 대로 삭제가 이루어졌는지 확인 가능
	- 특정 시각 혹은 바로 뒤에 쓰여진 레코드의 오프셋을 가져오기 위해 `listOffsets()` 메서드를 이용 가능
- **`listOffsets()` 메서드와 `deleteRecords()` 메서드를 이용하여 특정 시각 이전에 쓰여진 레코드들을 삭제**할 수 있다.
```java
// delete records
Map<TopicPartition, ListOffsetsResult.ListOffsetsResultInfo> olderOffsets =
		admin.listOffsets(requestOlderOffsets).all().get();
Map<TopicPartition, RecordsToDelete> recordsToDelete = new HashMap<>();
for (Map.Entry<TopicPartition, ListOffsetsResult.ListOffsetsResultInfo>  e:
		olderOffsets.entrySet())
	recordsToDelete.put(e.getKey(), RecordsToDelete.beforeOffset(e.getValue().offset()));
admin.deleteRecords(recordsToDelete).all().get();
```
### 5.7.3 리더 선출
- `elecLeader()` 메서드는 2 가지 서로 다른 형태의 리더 선출을 할 수 있게 해준다.
- 비동기적으로 작동
	- 메서드가 성공적으로 리턴된 후에도 모든 브로커가 새로운 상태에 대해 확인할 때까지는 어느정도 시간 소요
	- `describeTopics()` 메서드 호출 결과 또한 일관적이지 않은 결과물 리턴 가능
	- 다수의 파티션에 대해 리더 선출 작동 → 몇 개는 성공, 나머지는 실패할 수도 있다
#### 선호 리더 선출 (preferred leader election)
- 각 파티션은 선호 리더 (preferred leader) 라 불리는 레플리카를 하나씩 가진다
	- preferred:  모든 파티션이 preferred leader 레플리카를 리더로 삼을 경우 각 브로커마다 할당되는 리더의 개수가 균형을 이루기 때문
- 기본적으로 카프카는 5분마다 선호 리더 레플리카가 실제로 리더를 맡고 있는지 확인하여 리더를 맡을 수 있는데도 맡고 있지 않은 경우 해당 레플리카를 리더로 삼음
- **`auto.leader.rebalance.enable` 가 false 로 설정되어 있거나 아니면 좀 더 빨리 이 과정을 작동시키고 싶다면 `electLeader()` 메서드 호출**
#### 언클린 리더 선출 (unclean leader election)
- 어느 파티션의 리더 레플리카가 사용 불능 상태가 되었는데 다른 레플리카들은 리더를 맡을 수 없는 상황이라면 해당 파티션은 리더가 없게 되고, 따라서 사용 불능 상태가 된다
	- 보통 데이터가 없는 경우이다
- 이 문제를 해결하는 방법 중 하나는 **리더가 될 수 없는 레플리카를 그냥 리더로 삼아버리는 언클린 리더 선출**
	- 데이터 유실을 초래함 (예전 리더에 쓰여졌지만 새 리더로 복제되지 않은 모든 이벤트는 유실되기 때문)
- **`elecLeader()` 메서드 통해 언클린 리더 선출 가능**
```java
// 1. 리더 선출을 수행할 토픽-파티션들을 담을 Set을 생성합니다.
Set<TopicPartition> electableTopics = new HashSet<>();
// 리더 선출 대상으로서 'TOPIC_NAME' 토픽의 0번 파티션을 추가합니다.
electableTopics.add(new TopicPartition(TOPIC_NAME, 0));

// 2. 리더 선출 작업 중 발생할 수 있는 예외를 처리하기 위해 try-catch 블록을 사용합니다.
try {
    // '선호 리더(Preferred Leader)' 선출을 요청하고, 작업이 완료될 때까지 기다립니다.
    // 이 명령은 지정된 파티션의 리더 역할을 해당 파티션의 '선호 리더'에게 넘기도록 시도합니다.
    admin.electLeaders(ElectionType.PREFERRED, electableTopics).all().get();
    System.out.println("선호 리더 선출 작업이 성공적으로 요청되었습니다.");

} catch (ExecutionException e) {
    // 3. 리더 선출 작업이 실패했을 경우 예외를 처리합니다.
    
    // 만약 발생한 예외의 원인이 'ElectionNotNeededException'이라면, 이는 실제 오류가 아닙니다.
    if (e.getCause() instanceof ElectionNotNeededException) {
        // 이 예외는 요청한 모든 파티션의 현재 리더가 이미 '선호 리더'라서,
        // 리더를 변경할 필요가 없다는 의미입니다.
        System.out.println("모든 리더가 이미 선호 리더이므로, 추가 작업이 필요 없습니다.");
    }
}
```
1. **특정 토픽의 한 파티션에 대해 선호 리더 선출**
- 지정할 수 있는 토픽과 파티션 수에는 제한이 없음
- 만일 파티션 모음이 아닌 null (위에서는 `electableTopics` 대신 null) 값을 지정하여 실행할 경우 모든 파티션에 대해 지정된 선출 유형 작업을 시작
2. 클러스터 상태가 좋다면 아무런 작업도 일어나지 않을것임  
- **`electLeaders()` 메서드는 선호 리더 선출과 언클린 리더 선출은 선호 리더가 아닌 레플리카가 현재 리더를 맡고 있을 경우에만 수행**
### 5.7.4 레플리카 재할당
- 레플리카의 현재 위치가 마음에 안 들 때가 있다
	- 브로커에 너무 많은 레플리카가 올라가 있어서 몇 개를 다른 곳으로 옮겨야 할 경우
	- 레플리카를 추가할 경우
	- 장비를 내리기 위해 모든 레플리카를 다른 장비로 내보내야 하는 경우
	- 몇몇 토픽에 대한 요청이 너무 많아서 나머지에서 따로 분리해놓아야 하는 경우
- 이럴 때 **`alterPartitionReassignment()` 메서드를 사용하면 파티션에 속한 각각의 레플리카의 위치를 정밀하게 제어 가능**
	- **레플리카를 하나의 브로커에서 다른 브로커로 재할당하는 일은 브로커 간에 대량의 데이터 복제가 일어날 수 있다는 점을 주의**
```java
// 1. 파티션들을 새로운 브로커로 재할당(reassign)합니다.
//    이 작업은 클러스터의 브로커 간 데이터 균형을 맞추거나, 특정 브로커를 교체할 때 사용됩니다.

// 파티션 재할당 계획을 담을 Map을 생성합니다.
// Key는 재할당할 TopicPartition이고, Value는 새로운 복제본 목록 또는 취소 여부입니다.
Map<TopicPartition, Optional<NewPartitionReassignment>> reassignment = new HashMap<>();

// 'TOPIC_NAME' 토픽의 0번 파티션에 대한 새로운 복제본 위치를 지정합니다.
// 앞으로 이 파티션의 데이터는 브로커 0과 1에 저장되며, 0번 브로커가 선호 리더가 됩니다.
reassignment.put(new TopicPartition(TOPIC_NAME, 0),
        Optional.of(new NewPartitionReassignment(Arrays.asList(0, 1))));

// 'TOPIC_NAME' 토픽의 1번 파티션은 이제 브로커 0에만 저장됩니다.
reassignment.put(new TopicPartition(TOPIC_NAME, 1),
        Optional.of(new NewPartitionReassignment(Arrays.asList(0))));

// 'TOPIC_NAME' 토픽의 2번 파티션은 브로커 1과 0에 저장됩니다. (선호 리더: 1번 브로커)
reassignment.put(new TopicPartition(TOPIC_NAME, 2),
        Optional.of(new NewPartitionReassignment(Arrays.asList(1, 0))));

// 'TOPIC_NAME' 토픽의 3번 파티션에 대해서는 `Optional.empty()`를 지정합니다.
// 이는 현재 이 파티션에 대해 진행 중인 재할당 작업이 있다면 '취소'하라는 의미입니다.
reassignment.put(new TopicPartition(TOPIC_NAME, 3),
        Optional.empty());

// 2. 위에서 정의한 재할당 계획을 클러스터에 전달하여 실행을 요청합니다.
try {
    // alterPartitionReassignments 메서드를 호출하고, 요청이 수락될 때까지 기다립니다.
    // (참고) 이 호출은 데이터 복제가 완료될 때까지 기다리는 것이 아니라, 재할당 '작업'이 시작될 때까지만 기다립니다.
    admin.alterPartitionReassignments(reassignment).all().get();

} catch (ExecutionException e) {
    // 3. 재할당 요청 중 발생할 수 있는 예외를 처리합니다.
    
    // 만약 예외의 원인이 'NoReassignmentInProgressException'이라면, 이는 심각한 오류가 아닙니다.
    if (e.getCause() instanceof NoReassignmentInProgressException) {
        // 이 예외는 진행 중이지 않은 재할당 작업을 취소하려고 할 때 발생하므로, 무시하고 계속 진행하겠다는 메시지를 출력합니다.
        System.out.println("진행 중이 아닌 재할당 작업을 취소하려고 했으므로, 이 메시지는 무시합니다.");
    }
}

// 4. 현재 진행 중인 재할당 상태를 모니터링합니다.
// 재할당 작업은 백그라운드에서 오랜 시간 동안 실행될 수 있습니다.
// 이 코드는 현재 클러스터에서 진행 중인 모든 파티션 재할당 작업의 상태를 조회하여 출력합니다.
System.out.println("현재 진행 중인 재할당 작업: " +
        admin.listPartitionReassignments().reassignments().get());

// 5. 토픽의 상세 정보를 다시 조회하여 재할당이 어떻게 반영되고 있는지 확인합니다.
// 재할당이 진행 중일 때는 복제본 목록(replicas)에 이전 브로커와 새 브로커가 함께 표시될 수 있습니다.
// 재할당이 완료되면 여기에 새 브로커 목록만 표시됩니다.
DescribeTopicsResult demoTopic = admin.describeTopics(TOPIC_LIST);
TopicDescription topicDescription = demoTopic.values().get(TOPIC_NAME).get();
System.out.println("재할당 작업 후 토픽 상세 정보:" + topicDescription);
// 1. 파티션들을 새로운 브로커로 재할당(reassign)합니다.
//    이 작업은 클러스터의 브로커 간 데이터 균형을 맞추거나, 특정 브로커를 교체할 때 사용됩니다.

// 파티션 재할당 계획을 담을 Map을 생성합니다.
// Key는 재할당할 TopicPartition이고, Value는 새로운 복제본 목록 또는 취소 여부입니다.
Map<TopicPartition, Optional<NewPartitionReassignment>> reassignment = new HashMap<>();

// 'TOPIC_NAME' 토픽의 0번 파티션에 대한 새로운 복제본 위치를 지정합니다.
// 앞으로 이 파티션의 데이터는 브로커 0과 1에 저장되며, 0번 브로커가 선호 리더가 됩니다.
reassignment.put(new TopicPartition(TOPIC_NAME, 0),
        Optional.of(new NewPartitionReassignment(Arrays.asList(0, 1))));

// 'TOPIC_NAME' 토픽의 1번 파티션은 이제 브로커 0에만 저장됩니다.
reassignment.put(new TopicPartition(TOPIC_NAME, 1),
        Optional.of(new NewPartitionReassignment(Arrays.asList(0))));

// 'TOPIC_NAME' 토픽의 2번 파티션은 브로커 1과 0에 저장됩니다. (선호 리더: 1번 브로커)
reassignment.put(new TopicPartition(TOPIC_NAME, 2),
        Optional.of(new NewPartitionReassignment(Arrays.asList(1, 0))));

// 'TOPIC_NAME' 토픽의 3번 파티션에 대해서는 `Optional.empty()`를 지정합니다.
// 이는 현재 이 파티션에 대해 진행 중인 재할당 작업이 있다면 '취소'하라는 의미입니다.
reassignment.put(new TopicPartition(TOPIC_NAME, 3),
        Optional.empty());

// 2. 위에서 정의한 재할당 계획을 클러스터에 전달하여 실행을 요청합니다.
try {
    // alterPartitionReassignments 메서드를 호출하고, 요청이 수락될 때까지 기다립니다.
    // (참고) 이 호출은 데이터 복제가 완료될 때까지 기다리는 것이 아니라, 재할당 '작업'이 시작될 때까지만 기다립니다.
    admin.alterPartitionReassignments(reassignment).all().get();

} catch (ExecutionException e) {
    // 3. 재할당 요청 중 발생할 수 있는 예외를 처리합니다.
    
    // 만약 예외의 원인이 'NoReassignmentInProgressException'이라면, 이는 심각한 오류가 아닙니다.
    if (e.getCause() instanceof NoReassignmentInProgressException) {
        // 이 예외는 진행 중이지 않은 재할당 작업을 취소하려고 할 때 발생하므로, 무시하고 계속 진행하겠다는 메시지를 출력합니다.
        System.out.println("진행 중이 아닌 재할당 작업을 취소하려고 했으므로, 이 메시지는 무시합니다.");
    }
}

// 4. 현재 진행 중인 재할당 상태를 모니터링합니다.
// 재할당 작업은 백그라운드에서 오랜 시간 동안 실행될 수 있습니다.
// 이 코드는 현재 클러스터에서 진행 중인 모든 파티션 재할당 작업의 상태를 조회하여 출력합니다.
System.out.println("현재 진행 중인 재할당 작업: " +
        admin.listPartitionReassignments().reassignments().get());

// 5. 토픽의 상세 정보를 다시 조회하여 재할당이 어떻게 반영되고 있는지 확인합니다.
// 재할당이 진행 중일 때는 복제본 목록(replicas)에 이전 브로커와 새 브로커가 함께 표시될 수 있습니다.
// 재할당이 완료되면 여기에 새 브로커 목록만 표시됩니다.
DescribeTopicsResult demoTopic = admin.describeTopics(TOPIC_LIST);
TopicDescription topicDescription = demoTopic.values().get(TOPIC_NAME).get();
System.out.println("재할당 작업 후 토픽 상세 정보:" + topicDescription);
```
1. 파티션 0 에 새로운 레플리카를 추가하고, 새 레플리카를 ID 가 1인 새 브로커에 배치했다. 
	- 리더는 변경하지 않음
2. 파티션 1 에는 아무런 레플리카를 추가하지 않는다
	- 단지 이미 있던 레플리카를 새 브로커로 옮겼을 뿐임  
	- 레플리카가 하나뿐이므로 이것이 리더가 됨
3. 파티션 2 에 새로운 레플리카를 추가하고 이것을 선호 리더로 설정함
	- 다음 선호 리더 선출 시 새로운 브로커에 있는 새로운 레플리카로 리더가 변경
	- 이전 레플리카는 팔로워가 된다
4. 파티션 3 에 대해서는 진행중인 재할당 작업이 없음
	- 하지만 있다면 작업이 취소되고, 재할당 작업이 시작되기 전 상태로 원상복구될 것임
5. 현재 진행중인 재할당을 보여준다
6. 새로운 상태를 보여준다
	- 단, 일관적인 결과가 보일 때까지는 잠시 시간이 걸릴 수 있다.