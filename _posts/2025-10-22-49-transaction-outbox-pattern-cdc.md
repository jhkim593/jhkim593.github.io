---
layout: post
title: "Transactional Outbox Pattern - Transaction Log Tailing 구현"
author: "jhkim593"
tags: Architecture
---

이전 게시글에서는 Transaction Outbox Pattern과 폴링 방식을 통한 구현을 다뤘었습니다. 폴링 방식은 구현이 간단하고 안정적이지만, 
주기적인 DB 조회로 인한 오버헤드와 폴링 주기에 따른 지연이 발생할 수 있다는 단점이 있었습니다. 이번 게시글에서는 이러한 문제를 보완할 수 있는 **Transaction Log Tailing** 방식에 대해 살펴보겠습니다.

<br>

## Transaction Log Tailing

<img src="/assets/images/49/1.png"  width="650" height="440"/>

**Transaction Log Tailing**은 데이터베이스의 트랜잭션 로그를 직접 읽어 변경 사항(INSERT, UPDATE, DELETE 등)을 감지하는 방식입니다.

동작 흐름은 다음과 같습니다
- Order Service가 주문 데이터를 저장하면서 Outbox 테이블에도 이벤트를 INSERT합니다.
- INSERT는 데이터베이스 트랜잭션 로그에 기록됩니다.
- Transaction log miner가 그 로그를 읽어 실시간으로 읽어 변경 사항을 감지합니다.
- 변경사항을 메세지 브로커에 발행합니다.

폴링 방식과 다르게 주기적으로 DB에 쿼리를 전송할 필요가 없고 실시간에 가깝게 변경 사항을 감지할 수 있다는 장점이 있습니다.
이번 게시글에서는 **Debezium**을 사용한 Transaction Log Tailing을 다뤄보겠습니다.


<br>

## Debezium

Transaction Log Tailing 방식을 사용하는 대표적인 Change Data Capture(CDC) 도구입니다. DB의 변경 사항 실시간으로 감지하여 이벤트 형태로 다른 시스템에 전송하는 역할을 합니다.

Debezium을 사용하는 방식은 여러가지가 있지만 대표적으로 Kafka Connect를 사용한 방식이 사용됩니다.

<img src="/assets/images/49/2.png"  width="900" height="220"/>

Debezium 커넥터는 소스 커넥터로 데이터베이스에 연결돼 변경 사항을 감지하고 Kafka 토픽에 기록합니다.  
PostgreSQL, MySQL 외에도 MongoDB, SQL Server, Oracle등 다양한 데이터베이스를 지원합니다.

또한 **at-least once** 전달을 보장합니다. 커넥터가 다운되고 시작될 때 동일한 이벤트를 여러 번 게시할 때가 있습니다. 따라서 수신측은 중복 이벤트가 다시 처리되지 않도록 멱등하게 설게해야합니다.


<br>

## 구현

Transaction Log Tailing 방식의 Outbox Pattern을 구현하기 위해 다음과 같은 인프라 구성이 필요합니다.

- **Kafka**
- **PostgreSQL**
- **Debezium**

로컬 개발 환경에서의 빠른 구성과 테스트를 위해 **docker-compose**를 사용했습니다.

<br>

### Kafka 설정

메시지 브로커인 **Kafka**와 모니터링을 위한 **Kafka UI**를 구성했습니다.
```yaml
  kafka:
    image: confluentinc/cp-kafka:latest
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: 'CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT,INTERNAL:PLAINTEXT'
      KAFKA_ADVERTISED_LISTENERS: 'EXTERNAL://localhost:9092,INTERNAL://kafka:9093'
      KAFKA_LISTENERS: 'EXTERNAL://0.0.0.0:9092,INTERNAL://0.0.0.0:9093,CONTROLLER://0.0.0.0:29093'
      KAFKA_PROCESS_ROLES: 'broker,controller'
      KAFKA_CONTROLLER_QUORUM_VOTERS: '1@kafka:29093'
      KAFKA_INTER_BROKER_LISTENER_NAME: 'INTERNAL'
      KAFKA_CONTROLLER_LISTENER_NAMES: 'CONTROLLER'
      KAFKA_LOG_DIRS: '/tmp/kraft-combined-logs'
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_NUM_PARTITIONS: 3
      CLUSTER_ID: 'MkU3OEVBNTcwNTJENDM2Qk'

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    depends_on:
      - kafka
    ports:
      - "8989:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9093
```

- `KAFKA_ADVERTISED_LISTENERS`: 클라이언트에게 알리는 접속 주소입니다. 클라이언트는 이 주소를 사용하여 Kafka에 연결합니다.
- `KAFKA_LISTENERS`: Kafka 브로커가 실제로 바인딩할 네트워크 인터페이스와 포트를 지정합니다.
- `KAFKA_INTER_BROKER_LISTENER_NAME`: 브로커 간 내부 통신에 사용할 리스너 이름입니다. `INTERNAL` 리스너를 사용하여 같은 Docker 네트워크 내에서 통신합니다.
- `KAFKA_CONTROLLER_LISTENER_NAMES` : 컨트롤러 역할을 수행할 때 사용할 리스너 이름입니다. `CONTROLLER` 리스너를 통해 메타데이터 복제 및 리더 선출을 수행합니다.
- `KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR` : 컨슈머 오프셋 토픽의 복제 개수입니다. 개발 환경이므로 1로 설정했습니다.
- `KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR`: 트랜잭션 상태 로그 토픽의 복제 개수입니다. 개발 환경이므로 1로 설정했습니다.
- `KAFKA_TRANSACTION_STATE_LOG_MIN_ISR` : 트랜잭션 상태 로그의 최소 동기화 복제본(In-Sync Replica) 수입니다. 개발 환경이므로 1로 설정했습니다.


<br>

### Postgresql 설정

```yaml
  article-db:
    image: postgres:13.15-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: article
      POSTGRES_USER: root
      POSTGRES_PASSWORD: "8834"
      TZ: Asia/Seoul
    command: [
      'postgres','-c','wal_level=logical'
    ]
    volumes:
      - article-db-data:/var/lib/postgresql/data

```
CDC를 사용하기 위한 핵심 설정은 다음과 같습니다.
- `command: ['postgres','-c','wal_level=logical']`: WAL레벨을 `logical`로 설정합니다. 이는 CDC가 트랜잭션 로그를 읽을 수 있도록 하는 필수 설정입니다.

<br>


### Debezium 설정
```yaml
  article-kafka-connect:
    image: debezium/connect:2.4
    depends_on:
      - kafka
      - article-db
    environment:
      BOOTSTRAP_SERVERS: kafka:9093
      GROUP_ID: 1
      CONFIG_STORAGE_TOPIC: connect_configs
      OFFSET_STORAGE_TOPIC: connect_offsets
      STATUS_STORAGE_TOPIC: connect_statuses
      CONFIG_STORAGE_REPLICATION_FACTOR: 1
      OFFSET_STORAGE_REPLICATION_FACTOR: 1
      STATUS_STORAGE_REPLICATION_FACTOR: 1
    ports:
      - "8083:8083"

  article-debezium-init:
    image: curlimages/curl:latest
    container_name: debezium-init
    depends_on:
      - article-kafka-connect
    volumes:
      - ./common/outbox-cdc/connector:/connector
    command: >
      sh -c "
        echo 'Waiting for Kafka Connect to be ready...'
        while ! curl -f http://article-kafka-connect:8083/connectors; do
          echo 'Kafka Connect not ready yet...'
          sleep 5
        done
        echo 'Kafka Connect is ready! Creating connector...'
        curl -X POST http://article-kafka-connect:8083/connectors -H 'Content-Type: application/json' -d @/connector/postgresql-connector.json
        echo 'Connector created successfully!'
      "
```

**article-kafka-connect**
- `BOOTSTRAP_SERVERS`: Kafka 브로커 주소입니다. 
- `CONFIG_STORAGE_TOPIC`: 커넥터 설정을 저장하는 Kafka 토픽입니다.
- `OFFSET_STORAGE_TOPIC`: 커넥터의 오프셋 정보를 저장하는 Kafka 토픽입니다. 장애 복구 시 마지막 처리 위치를 파악하는 데 사용됩니다.
- `STATUS_STORAGE_TOPIC`: 커넥터와 태스크의 상태 정보를 저장하는 Kafka 토픽입니다.
- 개발 환경이기 때문에 각 스토리지 토픽의 복제 개수를 1로 설정했습니다.

**article-debezium-init**
- Debezium 커넥터를 자동으로 등록하는 초기화 컨테이너입니다.
- Kafka Connect가 준비될 때까지 대기한 후 `postgresql-connector.json` 파일을 사용하여 커넥터를 생성합니다.


<br>
커넥터 설정 파일은 다음과 같습니다.

**postgresql-connector.json**
```json
{
  "name": "postgres-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "article-db",
    "database.port": "5432",
    "database.user": "root",
    "database.password": "8834",
    "database.dbname": "article",
    "plugin.name": "pgoutput",
    "publication.name": "outbox_publication",
    "slot.name": "debezium_slot",
    "table.include.list": "public.outbox_event",
    "topic.prefix": "article",
    "schema.history.internal.kafka.bootstrap.servers": "kafka:9093",
    "schema.history.internal.kafka.topic": "schema-changes.postgres",
    "key.converter": "org.apache.kafka.connect.json.StringConverter",
    "value.converter": "org.apache.kafka.connect.json.StringConverter",
    "key.converter.schemas.enable": false,
    "value.converter.schemas.enable": false,
    "transforms": "unwrap,route",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.route.type": "org.apache.kafka.connect.transforms.RegexRouter",
    "transforms.route.regex": "article.public.outbox_event",
    "transforms.route.replacement": "article"
  }
}
```

**데이터베이스 연결 설정**
- `connector.class`: PostgreSQL용 Debezium 커넥터 클래스를 지정합니다.
- `database.hostname`, `database.port`, `database.user`, `database.password`, `database.dbname`: PostgreSQL 데이터베이스 연결 정보입니다.

**CDC 플러그인 설정**
- `plugin.name`: PostgreSQL의 논리적 디코딩 플러그인입니다. `pgoutput`은 PostgreSQL 10 이상에서 기본 제공되는 플러그인입니다.
- `publication.name`: PostgreSQL에서 생성할 Publication 이름입니다. Publication은 복제할 테이블을 정의합니다.
- `slot.name`: Replication Slot 이름입니다. Slot은 CDC가 WAL을 읽는 위치를 추적합니다.

**테이블 및 토픽 설정**
- `table.include.list`: 모니터링할 테이블 목록입니다. `public.outbox_event` 테이블만 CDC 대상으로 지정했습니다.
- `topic.prefix`: 생성될 Kafka 토픽의 접두사입니다. 기본적으로 `{prefix}.{schema}.{table}` 형식으로 토픽이 생성됩니다.
- `schema.history.internal.kafka.bootstrap.servers`: 스키마 변경 이력을 저장할 Kafka 브로커 주소입니다.
- `schema.history.internal.kafka.topic`: 스키마 변경 이력을 저장할 Kafka 토픽입니다.

**데이터 변환 설정**
- `key.converter`, `value.converter`: 데이터를 String 형식으로 변환합니다.
- `key.converter.schemas.enable`, `value.converter.schemas.enable`: 카프카에 전송할 때 스키마 정보를 포함하지 않도록 설정합니다. 

**Transforms 설정**
- `transforms`: 적용할 변환 목록입니다. `unwrap`과 `route` 두 가지 변환을 순차적으로 적용합니다.
- `transforms.unwrap.type`: Debezium의 변경 이벤트 구조를 단순화합니다. 기본적으로 Debezium은 `before`, `after` 같은 메타데이터를 포함해 카프카에 전송하지만, `ExtractNewRecordState`를 사용하면 실제 데이터만 사용합니다
- `transforms.route.type`: 정규식을 사용하여 토픽 이름을 변경합니다.
- `transforms.route.regex`: 원본 토픽 이름 패턴입니다. 
- `transforms.route.replacement`: 변경할 토픽 이름입니다. 이를 통해 `article.public.outbox_event` 토픽이 `article` 토픽으로 라우팅됩니다.

<br>


### 어플리케이션 코드

애플리케이션 코드는 폴링 방식과 동일합니다. 자세한 코드 설명은 이전 게시글을 참고해주세요.

<br>
**OutboxEvent.java**
```java
public class OutboxEvent {
    @Id
    private Long id;
    @Enumerated(EnumType.STRING)
    private EventType type;
    private Long aggregateId;
    @Column(columnDefinition = "TEXT")
    private String payload;
    private LocalDateTime createdAt;
}
```
<br>

Spring Event를 사용해 내부 이벤트를 발행하고 발행된 이벤트를 처리할 Listener를 생성했습니다.
```java
@Override
@Transactional
public Article register(ArticleRegisterDto request) {
   // 1. 게시글 등록
   Article article = articleRepository.save(Article.create(idGenerator.getId(), request));
   
   // 2. Spring Event 발행
   eventPublisher.registeredEventPublish(article.createRegisteredEventPayload());
   return article;
}

@Component
@RequiredArgsConstructor
@Slf4j
public class EventListener {
    private final EventUpdater eventUpdater;

    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    public void beforeCommitEvent(EventData eventData) {
        eventUpdater.save(eventData);
    }
}
```
`@TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)`를 통해 같은 트래잭션에서 비즈니스 데이터 와 Outbox 테이블 이벤트를 저장하게됩니다.

<br>

## 테스트
이제 CDC 설정이 완료됐기 때문에 Outbox 테이블에 이벤트가 저장되면 즉시 카프카에 이벤트가 전송되게 됩니다. 테스트를 통해 확인해보겠습니다.

게시글 등록 요청을 전송하고 [kafka-ui](http://localhost:8989) 에 접속해보면 토픽을 확인할 수 있습니다.

<img src="/assets/images/49/3.png"  width="550" height="240"/>
<img src="/assets/images/49/4.png"  width="550" height="330"/>

자동 생성된 Storage 토픽과 실제 데이터가 저장되는 **article** 토픽이 생성된걸 확인할 수 있습니다.




<br>

## 모니터링

CDC 시스템은 실시간으로 데이터를 처리하기 때문에 안정적인 운영을 위해서는 모니터링이 필요합니다.  
Debezium은 JMX(Java Management Extensions)를 통해 다양한 메트릭을 제공하며, 이를 **Prometheus**와 **Grafana**로 시각화할 수 있습니다.

<br>

### 모니터링 메트릭

Debezium은 크게 **스냅샷 메트릭**과 **스트리밍 메트릭** 두 가지 범주의 메트릭을 제공합니다.  
<br>

- **스냅샷 메트릭 (Snapshot Metrics)**  
초기 스냅샷 단계에서 수집되는 메트릭으로, 커넥터가 처음 시작될 때 기존 데이터를 캡처하는 과정을 모니터링합니다.
초기 스냅샷이 완료되면 이 메트릭들은 더 이상 업데이트되지 않습니다.

- **스트리밍 메트릭 (Streaming Metrics)**  
실시간으로 트랜잭션 로그를 읽어 변경 사항을 스트리밍하는 단계에서 수집되는 메트릭입니다.


<br>

<img src="/assets/images/49/5.png"  width="1000" height="410"/>
Outbox 패턴에서는 실시간성이 중요하므로 스트리밍 메트릭 위주로 모니터링을 구성했습니다.

모니터링 대한 자세한 설명 및 코드는 공식 문서를 참고 부탁드립니다.
- [debezium 모니터링 공식 문서 링크](https://debezium.io/documentation/reference/stable/operations/monitoring.html#monitoring-debezium)
- [debezium 모니터링 설정 예시 코드](https://github.com/debezium/debezium-examples/tree/main/monitoring)


<br>

---
## Reference
<https://debezium.io/documentation/reference/stable/connectors/postgresql.html#postgresql-monitoring>