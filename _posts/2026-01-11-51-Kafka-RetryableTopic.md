---
layout: post
title: "Kafka - Consumer 실패시 처리"
author: "jhkim593"
tags: Kafka
---

## Consumer 실패 처리가 필요한 이유

Kafka Consumer에 적절한 실패처리가 적용되어있지 않다면 메시지 처리 실패가 전체 처리 흐름에 영향을 줄 수 있습니다.

예를들어 예외가 발생했을 때 이를 그대로 무시하면 메시지가 유실되고, 반대로 같은 메시지를 계속 재시도하면 해당 파티션의 메시지들이 처리되지 못한 채 대기 상태에 놓이게 됩니다.

따라서 Consumer 실패 처리는 메시지 유실을 방지하면서도 정상적인 처리 흐름을 방해하지 않도록 설계되어야 합니다.

<br>


## 실패 처리 전략

효율적인 실패 처리를 위해 예외의 성격에 따라 재시도 여부를 구분하고, 처리에 실패한 메시지는 DLT로 전송하여 별도 관리하도록 했습니다.

<br>

#### DLT(Dead Letter Topic)란?
DLT는 정상 처리에 실패한 메시지를 정상 처리 흐름과 분리해 보관하기 위한 토픽입니다.
이를 통해 실패 메시지로 인한 처리 흐름 차단을 방지하고, 메시지 유실 없이 데이터베이스 저장이나 관리자 알림과 같은 후속 처리가 가능하게 됩니다.

<br>

#### 예외 유형 구분

- **재시도 제외 유형**

    비즈니스 검증 실패나 데이터 파싱 실패와 같이 재시도해도 계속 실패하는 경우입니다.   
  이러한 예외들은 `NoRetryableException`로 정의해 재시도 없이 바로 DLT로 전송합니다.


- **일시적 장애 유형**

  외부 API 서버의 일시적인 장애나 데이터베이스 연결 실패처럼 시간이 지나면 자연스럽게 복구될 수 있는 상황에서는 **재시도**를 수행합니다.
  재시도 횟수를 모두 소진했음에도 여전히 실패한다면 더 이상 일시적인 문제로 보기 어렵다고 판단해 DLT로 전송합니다.

  발생 가능한 예외가 다양하기 때문에 **재시도 불가능한 유형을 제외한 모든 예외**를 대상으로 했습니다.

<br>
이제 위 기준을 Kafka Consumer에 적용해보도록 하겠습니다.

<br>
## DefaultErrorHandler를 통한 재시도 처리

```java
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<String, String> kafkaTemplate) {
    // 1
    DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(kafkaTemplate);

    // 2.
    ExponentialBackOffWithMaxRetries backOff = new ExponentialBackOffWithMaxRetries(2);
    backOff.setInitialInterval(3000L);
    backOff.setMultiplier(2);

    DefaultErrorHandler errorHandler = new DefaultErrorHandler(recoverer, backOff);

    // 3
    errorHandler.addNotRetryableExceptions(NoRetryableException.class);

    return errorHandler;
}

@Bean
public ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory(DefaultErrorHandler errorHandler) {
    ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory());
    //4
    factory.setCommonErrorHandler(errorHandler);
    return factory;
}
```
1. **DeadLetterPublishingRecoverer**: 모든 재시도 시도가 실패했을 때, 메시지를 DLT로 전송합니다.
2. **BackOff 설정**: `ExponentialBackOffWithMaxRetries`를 사용하여 재시도 간격을 지수적으로 증가시켜 최대 2번 재시도합니다.
3. **재시도 제외 설정**: `addNotRetryableExceptions`를 통해 재시도하지 않고 즉시 DLT로 전송 할 예외를 지정합니다.
4. **ErrorHandler 등록**: 정의한 에러 핸들러를 Consumer에 적용합니다.

<br>
하지만 위 방식은 **블로킹으로 인한 문제가 발생**할 수 있습니다.

Consumer가 메시지 처리 중 예외가 발생하면 같은 파티션에서 계속 재시도를 수행하는데, **재시도하는 동안 해당 파티션의 다음 메시지들이 모두 대기 상태**가 되어 처리되지 못합니다. 
결과적으로 Consumer Lag이 증가하고 전체 처리량이 감소하게 됩니다.


<br>
## RetryableTopic을 사용한 재시도 처리

Spring-Kafka에서 제공하는 **Non-blocking retry** 기능입니다.

앞서 살펴본 DefaultErrorHandler의 블로킹 문제를 해결하기 위해 실패한 메시지를 **별도의 retry 토픽으로 전송**하여 재시도를 수행합니다.

<br>

동작 방식은 다음과 같습니다.
- 메인 토픽에서 메시지 처리 실패 시 해당 메시지만 retry 토픽으로 전송 (메인 토픽은 다음 메시지를 계속 처리함)
- retry 토픽에서 재시도 수행
- 재시도 실패 시 다음 retry 토픽으로 전송
- 모든 재시도 실패 시 최종적으로 DLT로 전송

이를 통해 **실패한 메시지가 다음에 처리 할 메시지들을 블로킹하지 않으며**, 정상 메시지들은 중단 없이 처리됩니다.

<br>
RetryableTopic를 사용해 구현한 코드는 다음과 같습니다.

```java
public class OrderEventConsumer {

    @RetryableTopic(
        attempts = "3",                   
        backoff = @Backoff(               
            delay = 3000,
            multiplier = 2.0
        ),
        topicSuffixingStrategy = TopicSuffixingStrategy.SUFFIX_WITH_INDEX_VALUE, 
        exclude = {NoRetryableException.class}
    )
    @KafkaListener(topics = "order-topic")
    public void listen(String message) {
        ...
    }

    @DltHandler
    public void handleDlt(String message, @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {
        log.error("DLT 메시지 처리 - topic: {}, message: {}", topic, message);
        // DB 저장, 알림 전송 등 후속 처리
    }
}
```
- **attempts**: 재시도 횟수를 지정합니다. `attempts = "3"`은 원본 1회 + 재시도 2회를 의미합니다.
- **backoff**: 재시도 간격을 설정합니다. `delay`는 초기 지연 시간, `multiplier`는 재시도마다 지연 시간을 증가시키는 배수를 의미합니다.
- **topicSuffixingStrategy**: retry 토픽의 네이밍 전략을 설정합니다. `SUFFIX_WITH_INDEX_VALUE`를 사용하면 `order-topic-retry-0`, `order-topic-retry-1` 형태로 retry 토픽이 생성됩니다.
- **exclude**: 재시도하지 않고 즉시 DLT로 전송 할 예외를 지정합니다. `NoRetryableException`이 발생하면 재시도 없이 바로 DLT로 전송됩니다.
- **@DltHandler**: `@RetryableTopic`이 적용된 Consumer와 같은 클래스에 위치해야 하며, 모든 재시도가 실패하여 DLT로 전송된 메시지를 처리합니다.

<br>

####  RetryableTopic 공통 설정

여러 Consumer에 동일한 실패 처리 전략을 적용하고 싶을 때 `@RetryableTopic`을 각 Consumer마다 개별적으로 설정하는 것은 비효율적 일 수 있습니다. 이런 경우 글로벌 설정을 통해 모든 Consumer에 공통으로 적용할 수 있습니다.

```java
@Configuration
public class KafkaRetryConfig {

    @Bean
    public RetryTopicConfiguration retryTopicConfiguration(KafkaTemplate<String, Object> template) {
        return RetryTopicConfigurationBuilder
                .newInstance()
                .maxAttempts(3)
                .exponentialBackoff(3000, 2.0, 10000)
                .notRetryOn(NoRetryableException.class)
                .suffixTopicsWithIndexValues()
                .dltHandlerMethod(new EndpointHandlerMethod(DltMessageHandler.class, "handleDltMessage"))
                .create(template);
    }
}
```
- **dltHandlerMethod**: DLT 메시지를 처리할 클래스와 메서드를 지정합니다. `@DltHandler`를 각 Consumer마다 작성할 필요 없이 공통 DLT 처리 로직을 하나의 클래스로 관리할 수 있습니다.



