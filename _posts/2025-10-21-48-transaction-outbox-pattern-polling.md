---
layout: post
title: "트랜잭션 Outbox 패턴"
author: "jhkim593"
tags: Architecture
---
마이크로서비스 아키텍처에서는 각 서비스가 독립적인 데이터베이스를 사용하기 때문에, 서비스 간 데이터 일관성을 유지하는 것이 쉽지 않습니다.

일관성을 위해서는 DB에 데이터를 저장하거나 수정하면서 동시에 메시지 브로커로 이벤트를 정상 발행해야 합니다. 만약 DB 작업은 성공했지만 메세지 브로커 장애 등으로 이벤트 발행에 실패하면, 전체 시스템의 데이터 정합성이 깨지는 문제가 발생할 수 있는데

이번 게시글에서는 이 문제를 해결하기 위한 방식 중 하나인 Transaction Outbox Pattern에 대해 정리해보았습니다.

<br>

## 기존 문제점

일반적으로 애플리케이션에서 데이터를 저장하고 이벤트를 발행하는 로직은 다음과 같은 형태로 구현됩니다.

```java
@Service
@RequiredArgsConstructor
public class ArticleService {
    private final ArticleRepository articleRepository;
    private final KafkaTemplate kafkaTemplate;

    public void createArticle(Article article) {
        // 1. DB에 게시글 저장
        articleRepository.save(article);

        // 2. Kafka에 이벤트 발행
        kafkaTemplate.send("article", new ArticleCreatedEvent());
    }
}
```

하지만 위 코드에는 다음과 같은 문제점이 있습니다.

<br>

### 데이터 일관성 문제

**DB 저장은 성공했지만 메세지 발행이 실패하는 경우**

DB에 게시글 데이터가 저장되었지만 네트워크 장애나 메세지 브로커 장애로 인해 이벤트 발행에 실패할 수 있습니다. 이 경우 다른 서비스는 게시글이 생성되었다는 사실을 알 수 없게 됩니다.

**메세지 발행은 성공했지만 DB 저장이 실패하는 경우**

DB 트랜잭션이 롤백됐지만 이벤트는 발행됐을 수 있습니다. 이 경우 다른 서비스는 존재하지 않는 게시글에 대한 이벤트를 수신하게 됩니다.



## Transaction Outbox Pattern
앞서 말씀드린 데이터 일관성 문제는 DB와 메세지 브로커가 서로 다른 시스템이기 때문에 원자성을 보장하기 어려워 발생하는데 이를 해결하기 위한 방법으로 Transaction Outbox Pattern이 있습니다.

Transactional Outbox Pattern이란, 데이터 업데이트와 이벤트 저장을 하나의 트랜잭션으로 묶어 데이터 일관성을 보장하는 패턴입니다.
이 패턴의 핵심은 발행해야 할 이벤트를 별도의 Outbox 테이블에 저장하고, 이를 별도 프로세스가 읽어서 메세지 브로커로 발행하는 것입니다.

<br>

구현 방법은 다음과 같습니다.

1. 이벤트를 저장하는 별도 Outbox 테이블을 생성합니다.
2. 비즈니스 데이터와 이벤트를 동일한 DB 트랜잭션 내에서 저장합니다.
3. 별도의 프로세스가 Outbox 테이블을 읽어서 메세지 브로커로 이벤트를 발행합니다.
4. 발행이 완료된 이벤트는 Outbox 테이블에서 삭제하거나 발행 완료 상태로 변경해 다시 발행되지 않도록 합니다.

두 작업이 하나의 트랜잭션으로 묶여있기 때문에 만약 데이터 업데이트가 성공적으로 이루어지더라도 Outbox 저장에 실패하면 전체 트랜잭션이 롤백되기 때문에 데이터 일관성을 보장할 수 있게 됩니다.


여기까지 Transaction Outbox Pattern을 살펴보면 다음과 같은 의문이 생길 수도 있습니다.

```java
@Service
@RequiredArgsConstructor
public class ArticleService {
    private final ArticleRepository articleRepository;
    private final KafkaTemplate kafkaTemplate;

    @Transactional
    public void createArticle(Article article) {
        // 1. DB에 게시글 저장
        articleRepository.save(article);

        // 2. Kafka에 이벤트 발행
        kafkaTemplate.send("article", new ArticleCreatedEvent());
    }
}
```

위 코드는 상위 메소드에 @Transactional을 적용해 데이터 업데이트, 이벤트 발행을 같은 트랜잭션으로 묶은 것인데
이벤트 발행이 실패해 예외가 발생하게 되면 DB 또한 롤백이 진행돼 일관성을 유지할 수 있을 것입니다.

하지만 다음과 같은 문제점이 존재합니다.
1. 불필요한 트랜잭션 롤백
   사용자가 게시글 등록 정보를 정상적으로 입력했어도, 이벤트 발행에 실패하면 게시글 등록까지 함께 실패하게 됩니다.
2. 외부 시스템 의존성으로 인한 위험
   외부 시스템에 문제가 발생했을 때 DB 커넥션과 요청 스레드가 외부 시스템의 응답이 올 때까지 대기하게 될 수 있어 최악의 경우 서버 전체의 장애로 이어질 수 있습니다.

위와 같은 문제로 해당 방식은 적절하지 않습니다.


이외에도 Transaction Outbox Pattern은 다음과 같은 특징이 있습니다.

**메세지 브로커 장애 대응**

메세지 브로커에 일시적인 장애가 발생해도 이벤트는 Outbox 테이블에 안전하게 저장되어 있습니다. 장애가 복구되면 저장된 이벤트를 다시 발행할 수 있습니다.

**At-Least-Once 전송 보장**

이벤트가 최소 한 번은 전송되는 것을 보장할 수 있습니다. 이는 네트워크 장애나 재시작 등의 상황에서도 이벤트가 유실되지 않음을 의미합니다. 단, 이벤트가 중복으로 발행될 수 있기 때문에 수신 측에서 멱등성을 보장해야 합니다. 예를 들어 이벤트 ID를 저장해 중복 처리를 방지하거나, 비즈니스 로직 자체를 멱등하게 설계하는 방법이 있습니다.

<br>

## Outbox 테이블 이벤트 처리 방법

Transaction Outbox Pattern에서 Outbox 테이블에 저장된 메세지를 발행하는 방법에는 크게 두 가지가 있습니다.

<br>

### 폴링 방식

별도 프로세스가 주기적으로 Outbox 테이블을 조회하여 미발행된 이벤트를 찾아 메세지 브로커로 발행하는 방식입니다.

**동작 방식**
- 스케줄러가 일정 주기로 Outbox 테이블을 조회합니다.
- 발행되지 않은 이벤트를 찾아 메세지 브로커로 발행합니다.
- 발행이 완료되면 해당 레코드를 삭제하거나 상태를 업데이트합니다.

**장점**
- 구현이 간단합니다.
- 기존 DB 구조에 큰 변경 없이 적용할 수 있습니다.

**단점**
- 폴링 주기에 따라 이벤트 발행 지연이 발생할 수 있습니다.
- 주기적인 DB 조회로 인한 성능 오버헤드가 있습니다.
- 미발행 이벤트가 없어도 계속 조회하므로 비효율적일 수 있습니다.

<br>

### Transaction log tailing 방식

DB의 변경 로그를 모니터링하여 Outbox 테이블에 데이터가 업데이트 되면 즉시 이벤트를 발행하는 방식입니다.

**동작 방식**
- Debezium과 같은 CDC 도구가 DB의 변경 로그를 구독합니다.
- Outbox 테이블에 INSERT가 발생하면 CDC 도구가 이를 감지합니다.
- 감지된 변경 사항을 메세지 브로커로 자동으로 발행합니다.

**장점**
- 실시간에 가까운 이벤트 발행이 가능합니다.
- 애플리케이션 코드 변경 없이 DB 레벨에서 처리됩니다.
- 폴링에 비해 DB 부하가 적습니다.

**단점**
- CDC 도구 설정 및 운영이 복잡합니다.
- DB별로 CDC 구현이 다릅니다.
- 추가 인프라(Debezium, Kafka Connect 등)가 필요합니다.

**CDC를 활용한 방식**은 다음 게시글에 다루도록 하겠습니다.

<br>

## 구현 코드
이 게시글에서는 **폴링 방식**을 사용해 Transaction Outbox Pattern을 구현할 것이며 메세지 브로커로는 kafka, DB는 Postgresql을 사용했습니다.

위에서는 게시글 등록에 대한 이벤트만 다뤘지만 만약 다른 서비스에서 도메인 이벤트가 추가됐을 때 매번 이벤트 발행 기능과 저장소를 다시 구현해야 할 것입니다. 
그렇기 때문에 특정 도메인 이벤트에 종속되지 않는 추상화된 이벤트 구조로 설계했으며, **공통 모듈로 분리해 여러 서비스에서 재사용할 수 있도록 구성**했습니다.

<br>

### 1. Outbox 엔티티

먼저 Outbox 이벤트를 저장할 엔티티를 정의합니다.

**OutboxEvent.java**
```java
@Entity
public class OutboxEvent {
    @Id
    private Long id;
    @Enumerated(EnumType.STRING)
    private EventType type;
    private Long aggregateId;
    @Column(columnDefinition = "TEXT")
    private String payload;
    private boolean published;
    private LocalDateTime createdAt;
}
```
- `type` : 이벤트 타입을 나타내며 enum으로 설정했습니다. (예: ARTICLE_REGISTERED, ARTICLE_UPDATED 등)
- `aggregateId` :  이벤트가 발생한 도메인 엔티티의 식별자입니다. 게시글 등록 이벤트의 경우 해당 게시글의 id가 저장됩니다. Kafka 발행 시 파티션 키로 사용되어 같은 게시글에 대한 이벤트 처리 순서를 보장할 수 있습니다.
- `payload` : 이벤트의 payload를 JSON 형태로 저장합니다. 이벤트에 따라 형식이 달라질 수 있습니다.
- `published` : 저장된 이벤트가 성공적으로 발행되었는지를 나타내는 값입니다.

<br>

### 2. Spring Event를 사용한 이벤트 발행

게시글 등록 시 Spring의 `ApplicationEventPublisher`를 사용해 이벤트를 발행합니다.

**ArticleService.java**
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
```

여기서 Spring Event를 사용한 이유는 **비즈니스 로직에서 이벤트 저장 로직을 분리**하기 위함입니다. 이렇게 분리하게 된다면 이벤트 저장 코드를 변경하더라도 비즈니스 로직 코드는 수정할 필요가 없게됩니다.


추가로 Spring Event 사용하면 `@TransactionalEventListener`를 통해 트랜잭션을 제어할 수 있습니다. 아래는 위에서 Spring Event로 발행한 이벤트를 처리하는 부분입니다.

**EventListener.java**
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class EventListener {
   private final EventUpdater eventUpdater;
   private final EventPublisher eventPublisher;

   @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
   public void beforeCommitEvent(EventData eventData) {
      eventUpdater.save(eventData);
   }
}
```
`@TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)` 를 통해 트랜잭션을 확장해 데이터 업데이트와 Outbox 이벤트 저장을 하나의 트랜잭션으로 묶어 일관성을 보장하게됩니다.

이제 저장된 이벤트를 조회해서 메세지 브로커에 전송해보도록 하겠습니다.
<br>

### 3. 이벤트 발행 및 폴링

**EventPublisher.java**
```java
@Component
@RequiredArgsConstructor
@Slf4j
public class EventPublisher {
   private final EventUpdater eventUpdater;
   private final KafkaTemplate<String, String> kafkaTemplate;

   @Scheduled(
           fixedDelay = 1,
           initialDelay = 5,
           timeUnit = TimeUnit.SECONDS,
           scheduler = "messageRelayPublishPendingEventExecutor"
   )
   public void publishPendingEvent() {
      List<OutboxEvent> outboxEvents = eventUpdater.findPendingEvents();
      for (OutboxEvent outboxEvent : outboxEvents) {
         try {
            publishEvent(outboxEvent.toEventData());
         } catch (Exception e) {
            log.error("pending event publish fail", e);
         }
      }
   }

   public void publishEvent(EventData eventData) throws Exception {
      try {
         kafkaTemplate.send(
                 eventData.getType().getTopic(),
                 eventData.getAggregateId().toString(),
                 eventData.toJson()
         ).get(1, TimeUnit.SECONDS);
         eventUpdater.publishedUpdate(eventData.getId());
      } catch (Exception e) {
         log.error("event send fail", e);
         throw e;
      }
   }
}
```

인덱스 설정 
```sql
CREATE INDEX outbox_event_published_created_at_idx ON outbox_event(published, created_at);
```
데이터 업데이트 , 이벤트 저장 까지 성공 했기 때문에 이후에 `@Scheduled`의 주기를 1초로 설정해서 `published=false`인 이벤트들만 조회해 이벤트 브로커에 전송하게됩니다. 또한 조회 최적화를 위해 Outbox 테이블에 인덱스를 설정했습니다.
폴링 주기는 DB에 부하와 이벤트 발행 지연 사이의 균형을 맞추기 위해 1초로 설정했습니다. 주기가 너무 짧으면 DB에 과도한 조회 부하가 발생할 것이고 반대로 너무 길면 이벤트 발행이 지연되어 실시간성이 떨어질것 입니다.

하지만 여러 서비스가 폴링 주기를 1초로 사용 중이라면 각 서비스마다 최대 1초씩 지연이 누적될 수 있습니다. 예를 들어 게시글 서비스 → 알림 서비스 → 통계 서비스로 이벤트가 전파되는 경우, 최대 3초의 지연이 발생할 수 있습니다.
그래서 폴링 주기로 인한 지연을 줄이기 위해 트랜잭션 커밋 직후 즉시 이벤트를 발행하도록 구현했습니다. 정상적인 경우에는 바로 이벤트가 발행되고, 폴링은 발행 실패 시 재시도 용도로만 사용됩니다.
<br>

**EventListener.java**
```java

@Component
@RequiredArgsConstructor
@Slf4j
public class EventListener {
   private final EventUpdater eventUpdater;
   private final EventPublisher eventPublisher;
   
   //추가 
   @Async("messageRelayPublishEventExecutor")
   @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
   public void afterCommitEvent(EventData eventData) throws Exception {
      try {
         eventPublisher.publishEvent(eventData);
      } catch (Exception e) {
         log.error("event publish fail", e);
         throw e;
      }
   }
}
```
`@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)`를 통해 트랜잭션이 끝나면 즉시 이벤트 발행되고 이벤트가 정상발행됐다면 `published=true`로 설정하게됩니다.


즉시 이벤트가 발행되도록 수정했기 때문에 폴링 주기와 이벤트 조회 로직을 수정했습니다.
- 폴링 주기를 1초에서 10초로 완화
- Outbox 테이블 조회 시 저장된 지 5초 이상 지난 이벤트만 조회되도록 조건을 추가했습니다. 5초 이상 지났다는 것은 즉시 발행에 실패했음을 의미하므로 재시도 대상으로 판단했습니다


```java
@Component
@RequiredArgsConstructor
@Slf4j
public class EventPublisher {
    private final EventUpdater eventUpdater;
    private final KafkaTemplate<String, String> kafkaTemplate;
    
    @Scheduled(
            fixedDelay = 10,  // 폴링 주기 수정 
            initialDelay = 5,
            timeUnit = TimeUnit.SECONDS,
            scheduler = "messageRelayPublishPendingEventExecutor"
    )
    public void publishPendingEvent() {
       List<OutboxEvent> outboxEvents = eventUpdater.findPendingEvents();
       for (OutboxEvent outboxEvent : outboxEvents) {
          try {
             publishEvent(outboxEvent.toEventData());
          } catch (Exception e) {
             log.error("pending event publish fail", e);
          }
       }
    }
 }

@Service
@RequiredArgsConstructor
public class EventUpdateService implements EventUpdater {
    private final EventRepository eventRepository;
   
    ...

    @Override
    public List<OutboxEvent> findPendingEvents() {
        return eventRepository.findAllByCreatedAtLessThanEqualAndPublishedFalseOrderByCreatedAtAsc(
                LocalDateTime.now().minusSeconds(5),  // 저장된지 5초이상 지난 것 조회하도록 조건 추가
                Pageable.ofSize(100)
        );
    }
}
 
 
```
- `@Scheduled`의 `fixedDelay` 옵션을 10으로 수정해 폴링 주기를 완화했습니다.
-  `LocalDateTime.now().minusSeconds(5)` 파라미터를 추가해 저장 후 5초이상 지난 것들을 조회하도록 설정했습니다.

<br>

### 최종 동작 흐름

1. **비즈니스 로직 실행**: `ArticleService.register()`가 호출되어 게시글을 저장하고 Spring의 `ApplicationEventPublisher`를 통해 이벤트를 발행합니다.

2. **BEFORE_COMMIT**: 트랜잭션 커밋 직전에 `EventListener.beforeCommitEvent()`가 호출되어 Outbox 테이블에 이벤트를 저장합니다. 이때 비즈니스 데이터(게시글)와 이벤트가 **동일한 트랜잭션 내에서 저장**되므로 원자성이 보장됩니다.

3. **트랜잭션 커밋**: 게시글 데이터와 Outbox 이벤트가 모두 DB에 커밋됩니다.

4. **AFTER_COMMIT**: 트랜잭션 커밋 직후 `EventListener.afterCommitEvent()`가 비동기로 실행되어 Kafka에 이벤트를 즉시 발행합니다. 성공하면 `published=true`로 업데이트됩니다.

5. **폴링 재시도**: 만약 4번 단계에서 발행이 실패하면, `EventPublisher.publishPendingEvent()` 스케줄러가 10초마다 실행되어 저장된 지 5초 이상 지난 미발행 이벤트를 조회하고 재시도합니다.

위와 같이 동작함으로써 정상적인 경우 즉시 이벤트를 발행하여 지연을 최소화하고, 장애 상황에서는 폴링을 통해 자동으로 재시도하여 안정성을 보장하도록 했습니다.

추가로 Outbox 테이블 이벤트 정리에 대한 고려가 필요합니다. 발행 완료된 이벤트가 계속 쌓이면 테이블 크기가 증가하므로 주기적으로 오래된 이벤트를 삭제하거나 애초에 발행이 성공했다면 바로 삭제하는것도 방법일 것입니다.



### 공통 모듈 구성
테이블 설정, 이벤트 저장, 이벤트 발행 `outbox-polling` 이라는 공통 모듈로 분리해 다른 서비스에서도 재사용 가능하도록 구성했습니다.

build.gradle (article 프로젝트)
```
implementation project(":common:outbox-polling")
```
