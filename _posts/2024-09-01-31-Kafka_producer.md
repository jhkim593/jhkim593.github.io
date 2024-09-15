---
layout: post
title: "Kafka - Producer"
author: "jhkim593"
tags: Kafka
---
# 프로듀서

카프카 브로커 특정 토픽에 데이터를 전송하는 역할을 합니다. 데이터를 전송시에는 반드시 **리더 파티션과 직접 통신**합니다.

내부 구조는 다음과 같습니다.

<img src="/assets/images/31/1.png"  width="700" height="300"/>

### ProducerRecord

- 프로듀서에서 전송할 메세지 객체입니다.
- 어떤 토픽 , 파티션에 보낼지 어떤 메세지 키 , 값을 보낼지에 대한 정보를 포함합니다.
- 오프셋은 값은 포함되어있지 않은데 프로듀서단에서 지정되지 않고 실제 브로커에 저장되는 시점에 지정되게 됩니다.

### Accumulator

카프카 브로커로 메세지를 보내기전 배치 전송할 데이터를 모으는 버퍼 영역입니다. 프로듀서의 send() 와 같은 메소드 호출시 브로커로 바로 전송되지 않고 일정 데이터를 쌓아 배치로 묶어 전송하게 됩니다.

### Partitioner

메세지를 토픽에 어느 파티션으로 전송할 지 결정하는 역할을 합니다.

파티셔너 종류는 다음과 같습니다.

- uniformStickyPartitioner : 파티셔너 지정하지 않을 경우 디폴트 설정
- RoundRobinPartitioner
- CustomPartitioner

**uniformStickyPartitioner , RoundRobinPartitioner 동작**

메세지 키가 있을 경우 : 두 파티션 모두 동일한 메세지키는 동일한 파티션에 전달됩니다. 만약 파티션 개수가 변경( 추가) 될 경우 메세지키 파티션 번호 매칭은 꺠지기 때문에 처음 토픽 생성시 충분한 파티션 개수로 설정하도록 유의해야합니다.

메세지키 없을 경우 : 두 파티셔너 모두 최대한 동일하게 메세지를 파티션에 분배하는데 uniformStickyPartitioner의 경우 고정된 파티셔닝으르로 배치 전송 효율이 더 높습니다.  예를들어 프로듀서가 키값이 null인 레코드0을 보내면 파티셔너는 키값이 null인 레코드를 확인하고 배치를 위해 임의로 파티션0에 레코드0을 저장합니다. 이후 키 값이null인 레코드1를 보내면 똑같은 파티션으로 레코드1를 저장합니다. 이러한 과정을 반복하면 전송 최소 레코드의 수를 충족해 Accumulator에서 더 효율적으로 배치 전송할 수 있게됩니다.

<br>
# 프로듀서 주요 옵션


### 필수 옵션

- bootstrap.servers : 브로커에 host:port를 설정 , 2개이상 입력해서 일부 브로커 이슈 발생해도 문제없도록하는것이 일반적임.
- key.serializer : 레코드의 메세지 키를 직렬화하는 클래스 지정
- value.serializer : 레코드 메시지 값을 직렬화하는 클래스 지정

### 선택 옵션

- acks  : 프로듀서가 전송한 데이터가 브로커들에 **정상적으로 저장되었는지 성공 여부를 확인**하는데 0 ,1 , all ( -1 ) 중 하나로 설정 (아래에서 다시 설명하겠습니다.)
- linger.ms : 배치 전송전 대기 시간
- retries : 브로커 에러받고난뒤 재전송 시도 횟수
- max.in.flight.requests.per.connection: 한번에 요청하는 최대 커넥션 개수
- partitioner.class : 레코드를 전송할때 적용할 파티셔너 클래스 지정
- enable.idempotence :멱등성 프로듀서로 동작할지 여부 설정 (멱등성 프로듀서에 대해서는 추후에 다루도록 하겠습니다.)
- transaction.id : 레코드 전송시 트랜잭션 단위로 묶을지 여부를 설정 (멱등성 프로듀서에 대해서는 추후에 다루도록 하겠습니다.)

<br>
## acks 옵션

프로듀서가 메세지를 보내고 그 메세지를 카프카가 잘 받았는지 확인할 것인지 , 확인하지 않을 것인지 결정하는 옵션입니다.

acks 옵션은 0 ,1 , all (-1) 로 설정 할 수 있습니다.

### **acks = 0**

**리더 파티션으로 데이터 전송했을 때 리더파티션으로 데이터가 저장되었는지 확인하지 않습니다**. 응답을 기다리지 않기 때문에 acks 옵션중 속도가 가장 빠릅니다.

메세지를 보내는 와중에 리더 파티션이 다운된다면 메세지가 유실될 수 있기 때문에 일부 데이터가 유실되도 괜찮다면 효율적인 방식입니다.

### **acks =1**

**리더 파티션에만 정상적으로 데이터가 저장되었는지 확인**하며 정상적으로 적재되지 않았다면 적재될때까지 재시도할 수 있습니다. 응답을 기다리기 때문에 ack=0 보다 속도가 낮습니다.

팔로워 파티션에 복제 완료된 것은 확인하지 않기 때문에 데이터 유실 가능성 존재합니다.

예를들어 프로듀서가 보낸 메세지를 리더 파티션이 받아 응답을 보낸 뒤 팔로워 파티션이 리더 파티션의 메세지를 복제하기전 리더 파티션이 다운된다면 메세지가 유실될 수 있습니다.

하지만 이런 케이스는 거의 발생하지 않기 때문에 일반적인 운영 환경에서 acks = 1을 사용해도 무방합니다.

### **acks = all ( -1 )**

카프카 클라이언트 프로듀서 3.0 이후부터 **acks가 all이 기본값**으로 설정됩니다.

**리더 팔로워 파티션 모두 메세지가 정상적으로 저장되었는지 확인**합니다. 팔로워 파티션 까지 확인하기 때문에 속도가 가장 느립니다.

acks = all (-1) 인 경우 모든 파티션을 확인하는 것은 아니고 **min.insync.replicas 수만큼 파티션을 확인**합니다.

예를들어 min.insync.replicas 가 2이면 리더 , 팔로워 1 , 팔로워 2 파티션이 있을 때 리더 ,팔로워1 파티션에 적재된 것만 확인합니다. 1로 설정하면 acks =1 옵션과 동일한 동작을 하기 때문에 2이상일 때 의미가 있습니다.

Kafka에서 replication.factor와 min.insync.replicas가 동일하게 설정되어 있다면, 팔로워 파티션 중 하나라도 문제가 발생할 경우 메시지 송수신에 문제가 생길 수 있습니다. 이 때문에 min.insync.replicas 값을 replication.factor보다 낮게 설정하는 것이 일반적입니다.

실제 운영 환경에서 브로커 2개가 동시에 다운되는 일은 거의 없기 때문에 min.insync.replicas를 2로 설정하더라도 메세지 손실을 충분히 방지할 수 있습니다.

<br>
# Kafka CLI를 통한 프로듀서 테스트

bin/Kafka-console-producer.sh 를 통해 간단한 프로듀서 테스트가 가능합니다.

```bash
bin/kafka-console-producer.sh  --bootstrap-server localhost:9092 --topic test
```

해당 커맨드로 특정 토픽에 메세지를 전송할 수 있습니다.


메세지키를 메세지와 같이 보내기 위해서는 아래 커맨드를 실행하면됩니다.
```bash
bin/kafka-console-producer.sh  --bootstrap-server localhost:9092 --topic test --property "parse.key=true" --property "key.separator=:"
```

property에 `parse.key` : true 와 `key.separator` 를 설정해 메세지키를 포함해서 전송할 수 있습니다. 위 예시에서는 separator가 : 이므로 key:value 형식으로 전송하게되면 메세지키를 포함시켜 전송하게됩니다.

<br>
# Kafka 프로듀서 어플리케이션 테스트

메세지 전송 결과를 기다리지 않고 콜백으로 처리하는 비동기 프로듀서 어플리케이션 코드를 작성해 테스트해보겠습니다.

**build.gradle**

```groovy
implementation 'org.apache.kafka:kafka-clients:3.8.0'
```

테스트에 kafka-clients:3.8.0 버전을 사용했습니다.

<br>
**ProducerCallback.class**

```java
public class ProducerCallback implements Callback {
    private final static Logger logger = LoggerFactory.getLogger(ProducerCallback.class);

    @Override
    public void onCompletion(RecordMetadata metadata, Exception e) {
        if (e != null)
            logger.error(e.getMessage(), e);
        else
            logger.info("topic : {} , partition: {} , offset: {}",metadata.topic(),metadata.partition(),metadata.offset());
    }
}
```

메세지 전송 완료시 콜백 함수 호출을 위해 Callback 인터페이스를 구현했습니다. 간단하게 로그만 찍도록 했습니다

<br>
**ProducerWithAsyncCallback.class**

```java
public class ProducerWithAsyncCallback {
    private final static String TOPIC_NAME = "test";
    private final static String BOOTSTRAP_SERVERS = "localhost:9092";

    public static void main(String[] args) {

				//프로듀서 옵션 설정
        Properties configs = new Properties();
        configs.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        configs.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        configs.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

        //설정된 옵션으로 프로듀서 객체 생성
        KafkaProducer<String, String> producer = new KafkaProducer<>(configs);

				//토픽 , 메세지 정보로 레코드 생성
        ProducerRecord<String, String> record = new ProducerRecord<>(TOPIC_NAME, "key", "value");
        //레코드와 콜백 객체로 send 메소드 호출
        producer.send(record, new ProducerCallback());

        producer.flush();
        producer.close();
    }
}
```

KafkaProducer 객체 생성 후 send 메소드 호출에 콜백 객체를 파라미터로 추가했습니다.

실행결과는 다음과 같습니다.

<img src="/assets/images/31/2.png"  width="850" height="27"/>

콜백 함수를 통해 **test 토픽 , 0 파티션에 , offset 6으로 저장**됐다는 것을 확인 할 수 있었습니다.

<br>
현재 acks 기본값은 all인데 여기서 0으로 바꾸게 된다면 어떤 결과가 나올지 보겠습니다.

```java
configs.put(ProducerConfig.ACKS_CONFIG, "0");
```

프로듀서 옵션에 acks = 0 을 설정했습니다. 실행 후 로그를 다음과 같이 찍힙니다.

<img src="/assets/images/31/3.png"  width="1050" height="25"/>

일반적으로 오프셋은 0부터 시작하기 때문에 로그에 찍힌 **offset -1은 기본적으로 설정될 수 없는 값**입니다. 그러나 acks=0 으로 설정된 경우, 프로듀서는 브로커로부터 **전송 결과에 대한 응답을 받지 않기 때문에** 프로듀서가 오프셋을 확인하지 못해 -1이 기록될 수 있습니다.
