---
layout: post
title: "Kafka - Transaction Producer , Consumer"
author: "jhkim593"
tags: Kafka
---
# 트랜잭션 프로듀서

<img src="/assets/images/34/1.png"  width="550" height="250"/>

트랜잭션 프로듀서는 여러 메세지 전송 작업을 하나의 원자적 단위로 묶어 처리하는 기능을 의미합니다. **즉 다수의 데이터를 동일 트랜잭션으로 묵음으로써 전체 데이터를 처리하거나 전체 데이터를 처리하지 않도록합니다.**

트랜잭션 프로듀서는 데이터를 레코드로 파티션에 저장할뿐 아니라 트랜잭션의 시작과 끝을 나타내기위해 트랜잭션 레코드를 한개 더 보냅니다.

<br>
## 설정

첫번째로 트랜잭션 프로듀서로 동작하기 위해 **`transactionId`를 설정**해야합니다. `transactionId`는 프로듀서별로 고유한 ID 값을 사용해야합니다.

`transactionId` 설정 예시입니다.
```java
configs.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, UUID.randomUUID());
```

<br>
또한 **멱등성 프로듀서 조건을 만족**해야합니다. 멱등성 프로듀서의 관해서는 [이전글](https://jhkim593.github.io/2024-09-15/33-Kafka_idempotence_producer)을 참고해주세요.

멱등성 프로듀서 조건이 만족되지 않았을 때 위와 같이 `ConfigException`이 발생하게됩니다.
<img src="/assets/images/34/2.png"  width="800" height="50"/>

<br>
# 트랜잭션 컨슈머

트랜잭션 컨슈머는 파티션에 저장된 트랜잭션 레코드를 확인하고 트랜잭션 상태를 확인해 데이터를 가져옵니다.

<br>
### isolation.level

트랜잭션으로 묶인 메세지를 어떻게 읽을지 정할 수 있으며 2가지 옵션이 있습니다.

`read_uncommitted` : 커밋여부와 상관없이 모든 메세지를 읽습니다. 카프카 클라이언트 3.8.0 기준 default 값입니다.

`read_committed`  : 커밋이 완료된 메세지만 읽습니다. 해당 설정은 트랜잭션 프로듀서와 함께 사용합니다.

<br>
# 카프카 트랜잭션 테스트

트랜잭션 처리를 적용한 프로듀서와 컨슈머를 작동시켜보면서 메세지를 어떻게 주고받는지 테스트 해보도록하겠습니다.

> 카프카 클라이언트 3.8.0 기준으로 테스트 진행했습니다.
>

<br>
먼저 컨슈머 클래스입니다.

TransactionConsumer.class

```java
public class TransactionConsumer {
    private final static Logger logger = LoggerFactory.getLogger(SimpleConsumer.class);
    private final static String TOPIC_NAME = "transaction-test";
    private final static String BOOTSTRAP_SERVERS = "localhost:9092";
    private final static String GROUP_ID = "test-group";

    public static void main(String[] args) {
        Properties configs = new Properties();
        configs.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        configs.put(ConsumerConfig.GROUP_ID_CONFIG, GROUP_ID);
        configs.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        configs.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(configs);

        consumer.subscribe(Arrays.asList(TOPIC_NAME));

        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
            for (ConsumerRecord<String, String> record : records) {
                logger.info("record:{}", record);
            }
        }
    }
}
```
`transaction-test` 토픽을 구독해 메세지를 받아오는 컨슈머입니다. 받아온 메세지를 로그를 통해 확인할 수 있도록 했습니다.

<br>
다음으로 프로듀서 클래스입니다.

TransactionProducer.class
```java
public class TransactionProducer {
    private final static Logger logger = LoggerFactory.getLogger(TransactionProducer.class);
    private final static String TOPIC_NAME = "transaction-test";
    private final static String BOOTSTRAP_SERVERS = "localhost:9092";

    public static void main(String[] args) {

        Properties configs = new Properties();
        configs.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        configs.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        configs.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        configs.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "1");

        try (KafkaProducer<String, String> producer = new KafkaProducer<>(configs)){

            //트랜잭션 init , begin
            producer.initTransactions();
            producer.beginTransaction();

            String messageValue1 = "testMessage1";
            ProducerRecord<String, String> record1 = new ProducerRecord<>(TOPIC_NAME, messageValue1);
            producer.send(record1);
            logger.info("{}", record1);

            String messageValue2 = "testMessage2";
            ProducerRecord<String, String> record2 = new ProducerRecord<>(TOPIC_NAME, messageValue2);
            producer.send(record2);
            logger.info("{}", record2);

            //메세지 전송 후 커밋
            producer.commitTransaction();
        }
    }
}
```

`initTransactions()`  , `beginTransaction()` 을 통해 트랜잭션을 시작하고 record1 , record2 두개 레코드를 `transaction-test` 토픽에 전송하는 프로듀서입니다.

먼저 컨슈머를 동작시킨 뒤 프로듀서로 메세지를 전송시켜보겠습니다.

<img src="/assets/images/34/3.png"  width="800" height="70"/>

로그를 확인해보면 컨슈머에 메세지가 잘 넘어온것을 확인 할 수 있습니다.

<br>

#### 프로듀서에서 트랜잭션 커밋전 예외가 발생한다면?

프로듀서에서 트랜잭션 커밋전 메세지가 발생한다면 컨슈머에서는 어떻게 동작하는지 살펴보겠습니다.

트랜잭션 커밋전 `throw new RuntimeException();` 을 추가해 의도적으로 예외를 발생 시켜보겠습니다.

```java
try (KafkaProducer<String, String> producer = new KafkaProducer<>(configs)){

	//트랜잭션 init , begin
	producer.initTransactions();
	producer.beginTransaction();

	String messageValue1 = "testMessage1";
	ProducerRecord<String, String> record1 = new ProducerRecord<>(TOPIC_NAME, messageValue1);
	producer.send(record1);
	logger.info("{}", record1);


	String messageValue2 = "testMessage2";
	ProducerRecord<String, String> record2 = new ProducerRecord<>(TOPIC_NAME, messageValue2);
	producer.send(record2);
	logger.info("{}", record2);

	//예외 발생
	if (true) throw new RuntimeException();

	//메세지 전송 후 커밋
	producer.commitTransaction();
}
```
<br>

 변경된 코드로 프로듀서 실행 후 컨슈머 로그를 보면

<img src="/assets/images/34/4.png"  width="800" height="130"/>

컨슈머가 이전에 수신한 메시지 2개와 함께, 방금 받은 메시지 2개를 포함하여 총 4개의 메시지를 수신했음을 확인할 수 있습니다.

트랜잭션이 실패했어도 메세지가 받아진 이유는 컨슈머 `ISOLATION_LEVEL` default 값이`read_uncommitted` 라서 **커밋이 완료되지 않아도 메세지를 받을 수 있기 때문**입니다.

트랜잭션 실패시 (커밋 되지 않았을때 ) 컨슈머에서 메세지 처리도 허용하지 않으려면 `ISOLATION_LEVEL` 를 `read_committed` 로 바꿔 **커밋된 메세지만 처리할 수 있도록 설정**하면됩니다.

`read_committed` 으로 바꾸기 위해서는 아래 설정을 추가하면됩니다.

```java
configs.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
```

컨슈머 설정 추가 후 재실행하고 트랜잭션 커밋전 예외 발생하는 프로듀서를 다시 실행해보면 컨슈머로 메세지가 넘어오지 않는것을 확인 할 수 있습니다.
