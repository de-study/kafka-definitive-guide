<h2>Introduction</h2>
<ul>
<li>데이터에는 유의미한 정보 존재</li>
<li>데이터 생성 → (이동) → 데이터 분석 가능한 곳</li>
</ul>
<h1>1.1 Pub/Sub 메시지 전달</h1>
<p>&lt;aside&gt;</p>
<p>&lt;핵심컨셉&gt;</p>
<p>전송자(Producer)는 데이터를 직접 수신자(Consumer)에게 보내지 않는다. → 브로커(Broker) 필요
(Producer) → (Broker) → (Consumer)</p>
<p>&lt;/aside&gt;</p>
<h2>1.1.1 초기의 발행 구독 시스템</h2>
<p><img src="attachment:c4a98d5b-3a98-4707-a284-b5b13738bd23:image.png" alt="image.png"></p>
<p><img src="attachment:1c283cee-5438-4490-bac3-864b18ade1fe:image.png" alt="image.png"></p>
<ul>
<li>
<p>(기존) - 포인트투포인트 연결  →  (변경) - 중앙 집중방식(메시지큐 활용)</p>
</li>
<li>
<p><strong>캡슐화(Encapsulation)</strong></p>
<p>컨슈머는 &quot;데이터가 어디서 어떻게 생산(produce)되는지&quot; 알 필요가 없습니다.</p>
<p>→ Kafka(브로커, 토픽, 파티션)가 그 과정을 캡슐화해서 <strong>인터페이스만 제공</strong>합니다.</p>
<p>→ 컨슈머는 &quot;이 토픽에서 메시지를 읽는다&quot;라는 행위만 하면 됩니다.</p>
</li>
<li>
<p><strong>느슨한 결합(Loose Coupling)</strong></p>
<p>프로듀서와 컨슈머는 서로 직접 연결되지 않습니다.</p>
<p>→ Kafka라는 메시지 큐(중개자)를 통해서만 데이터가 흐름.</p>
<p>→ 덕분에 프로듀서/컨슈머가 독립적으로 개발·운영·확장 가능.</p>
</li>
<li>
<p><strong>다형성(Polymorphism)</strong></p>
<p>같은 토픽이라도, 컨슈머 그룹에 따라 다른 방식으로 데이터를 소비할 수 있습니다.</p>
<p>→ 예:</p>
<ul>
<li>하나의 컨슈머 그룹은 &quot;실시간 알림&quot;에 사용</li>
<li>다른 컨슈머 그룹은 &quot;배치 데이터 적재&quot;에 사용</li>
</ul>
</li>
<li>
<p><strong>추상화(Abstraction)</strong></p>
<p>컨슈머 입장에서는 &quot;토픽에서 메시지를 가져온다&quot;는 것만 중요하지,</p>
<p>그 메시지가 IoT 센서에서 왔는지, 웹 로그에서 왔는지, DB CDC에서 왔는지는 몰라도 됩니다.</p>
</li>
</ul>
<h3>1.1.2 개별 메시지 큐 시스템</h3>
<ul>
<li><strong>포인트 투 포인트 문제</strong>
<ol>
<li>연결 복잡성이 폭증함
<ol>
<li>서비스가 3개일 때 3개, 5개 일 때 10개, 10개일 때 45개 연결이 필요하다.</li>
</ol>
</li>
<li>출처 관리의 어려움
<ol>
<li>어떤 서비스가 어떤 데이터를 보내는지 추적이 어렵다.</li>
<li>장애 발생 시 원인 파악이 어렵다</li>
<li>데이터 흐름 파악이 복잡해진다</li>
</ol>
</li>
</ol>
</li>
<li><strong>발행/구독의 해결책</strong>
<ol>
<li>중간자를 통한 복잡도 개선</li>
<li>느슨한 결합
<ol>
<li>발행자는 구독자가 누구인지 몰라도 된다.</li>
<li>구독자도 발행자가 누구인지 몰라도 된다.</li>
<li>서비스 추가/제거가 다른 서비스에 영향을 미치지 않는다.</li>
<li>Why?
<ol>
<li>결합도를 낮추는 행위
<ol>
<li>A 프로듀서가 죽으면, A 컨슈머도 죽어야 하는가? No → 일부 시스템 장애가 전체에 영향을 미치지 않음.</li>
</ol>
</li>
<li>클러스터
<ol>
<li>분산처리 시스템.</li>
<li>여러개의 노드가 하나의 집단을 이뤄서 병렬로 처리 가능.</li>
</ol>
</li>
</ol>
</li>
</ol>
</li>
<li>확장성
<ol>
<li>새로운 서비스가 추가되더라도 기존 연결에 변경은 불필요하다.</li>
<li>수평적 확장에 용이하다</li>
</ol>
</li>
<li>내결함성
<ol>
<li>한 서비스가 다운되더라도 다른 서비들은 정상 동작한다.</li>
<li>메시지가 브로커에 저장되어 일시적 장애 상황에서도 데이터 손실 방지가 쉽다.</li>
<li>큐 → 데이터 영속성 여부는 메모리 vs 디스크임.
<ol>
<li>레빗엠큐 ( (디폴트)메모리 큐 , 디스크 기반으로도 가능) → 비교 대상으로서 조사 필요.</li>
<li>레디스 ( 메모리 큐 )</li>
<li>카프카 ( 디스크 큐 )</li>
</ol>
</li>
</ol>
</li>
<li>비동기 처리
<ol>
<li>발행자가 구독자의 응답을 기다릴 필요가 없다.</li>
<li>처리 속도가 다른 서비스들 간의 동기화 문제를 해결해준다.</li>
</ol>
</li>
</ol>
</li>
</ul>
<h1>1.2 카프카 입문</h1>
<h2>1.2.1 메시지와 배치</h2>
<ul>
<li>메시지
<ul>
<li>카프카에서 데이터의 기본 단위는 메시지</li>
<li>카프카 입장에서 단순한 바이트 배열 → 특정한 형식이나 의미가 없다.</li>
<li>메시지는 키라고 불리는 메타 데이터를 포함</li>
<li>동일한 키를 가진 메시지에 한해서 동일한 파티션에 저장</li>
</ul>
</li>
</ul>
<h2>1.2.2 스키마</h2>
<ul>
<li>
<p>정의</p>
<ul>
<li>메시지의 내용을 이해하기 쉽도록 일정한 구조를 부여하는 것</li>
</ul>
</li>
<li>
<p>어떻게 구현?</p>
<ul>
<li>파일형식 - JSON, XML → 가장 간단
<ul>
<li>타입 처리, 스키마 버전 간의 호환성 유지 어려움</li>
</ul>
</li>
</ul>
</li>
<li>
<p>가장 적합한 파일 형식  - Avro</p>
<ul>
<li>
<p>일반적으로 카프카 개발자들은 아파치 Avro 선호</p>
</li>
<li>
<p>바이너리 형태로 16진수의 나열 형태 → 사람이 읽을 수 없음, 용량이 작음</p>
</li>
<li>
<p>Avro 스키마 정의 (Schema Definition)</p>
<ul>
<li>
<p>예시</p>
<pre><code class="language-json">{
  &quot;type&quot;: &quot;record&quot;,
  &quot;name&quot;: &quot;ViewingEvent&quot;,
  &quot;namespace&quot;: &quot;com.netflix.events&quot;,
  &quot;fields&quot;: [
    {
      &quot;name&quot;: &quot;user_id&quot;,
      &quot;type&quot;: &quot;string&quot;,
      &quot;doc&quot;: &quot;사용자 고유 식별자&quot;
    },
    {
      &quot;name&quot;: &quot;content_id&quot;, 
      &quot;type&quot;: &quot;string&quot;,
      &quot;doc&quot;: &quot;시청한 컨텐츠 ID&quot;
    },
    {
      &quot;name&quot;: &quot;event_type&quot;,
      &quot;type&quot;: {
        &quot;type&quot;: &quot;enum&quot;,
        &quot;name&quot;: &quot;EventType&quot;,
        &quot;symbols&quot;: [&quot;play&quot;, &quot;pause&quot;, &quot;stop&quot;, &quot;complete&quot;]
      },
      &quot;doc&quot;: &quot;시청 이벤트 타입&quot;
    },
    {
      &quot;name&quot;: &quot;timestamp&quot;,
      &quot;type&quot;: &quot;long&quot;,
      &quot;logicalType&quot;: &quot;timestamp-millis&quot;,
      &quot;doc&quot;: &quot;이벤트 발생 시간&quot;
    },
    {
      &quot;name&quot;: &quot;duration_seconds&quot;,
      &quot;type&quot;: [&quot;null&quot;, &quot;int&quot;],
      &quot;default&quot;: null,
      &quot;doc&quot;: &quot;시청 시간 (초 단위, 선택적)&quot;
    },
    {
      &quot;name&quot;: &quot;device_info&quot;,
      &quot;type&quot;: {
        &quot;type&quot;: &quot;record&quot;,
        &quot;name&quot;: &quot;DeviceInfo&quot;,
        &quot;fields&quot;: [
          {&quot;name&quot;: &quot;device_type&quot;, &quot;type&quot;: &quot;string&quot;},
          {&quot;name&quot;: &quot;os_version&quot;, &quot;type&quot;: &quot;string&quot;},
          {&quot;name&quot;: &quot;app_version&quot;, &quot;type&quot;: &quot;string&quot;}
        ]
      },
      &quot;doc&quot;: &quot;디바이스 정보&quot;
    }
  ]
}
</code></pre>
</li>
</ul>
</li>
<li>
<p>실제 avro 어떻게 생겼는지</p>
<ul>
<li>
<p>예시</p>
<p><strong>JSON 형태로 표현하면 (이해를 위해):</strong></p>
<pre><code class="language-json">{
  &quot;user_id&quot;: &quot;user_12345&quot;,
  &quot;content_id&quot;: &quot;movie_parasite_2019&quot;,
  &quot;event_type&quot;: &quot;complete&quot;,
  &quot;timestamp&quot;: 1692284400000,
  &quot;duration_seconds&quot;: 7920,
  &quot;device_info&quot;: {
    &quot;device_type&quot;: &quot;smart_tv&quot;,
    &quot;os_version&quot;: &quot;tizen_6.0&quot;,
    &quot;app_version&quot;: &quot;netflix_8.2.1&quot;
  }
}
</code></pre>
<p><strong>실제 Avro 바이너리 (16진수):</strong></p>
<pre><code class="language-json">14 75 73 65 72 5f 31 32 33 34 35 2e 6d 6f 76 69 65 5f 70 61 
72 61 73 69 74 65 5f 32 30 31 39 06 80 b4 f4 c9 ed 63 02 f0 
7b 12 73 6d 61 72 74 5f 74 76 14 74 69 7a 65 6e 5f 36 2e 30 
1a 6e 65 74 66 6c 69 78 5f 38 2e 32 2e 31
</code></pre>
</li>
</ul>
</li>
</ul>
</li>
<li>
<p>스키마 레지스트리</p>
<ul>
<li>선택적인 옵션임</li>
<li>avro를 사용하기위해 반드시 스키마 레지스트리 필요 x</li>
</ul>
</li>
</ul>
<h2>1.2.3 토픽과 파티션</h2>
<h3>(1) 토픽(Topic)</h3>
<ul>
<li>
<p><strong>정의</strong>: 데이터가 카테고리별로 나뉘어 저장되는 이름표(논리적 단위)</p>
</li>
<li>
<p><strong>비유</strong>: 뉴스 웹사이트의 <strong>카테고리</strong>처럼 생각하면 됨. (예: <code>sports</code>, <code>finance</code>, <code>weather</code>)</p>
</li>
<li>
<p><strong>특징</strong>: 저장관점. 읽기/쓰기 구분 없이 모든 메시지는 해당 토픽에 기록됨</p>
<pre><code>토픽(Topic)
│
├─ 파티션(Partition)
│   ├─ 메시지1
│   ├─ 메시지2
│   └─ 메시지3
│
├─ 파티션(Partition)
│   ├─ 메시지4
│   └─ 메시지5
│
└─ 파티션(Partition)
    ├─ 메시지6
    └─ 메시지7

스트림(Stream) = 토픽 전체 또는 파티션별로 소비되는 메시지 흐름
</code></pre>
</li>
</ul>
<h3>2. 메시지(Message)</h3>
<ul>
<li>
<p><strong>정의</strong>: 토픽에 기록되는 데이터 단위</p>
</li>
<li>
<p><strong>내용</strong>: key, value, timestamp 등 포함 가능</p>
</li>
<li>
<p><strong>예시</strong>:</p>
<pre><code class="language-json"># 실제 메시지 예시
{
    &quot;key&quot;: &quot;user_123&quot;,           # 파티션 결정에 사용
    &quot;value&quot;: {                   # 실제 데이터
        &quot;user_id&quot;: &quot;123&quot;,
        &quot;action&quot;: &quot;add_to_cart&quot;,
        &quot;product_id&quot;: &quot;prod_456&quot;,
        &quot;timestamp&quot;: &quot;2024-08-17T10:30:00Z&quot;
    },
    &quot;headers&quot;: {                 # 메타데이터
        &quot;source&quot;: &quot;web-app&quot;,
        &quot;version&quot;: &quot;v1.0&quot;
    },
    &quot;offset&quot;: 12345,            # 파티션 내 순서
    &quot;partition&quot;: 1              # 어느 파티션에 저장됐는지
}
</code></pre>
</li>
</ul>
<h3>3. 파티션(Partition)</h3>
<ul>
<li><strong>정의</strong>: 토픽을 나눈 <strong>물리적 저장 단위</strong></li>
<li><strong>목적</strong>:
<ul>
<li>데이터를 여러 서버에 분산 저장 → 확장성 확보, 클러스터, 분산처리</li>
<li>순서 보장: 같은 파티션 내 메시지는 순서 보장</li>
<li>파티션간의 전체 메시지의 순서를 보장하지는 않는다.</li>
</ul>
</li>
<li><strong>예시</strong>: 토픽 <code>purchases</code>를 3개의 파티션으로 나눔
<ul>
<li>파티션 0: user1, user4, user7</li>
<li>파티션 1: user2, user5, user8</li>
<li>파티션 2: user3, user6, user9</li>
</ul>
</li>
</ul>
<h3>4. 스트림(Stream)</h3>
<ul>
<li>
<p><strong>정의</strong>: 메시지 처리관점. 토픽에서 발생하는 데이터의 <strong>연속적인 흐름(논리적개념)</strong></p>
</li>
<li>
<p>메시지를 처리해서 어디에 사용하나? ex) 사용차추천 스트림, 매출집계 스트림, 트래픽 집계스트림</p>
</li>
<li>
<p>여러 토픽의 메시지를 받아들여 하나의 스트림을 구성할 수 있음.</p>
<pre><code class="language-json">🌊 개인화 추천 스트림 (김민수 담당)
   ├── viewing-events 토픽에서 시청 기록 읽기
   ├── user-profiles 토픽에서 사용자 정보 읽기
   ├── 실시간으로 개인 맞춤 추천 생성
   └── recommendation-results 토픽으로 결과 전송

🌊 실시간 트렌드 스트림 (박영희 담당)  
   ├── viewing-events 토픽에서 조회수 집계
   ├── 1분마다 인기 급상승 컨텐츠 탐지
   └── trending-topics 토픽으로 결과 전송
</code></pre>
</li>
<li>
<p><strong>예시</strong>: <code>purchases</code> 토픽에서 들어오는 모든 메시지를 분석해서 <strong>실시간 매출 집계</strong></p>
</li>
</ul>
<h2>1.2.4 프로듀서와 컨슈머</h2>
<h3>(1)프로듀서(Producer)</h3>
<ul>
<li>메시지 생성자</li>
<li>발행자 또는 작성자</li>
<li>메시지 생성시 골고루 파티션에 분배</li>
<li>파티셔너
<ul>
<li>키-키값의 해시 → 파티션으로 대응시켜줌</li>
<li>동일한 키 - 동일 파티션</li>
<li>커스텀하게 설정 가능</li>
</ul>
</li>
</ul>
<h3>(2)컨슈머(Consumer)</h3>
<ul>
<li>메시지 읽어가는 대상</li>
<li>구독자 또는 독자</li>
<li>1개 이상의 토픽 구독 → 스트림에 따라 여러 토픽 구독 가능</li>
<li>메시지 오프셋 기록 → 어디까지 읽어갔는지 구분</li>
<li>오프셋
<ul>
<li>지속적으로 증가하는 정수값</li>
<li>카프카가 메시지 저장시 부여</li>
</ul>
</li>
<li>컨슈머그룹
<ul>
<li>클러스터 형태</li>
<li>하나의 토픽을 읽어오는 여러개의 컨슈머 존재 → 각 컨슈머별 파티션 정해져있음</li>
</ul>
</li>
</ul>
<h2>1.2.5 브로커와 클러스터</h2>
<h3>(1)브로커</h3>
<ul>
<li>카프카 서버 1개 (물리적)</li>
<li>역할
<ul>
<li>메시지에 오프셋 할당 - 디스크에 저장</li>
<li>컨슈머의 파티션 읽기 요청(fetch) 처리</li>
</ul>
</li>
<li>클러스터
<ul>
<li>leader - follower 노드</li>
<li>파티션리더(leader)  - 컨트롤러 역할
<ul>
<li>파티션을 브로커에게 할당</li>
<li>장애 브로커 모니터링</li>
</ul>
</li>
<li>Follower
<ul>
<li>복제기능</li>
<li>파티션의 메시지를 중복 저장 → 장애 대처</li>
<li>리더 브로커 장애 발생시 - 팔로워중 1개가 리더 역할 수행</li>
</ul>
</li>
</ul>
</li>
<li>메시지 지속성(durability)
<ul>
<li>보존기능(retention)
<ul>
<li>특정기간 동안 메시지 보관 - 이후 삭제</li>
<li>파티션 크기가 일정 (~1GB) 도달시까지 메시지 보관 - 이후 삭제</li>
</ul>
</li>
</ul>
</li>
<li>로그압착(Log compaction)
<ul>
<li>키값 기준 → 가장 최신 메시지만 보존</li>
</ul>
</li>
</ul>
<h3>(2)동작방식</h3>
<p>&lt;aside&gt;</p>
<p>프로듀서 → 리더 브로커 (쓰기 요청)
리더 브로커 → 팔로워 브로커들 (복제 전달, pull 방식)</p>
<p>&lt;/aside&gt;</p>
<ul>
<li>
<p>클러스터에서 컨슈머가 메시지 가져가는 방식</p>
<ul>
<li>
<p>예를 들어 <code>topicA</code>의 <code>partition0</code>이 있고, 복제 계수(RF)=3이라면:</p>
<ul>
<li>브로커1 → <strong>리더</strong> (partition0 리더)</li>
<li>브로커2 → <strong>팔로워</strong> (partition0 복제본)</li>
<li>브로커3 → <strong>팔로워</strong> (partition0 복제본)</li>
</ul>
<pre><code class="language-json">1. 프로듀서가 메시지를 브로커1(리더)에 전송
2. 브로커2, 브로커3의 팔로워들이 브로커1로부터 pull해서 복제
    - 즉, 리더는 데이터를 &quot;푸시&quot;하지 않고, 팔로워들이 &quot;나도 가져갈래&quot; 하면서 따라잡음
3. 팔로워들이 리더와 동일한 오프셋까지 데이터를 받아야 ISR(In-Sync Replica)로 간주됨

프로듀서 → 리더 브로커 (쓰기 요청)
리더 브로커 → 팔로워 브로커들 (복제 전달, pull 방식)
</code></pre>
</li>
</ul>
</li>
<li>
<p>파티션 단위로 리더가 정해짐</p>
<ul>
<li>
<p>(주의) 브로커는 특정 파티션의 리더일 수도 있고, 다른 파티션의 팔로워일 수도 있다.</p>

파티션 | 리더 | 팔로워1 | 팔로워2
-- | -- | -- | --
p0 | B1 | B2 | B3
p1 | B2 | B1 | B3
p2 | B3 | B1 | B2
p3 | B1 | B2 | B3


</li>
</ul>
</li>
</ul>
<h2>1.2.6 다중 클러스터</h2>
<h3>미러메이커</h3>
<ul>
<li>카프카 클러스터간의 복제</li>
</ul>
<h2>1.3 왜 카프카인가?</h2>
<p>발행/구독 메시지 전달 시스템에는 여러 가지가 존재한다. 그렇다면 카프카가 좋은 이유에는 무엇이 있을까?</p>
<ul>
<li>MSA 기반 서비스와 연결성이 좋음</li>
</ul>
<h3>1.3.1 다중프로듀서</h3>
<h3>1.3.2 다중 컨슈머</h3>
<h3>1.3.3 디스크 기반 보존</h3>
<ul>
<li>디스크에 저장해서 유실걱정없음</li>
<li>유지보수해도 유실될 걱정 안해도됨</li>
</ul>
<h3>1.3.4 확장성</h3>
<ul>
<li>분산처리, 스케일링 가능함 -브로커,컨슈머</li>
</ul>
<h3>1.3.5 고성능</h3>
<p>아파치 카프카가 고부하 아래에서도 높은 성능을 보여주는 발행/구독 메시지 전달 시스템이 될 수 있었던 것은 지금까지 설명한 모든 특징들 덕분이다. 발행된 메시지가 컨슈머에게 전달되는 시간이 1초도 안걸리면서 프로듀서, 컨슈머, 브로커 모두가 매우 큰 메시지 스트림을 쉽게 다룰 수 있도록 수평적으로 확장될 수 있는 것이다.</p>
<h3>1.3.6 플랫폼 기능</h3>
<p>아파치 카프카의 코어 프로젝트에는 개발자들이 자주 하는 작업을 훨씬 쉽게 수행할 수 있도록 해주는 플랫폼 기능이 추가되어 있다. YARN 처럼 구조화된 런타임 환경을 포함하는 완전한 플랫폼은 아니지만, 이 기능들을 탄탄한 기반과 자유로운 형태로 실행할 수 있는 유연성을 갖춘 API의 라이브러리의 형태로 사용이 가능하다.</p>
<h2>1.4 데이터 생태계</h2>
<h3>1.4.1 이용 사례</h3>
<p>활동 추적</p>
<p>카프카의 원래 용도는 사용자 활동 추적이었으며, 이는 웹 사이트 사용자가 행동한 것에 대한 메시지를 생성하기 위한 프론트엔드 애플리케이션이 동작한다. 이를 통해서 백엔드에서 처리하여 보고서나 머신 러닝을 위한 데이터로서 활용되도록 수행할 수 있다.</p>
<p>메시지 교환</p>
<p>카프카는 메시지 교환 시에도 사용되므로, 알림을 보내야 하는 애플리케이션 등에도 활용할 수 있다.</p>
<p>지표 및 로그 수집</p>
<p>카프카는 애플리케이션과 시스템의 지푯값과 로그를 수집할 때에도 이상적이다. 여러 애플리케이션에서 생성된 동일한 유형의 메시지를 활용하여 로그 메시지와 같은 방식으로 발행될 수 있으며, 또 다른 장점으로 목적 시스템을 변경해야 할 때, 프론트엔드 애플리케이션이나 메시지 수집 방법을 변경할 필요가 없다는 것이다.</p>
<p>커밋 로그</p>
<p>스트림 처리</p>
<p>하둡, 맵리듀스 등과 같은 애플리케이션을 의미한다. 하둡은 오랜 시간에 걸쳐 누적된 데이터를 처리하는 반면에, 스트림 처리는 메시지가 생성되자마자 실시간으로 데이터를 처리한다는 차이가 있다. 스트림 처리 프레임워크를 사용하면 카프카 메시지를 처리하는 작은 애플리케이션을 작성하거나, 각 종 지푯값을 계산하는 것과 같은 작업을 수행하거나, 다른 애플리케이션이 효율적으로 처리할 수 있게 메시지를 파티셔닝하거나, 다수의 원본으로부터 들어온 데이터를 사용하여 메시지를 변환한다거나 할 수 있다.</p>
<h2>1.5 카프카의 기원</h2>
<ul>
<li>기존방식
<ul>
<li>서버에 데이터가 저장되면 → 1시간 간격으로 폴링(polling) 읽어와서 연산후 모니터링 대시보드 뿌리기</li>
</ul>
</li>
<li>문제점
<ul>
<li>Polling 기반 → 실시간성 부족 (5분~1시간 지연)</li>
<li>High-touch 운영 → 사람이 수동으로 처리해야 할 일 많음</li>
<li>Inconsistent naming → 같은 지표를 시스템마다 다른 이름으로 사용</li>
<li>개발자 권한 없음 → 애플리케이션 개발자가 메트릭 관리 불가</li>
<li>Large intervals → 메트릭 수집 간격이 커서 문제 발견 늦음</li>
</ul>
</li>
</ul>
<p>&lt;aside&gt;</p>
<h3>Polling (폴링)</h3>
<p><strong>정의</strong>: 주기적으로 상태를 확인하는 방식</p>
<p><strong>전형적인 Polling 상황:</strong></p>
<ul>
<li>서버 관리자가 5분마다 CPU 사용률 체크</li>
<li>배치 작업이 1시간마다 새로운 파일 확인</li>
<li>모니터링 시스템이 정해진 시간마다 모든 서버 상태 점검</li>
</ul>
<p><strong>특징</strong>: 정해진 간격으로 반복 확인</p>
<h3>Pull (풀)</h3>
<p><strong>정의</strong>: 필요할 때 데이터를 가져오는 방식</p>
<p><strong>전형적인 Pull 상황:</strong></p>
<ul>
<li>이메일 앱에서 &quot;새로고침&quot; 버튼 눌렀을 때 메일 확인</li>
<li>Kafka Consumer가 메시지 있는지 확인하고 즉시 가져오기</li>
<li>웹브라우저가 서버에서 데이터 요청해서 받아오기</li>
</ul>
<h2>Kafka 메시지 흐름 순서</h2>
<p><strong>Producer → Broker → Consumer 순서:</strong></p>
<ol>
<li><strong>Producer가 메시지 생성</strong> → Kafka Broker에 Push</li>
<li><strong>Broker가 메시지 저장</strong> → 토픽의 파티션에 보관</li>
<li><strong>Consumer가 요청</strong> → &quot;새 메시지 있나요?&quot; (Pull) - 밀리seconds단위로 확인</li>
<li><strong>Broker가 응답</strong> → &quot;네, 여기 있어요&quot; 또는 &quot;없어요&quot;</li>
<li><strong>Consumer가 메시지 처리</strong> → 비즈니스 로직 실행</li>
</ol>
<p>결국 <strong>polling, pull 동일한 방식</strong> → 간격의 차이가 있을뿐
pull - 브로커, 컨슈머와 지속적인 연결유지, 실시간성 있음</p>
<p>&lt;/aside&gt;</p>
<!-- notionvc: d8b3bd1c-c387-46f8-b216-9892aa9fc26b -->
