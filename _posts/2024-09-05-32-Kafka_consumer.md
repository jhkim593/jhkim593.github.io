---
layout: post
title: "Kafka - Consumer"
author: "jhkim593"
tags: Kafka
---
# Consumer

컨슈머는 적대된 데이터를 사용하기 위해 카프카 브로커로부터 **토픽에 저장된 데이터를 가져와 처리하는 역할**을 합니다.

<br>
### 컨슈머 내부 구조

<img src="/assets/images/32/1.png"  width="600" height="250"/>

- fetcher : 리더 파티션으로부터 레코드을 **미리 가져와서 대기**
- poll : fetcher에 있는 레코드들을 리턴
- consumerRecords : 처리하고있는 레코들의 모음이며 오프셋이 포함되어있습니다.

<br>
# Consumer Group

<img src="/assets/images/32/2.png"  width="350" height="300"/>

컨슈머들이 1개 이상 모여 컨슈머 그룹을 구성합니다. 컨슈머 그룹은 각 컨슈머 그룹으로부터 **격리된 환경에서 서로 영향을 주고받지 않도록 동작**합니다.

예를들어 T1 토픽을 구독 중인 컨슈머 그룹 1이 존재하고있는 상황에서 컨슈머 그룹 2가 새로 추가되어도 컨슈머 그룹2는 기존 컨슈머 그룹1의 작업과 전혀 상관없이 T1 토픽에 메세지를 모두 처리할 수 있습니다.

컨슈머 그룹이 토픽을 구독해서 데이터를 가져갈 때 **1개의 파티션은 최대 1개 컨슈머에 할당 가능**합니다. 반대로 **1개 컨슈머는 여러개의 파티션에 할당** 될 수 있습니다. 컨슈머가 파티션 개수보다 많을 경우 유휴 상태인 컨슈머가 존재하기 때문에 **컨슈머 그룹의 컨슈머 개수는 토픽의 파티션 개수보다 작거나 같아야 효율적**입니다.

<br>
# 리밸런싱

컨슈머 그룹 내의 컨슈머들은 자신들이 읽은 파티션의 소유권을 공유할 수 있습니다. 즉 **컨슈머 1이 할당받은 파티션 읽기 작업을 컨슈머 2가 이관받아서 처리**할 수 있습니다. 컨슈머 그룹 내에서 이루어지는 이관 작업을 리밸런싱이라고 합니다.

리밸런싱은 크게 두가지 상황에서 일어납니다.

- 컨슈머 그룹에 컨슈머가 추가되는 상황
- 컨슈머 그룹에 컨슈머가 제외되는 상황
- 파티션이 추가 됐을 때

**리밸런싱이 발생한 컨슈머 그룹내 모든 읽기 작업은 중단**되게되는데 파티션 개수가 적을때는 중단되는 시간이 짧기 때문에 문제가 되지 않지만 파티션 개수가 많다면 중단 되는 시간 또한 길어질 수 있기 때문에 컨슈머 , 파티션의 변경 , 추가시에는 주의가 필요합니다.  

<br>
# 컨슈머 주요 옵션

### 필수 옵션
- bootstrap.servers : 브로커에 host:port를 설정 2개이상 입력해서 일부 브로커 이슈 발생해도 문제없도록하는것이 일반적임.
- key.deserializer : 레코드 메세지키 역직렬화 클래스 지정
- value.deserializer : 레코드 메세지값 역직렬화 클래스 지정

<br>
### 선택 옵션
- group.id : 컨슈머 그룹 id를 지정 , subscribe 메소드에서는 필수
- auto.offset.reset : 컨슈머 오프셋이 없는경우 (커밋을 한적 없는 경우) 어느 오프셋부터 읽을지 선택. **이미 컨슈머 오프셋이 있으면 무시되므로 자주 사용되지는 않음**
    - latest : 기본값이며 가장 최근에 넣은 오프셋읽기 시작
    - earliest 가장 오래전에 넣은 오프셋 읽음
    - none : 이전 오프셋값을 찾지 못하면 에러 발생
- enable.auto.commit : 자동 커밋 설정, 기본값은 true
- auto.commit.interval.ms : 자동 커밋일때 오프셋 커밋 간격 지정 , 컨슈머에서 poll() 메소드가 호출됐을 때 auto.commit.interval.ms이 지났는지 확인하고 지났다면 자동 커밋을 수행
- max.poll.records : poll 메소드를 통해 반환되는 레코드수
- session.timeout.ms : 컨슈머가 브로커와 연결이 끊기는 최대시간  , 컨슈머는 브로커에 주기적으로 하트비트를 보내는데 하트비트가 session.timeout.ms 이상 전송 되지 않으면 컨슈머 그룹에서 해당 컨슈머를 제외시키고 리밸런싱이 이루어집니다.
- heartbeat.interval.ms : 하트비트 전송 주기를 설정 , session.timeout.ms보다는 낮아야합니다.
- max.poll.interval.ms : poll 메소드 호출 간격의 최대 시간  ,  컨슈머가 하트비트만 보내고 실제 데이터 처리를 안하고 있을 수있으므로 **시간을 초과하면 리밸런싱이 이루어집니다. 만약 데이터 처리를 안하고 있는게 아니라 오래 걸리는 것이라면** 원치 않는 리밸런싱 일어날수있습니다. 일반적으로 기본값을 사용해도 무방합니다.
- isolation.level : 트랜잭션 프로듀서가 레코드를 트랜잭션 단위로 보낼경우 사용 (트랜잭션에 대해서는 다음에 자세하게 다루도록 하겠습니다.)

<br>
# Commit

컨슈머가 카프카 브로커로부터 **데이터를 어디까지 가져갔는지 오프셋을 기록**하는 작업을 커밋이라고합니다. 커밋을 통해 컨슈머는 이미 읽은 데이터는 제외하고 아직 읽지 않은 데이터를 가져올 수 있게됩니다.

카프카에서는 파티션 별로 컨슈머 그룹 오프셋 정보를 저장하기 위해서 별도로 **__consumer_offsets 토픽을 생성해 관리**합니다.

만약 커밋된 오프셋이 실제 마지막으로 처리한 오프셋보다 작으면 메세지는 중복 처리 될 수 있수있으며

반대로 커밋된 오프셋이 마지막으로 처리된 오프셋보다 크다면 메세지 누락이 발생 할 수있습니다. 이런 문제 발생을 방지하기 위해 정상적으로 커밋을 하는것이 중요합니다.


커밋은 크게 **자동 커밋**과 **수동 커밋**으로 나뉩니다.

<br>
### 자동 커밋 (**enable.auto.commit = true**)

카프카 클라이언트 3.8 기준 기본값입니다.
컨슈머는 poll() 메소드를 호출할 때마다 `auto.commit.interval.ms`에 설정된 주기를 체크해 자동 커밋을 수행합니다.

개발이 쉽고 처리 속도가 빠르다는 장점이 있지만 메세지 중복 처리 및 유실이 발생할 수 있습니다.

**메세지 유실**

자동 커밋 주기가 1초일 때 메세지를 poll 하고 처리중인 컨슈머가 1초를 초과한 후 장애 발생으로 죽은 경우 이미 커밋을 했기 때문에 해당 메세지는 유실되게됩니다.

**메세지 중복**

자동 커밋 주기가 5초일때 메세지를 poll하고 컨슈머가 처리 완료했지만 5초이내에 죽게되면 커밋은 하지 못했기 때문에 이후에 메세지 중복 처리 문제가 발생할 수 있습니다.

<br>
### 수동 커밋 (**enable.auto.commit = false**)

오프셋이 자동으로 커밋되지 않고 수동으로 커밋 메소드를 호출해야합니다.

수동 커밋은 크게 **동기 커밋** , **비동기 커밋**으로 나눌수 있습니다.

- 동기 커밋 :  `commitSync()`
    - `poll()` 메소드로 받은 ConsumerRecord 의 마지막 offset을 커밋
    - 커밋이 완료될 때 까지 대기 후 다음 작업을 진행
- 비동기 커밋 : `commitAsync()`
    - 커밋 응답 기다지리 않고 다음 작업을 수행
    - 커밋 응답을 기다리지 않기 때문에 성공 실패 여부를 바로 알 수없고 콜백을 통해 확인 할 수 있음

하지만 **수동 커밋의 경우에도 메세지 중복이 발생** 할 수 있습니다. 예를 들어 데이터 처리가 끝난 후 문제가 생겨 커밋이 정상적으로 진행되지 않았다면 메세지를 다시 읽어 중복 처리를 하게 될 수 있습니다.

<br>
# 컨슈머 랙
<img src="/assets/images/32/3.png"  width="600" height="160"/>

**파티션 최신 오프셋과 컨슈머 오프셋간에 차이를 의미** 하는데 컨슈머가 정상적으로 데이터를 처리하고 있는지 확인하는 중요 지표이기 떄문에 **필수로 모니터링** 해야합니다.

컨슈머 랙은 컨슈머 그룹과 토픽 , 파티션 별로 생성됩니다. 예를 들어 1개 토픽에 3개 파티션이 있고 하나의 컨슈머 그룹이 데이터를 받아 처리하고 있을때 컨슈머 랙은 3개가됩니다.

프로듀서가 보내는 데이터양이 컨슈머 에서 처리하는 양보다 많으면 컨슈머 랙은 늘어날 것이며 반대로 프로듀서가 보내는 양이 컨슈머 처리량보다 적으면 컨슈머 랙은 줄어들게됩니다.

<br>
# **Kafka CLI를 통한 프로듀서 테스트**

### kafka-console-consumer.sh

```bash
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic hello.kafka --from-beginning --max-messages 2
```

`--from-beginning` : 특정 토픽 첫번째 메세지 부터 읽음

`--max-messages` : 읽을 메세지 수 제한 설정 해당 명령어로 특정 토픽 처음부터 얼마나 메세지를 읽을 것인지 설정할 수 있습니다.

<br>
```bash
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic hello.kafka --from-beginning --partition 0
```
`--partition` 를 통해 특정 파티션의 메세지만 읽을 수 있는데 프로듀서에서 보낸 메세지 키가 같으면 같은 파티션으로 들어가기 때문에 이를 테스트 하기 유용합니다.

<br>
```bash
 bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic hello.kafka --group test-group
```
group 옵션을 사용하면 컨슈머 그룹 기반으로 동작하게됩니다.

<br>
추가로 토픽에서 받은 메세지의 메세지 키와 메세지 값을 모두 확인 할 수 있습니다.

```bash
bin/kafka-console-consumer.sh  --bootstrap-server localhost:9092 --topic test --property "print.key=true" --property "key.separator=:" --from-beginning
```

property에 `print.key=true` 와 `key.separator` 를 설정해 메세지키를 포함해서 읽을 수 있습니다. 위 예시에서는 separator가 : 이므로 key : value 형식으로 메세지를 읽게 됩니다.

<br>
### Kafka-consumer-groups.sh

```bash
kjh@:~/kafka_2.12-3.8.0$ bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group test-group --describe

Consumer group 'test-group' has no active members.

GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
test-group     hello.kafka     0          13              13              0               -               -               -
```
`--describe` 를 통해 컨슈머 그룹 상세 정보 조회가 가능합니다.

여기서 해당 토픽에 메세지를 추가로 저장하게되면 LAG이 올라가는것을 알수 있습니다.

```bash
GROUP           TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
test-group     hello.kafka     0          13              19              6               -               -               -
```

<br>
오프셋 리셋 또한 가능한데 현재 컨슈머 그룹이 읽고있는 오프셋을 특정위치로 이동할 수 있는것을 의미합니다.

```bash
 bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --topic hello.kafka --group test-group --reset-offsets --to-earliest --execute
```
해당 커맨드를 사용하게되면 오프셋이 초기화되게 됩니다.

<br>
# **Kafka 컨슈머 어플리케이션 테스트**

**build.gradle**

```groovy
implementation 'org.apache.kafka:kafka-clients:3.8.0'
```

테스트에 kafka-clients:3.8.0 버전을 사용했습니다.
<br>

### 자동 커밋 컨슈머

```java
public class ConsumerWithAutoCommit {
    private final static Logger logger = LoggerFactory.getLogger(ConsumerWithAutoCommit.class);
    private final static String TOPIC_NAME = "test";
    private final static String BOOTSTRAP_SERVERS = "localhost:9092";
    private final static String GROUP_ID = "test-group";

    public static void main(String[] args) {
	    //컨슈머 properties 설정
        Properties configs = new Properties();
        configs.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        configs.put(ConsumerConfig.GROUP_ID_CONFIG, GROUP_ID);
        configs.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        configs.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        configs.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, true);
        configs.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, 60000);

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(configs);
        consumer.subscribe(Arrays.asList(TOPIC_NAME));

		// 무한루프 돌며 poll 수행
        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
            for (ConsumerRecord<String, String> record : records) {
                logger.info("record:{}", record);
            }
        }
    }
}
```

consumer.poll() 메소드에 파라미터로 타임아웃을 지정하는데 **타임아웃이 지나도록 데이터를 받지못하면 poll은 빈 결과를 반환**하게되며 만약 대기 중 토픽에 메세지가 들어온다면 poll 메소드는 즉시 ConsumerRecords를 반환하게됩니다.

<br>
### 비동기 커밋 컨슈머

```java
public class ConsumerWithASyncCommit {
    private final static Logger logger = LoggerFactory.getLogger(ConsumerWithASyncCommit.class);
    private final static String TOPIC_NAME = "test";
    private final static String BOOTSTRAP_SERVERS = "localhost:9092";
    private final static String GROUP_ID = "test-group";

    public static void main(String[] args) {
		//컨슈머 properties 설정
        Properties configs = new Properties();
        configs.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        configs.put(ConsumerConfig.GROUP_ID_CONFIG, GROUP_ID);
        configs.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        configs.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        configs.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        KafkaConsumer<String, String> consumer = new KafkaConsumer<>(configs);
        consumer.subscribe(Arrays.asList(TOPIC_NAME));

		// 무한루프 돌며 poll 수행
        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
            for (ConsumerRecord<String, String> record : records) {
                logger.info("record:{}", record);
            }
            consumer.commitAsync(new OffsetCommitCallback() {
                public void onComplete(Map<TopicPartition, OffsetAndMetadata> offsets, Exception e) {
                    if (e != null)
                        System.err.println("Commit failed");
                    else
                        System.out.println("Commit succeeded");
                }
            });
        }
    }
}
```
비동기 커밋을 위해 `ENABLE_AUTO_COMMIT_CONFIG`를 false로 설정했고 커밋 수행 결과 확인을 위해 OffsetCommitCallback를 구현해 콜백 객체를 추가했습니다.

정상적으로 커밋되었다면 exception 파라미터는 null이며 예외가 발생해다면 exception 파라미터로 실패 이유를 확인 할 수 있습니다.
