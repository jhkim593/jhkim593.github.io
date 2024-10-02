---
layout: post
title: "Kafka - Streams"
author: "jhkim593"
tags: Kafka
---
# Kafka Streams

카프카 스트림즈는 **토픽에 적재된 데이터를 실시간으로 변환해서 다른 토픽에 적재**하는 라이브러리이며 카프카가 공식 지원합니다. 특징은 다음과 같습니다.

#### 카프카와 호환
카프카가 버전업 될때마다 스트림즈 라이브러리 또한 버전업 되기 때문에 **호환성에 대한 문제가 발생하지 않습니다.**

<br>
#### Exactly-once 처리 보장
메시지가 중복 처리 없이 단 한 번만 처리되는 것을 보장합니다.

<br>
#### 스트림즈 DSL, 프로세서 API 제공
스트림 데이터 처리에 필요한 **대부분에 기능을 스트림즈 DSL에서 제공**하기 때문에 편리합니다. 만약 스트림즈 DSL에 없는 기능이 필요하다면 프로세서 API 사용해 직접 구현이 가능합니다.

<br>
# Kafka Streams 구조

스트림즈 어플리케이션은 내부적으로 1개 이상 스레드를 생성할 수 있으며 **스레드는 1개이상의 태스크 (task) 를 가집니다.** 여기서 태스크는 스트림즈 어플리케이션 데이터 처리 최소 단위를 의미합니다. 카프카 스트림즈 또한 컨슈머와 마찬가지로 스트림즈 프로세스 , 스레드를 늘려 처리량을 높일 수 있습니다.

실제 운영 환경에서는 장애 발생을 대비해 2개 이상의 서버를 구성해 스트림즈 어플리케이션을 운영하는 것이 일반적입니다.

<br>
## 토폴로지

<img src="/assets/images/35/1.png"  width="550" height="250"/>

먼저 토폴리지란 2개 이상의 노드들과 선으로 이루어진 집합을 뜻하는데 크게 링형 , 트리혀 , 성형으로 나눌 수 있습니다. **카프카 스트림즈에서 사용하는 토폴로지는 트리 형태와 유사합니다.**

<br>
카프카 스트림즈 구조를 도식화 해보면 다음과 같습니다.

<img src="/assets/images/35/2.png"  width="430" height="370"/>

스트림즈에서 토폴로지를 이루는 노드는 **데이터 처리를 위한 프로세서(Processor)를 의미**하며 선은 **다음 노드로 넘어가는 스트림을 의미**한다고 볼 수 있습니다.

각 프로세서의 역할은 다음과 같습니다.

<br>
#### 소스 프로세서
데이터 처리를 위해 최초로 선언되야하는 노드로 하나 이상의 토픽에서 데이터를 가져오는 역할을 합니다.
<br>
#### 스트림 프로세서
다른 프로세서가 반환한 데이터를 받아 변환한 후, 다음 프로세서에 전달하는 역할을 합니다.
<br>
#### 싱크 프로세서
데이터를 특정 카프카 토픽으로 저장하는 역할을 합니다.

<br>
# **KStream, KTable, GlobalKTable**

스트림즈 DSL을 사용해 애플리케이션을 구현하기 전에, 레코드 흐름을 추상화하는 개념인 KStream, KTable, GlobalKTable에 대해 알아보겠습니다. 이 개념들은 스트림즈 DSL에서만 사용됩니다.

<br>
## **KStream**

<img src="/assets/images/35/3.png"  width="380" height="370"/>

레코드의 흐름을 표현한 것으로 **메세지 키와 메세지 값으로 구성**되어 있습니다. 레코드 스트림안에 있는 모든 레코드들은 INSERT로 동작하며 삭제 , 업데이트 개념이 없습니다. 다시 말해 레코드의 키가 같다 하더라도 **같은 키를 가진 기존행을 대체할 수 없습니다.**

예를들어 다음 두개 레코드가 스트림으로 보내지고 있을 때

```java
("user", 1) --> ("user", 3)
```

스트림즈에서 user에 해당하는 값을 합친다고 하면 user에 대해 4를 반환하게 됩니다. 왜냐하면 두번째 레코드가 값은 키값을 가지지만 첫번째 레코드의 업데이트로 동작하지 않기 때문입니다.

<br>
## KTable

<img src="/assets/images/35/4.png"  width="380" height="370"/>

KStream과 다르게 메세지 키를 기준으로 묶어서 사용합니다. KStream은 토픽의 모든 레코드를 조회할 수 있지만 KTable은 **메세지키를 기준으로 가장 최신 레코드만 저장**합니다.


<br>
## 코파티셔닝

KStream과 KTable을 조인하려면 반드시 **코파티셔닝** 되어있어야 하는데 코파티셔닝이란 조인을 하는 서로 다른 토픽의 파티션 개수가 동일하게하며 파티셔닝 전략 또한  동일하게 맞추는 작업입니다.

코파티셔닝이 필요한 이유는, **조인이 메시지 키를 기반**으로 이루어지기 때문입니다. 코파티셔닝을 통해 동일한 메시지 키를 가진 데이터가 동일한 태스크에서 처리되도록 보장할 수 있습니다.

<br>
## GlobalKTable

<img src="/assets/images/35/5.png"  width="350" height="370"/>

KTable은 메시지 키를 기준으로 데이터를 묶어 사용하며, 1개의 파티션이 1개의 태스크에 할당됩니다. 반면, GlobalKTable은 **모든 파티션 데이터가 각 태스크에 할당되기 때문에 조인 시 코파티셔닝이 필요하지 않습니다**.

GlobalKTable은 리파티셔닝 (동일한 파티션 개수로 토픽을 새로 만듬 등) 과정없이 조인 할 수 있어 저장된 데이터가 많지 않다면 좋은 대안이 될 수 있지만 모든 데이터를 각 태스크가 가지기 때문에 저장된 데이터가 많다면 **애플리케이션 성능에 영향을 미칠 수 있어 주의가 필요합니다**.

<br>
# 스트림즈DSL 테스트

스트림즈DSL을 사용해 KStream , KTable , GlobalKTable 조인 어플리케이션 코드를 작성해 테스트를 진행해보겠습니다.

<br>
### 의존성 설정

```groovy
implementation 'org.apache.kafka:kafka-streams:3.8.0'
```

kafka-streams:3.8.0 기준으로 테스트를 진행했습니다.

<br>
### 실행 옵션 정리

**필수 옵션**

- bootstrap.servers : 브로커에 host:port를 설정 2개이상 입력해서 일부 브로커 이슈 발생해도 문제없도록하는것이 일반적입니다.
- application.id : 스트림즈 어플리케이션 구분을 위한 아이디입니다.

**선택 옵션**

- default.key.serde : 레코드의 메세지 키를 직렬화 , 역직렬화 하는 클래스를 지정합니다.
- default.valud.serve : 레코드의 메세지 값을 직렬화 역직렬화하는 클래스를 지정합니다.
- num.stream.thread : 스트림 프로세싱 실행시 실행될 스레드 개수를 지정하며 기본값은 1입니다.
- state.dir : 상태 기반 데이터 처리시 데이터를 저장할 디렉토리를 지정합니다.

<br>
## KStream 테스트

join 테스트를 하기전에 KStream을 통한 스트림 데이터 처리 테스트를 진행해보도록하겠습니다.
```java
public class StreamsFilter {

    private static String APPLICATION_NAME = "streams-filter-application";
    private static String BOOTSTRAP_SERVERS = "localhost:9092";
    private static String STREAM_LOG = "stream-test1";
    private static String STREAM_LOG_FILTER = "stream-test2";

    public static void main(String[] args) {

        //1. 스트림즈 옵션 설정
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, APPLICATION_NAME);
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        //2. 소스 프로세서
        StreamsBuilder builder = new StreamsBuilder();
        KStream<String, String> streamLog = builder.stream(STREAM_LOG);

        streamLog
                //3. 스트림 프로세서
                .mapValues(String::trim)
                .filter((key, value) -> value.length() > 5)
                .mapValues((String v1) -> v1 + " 처리완료 ")
                //4. 싱크 프로세서
                .to(STREAM_LOG_FILTER);

        KafkaStreams streams;
        streams = new KafkaStreams(builder.build(), props);
        streams.start();
    }
}
```

위 코드를 간단하게 설명하면

1. Properties 객체를 통해 스트림즈 옵션을 설정합니다.
2. **stream-test1** 토픽에서 데이터를 가져옵니다.
3. `map.values` 를 통해 각 레코드의 값을 변환하고 `fileter` 를 통해 키와 값을 기반으로 레코드를 필터링 합니다.
4. **stream-test2** 토픽으로 변환된 레코드를 전송합니다.

<br>
어플리케이션을 실행시킨 뒤 **stream-test1** 토픽에 아래 메세지를 전송해보록 하겠습니다.

```java
key : "k1"  value : "v1      "
key : "key1"  value : "value1"
```

전송 후 **stream-test2** 토픽에서 메세지를 가져와 로그를 찍어보면

<img src="/assets/images/35/6.png"  width="900" height="70"/>

첫번째 메세지는 value에 공백이 제거된 후 필터링 되기 때문에 **stream-test2**로 저장되지 않은 반면 두번째 메세지는 value에 처리완료 값이 추가돼 **stream-test2** 토픽으로 저장된 것을 확인 할수 있습니다.

<br>
## KStream , KTable join

**product-name** , **product-price** 토픽을 각각 KStream , KTable로 가져와 조인 후 **product-join** 토픽에 저장해보도록 하겠습니다. 먼저 아래 명령어를 통해 토픽을 생성합니다.

```bash
bin/kafka-topics.sh --create --bootstrap-server localhots:9092 --partitions 3 --topic product-name
bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --partitions 3 --topic product-price
bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --partitions 3 --topic product-join
```
코파티셔닝을 위해 파티션 개수를 3으로 동일하게 설정했습니다.

<br>
KStream ,KTable을 조인하기 위한 어플리케이션 코드입니다.

```java
public class KStreamJoinKTable {

    private static String APPLICATION_NAME = "product-join-application";
    private static String BOOTSTRAP_SERVERS = "localhost:9092";
    private static String PRODUCT_NAME = "product-name";
    private static String PRODUCT_PRICE = "product-price";
    private static String PRODUCT_JOIN = "product-join";

    public static void main(String[] args) {

        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, APPLICATION_NAME);
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        //1
        StreamsBuilder builder = new StreamsBuilder();
        KTable<String, String> productPriceTable= builder.table(PRODUCT_PRICE);
        KStream<String, String> productNameStream = builder.stream(PRODUCT_NAME);
        //2
        productNameStream
                .join(productPriceTable, (name, price) -> name + ": " + price)
                .to(PRODUCT_JOIN);

        KafkaStreams streams;
        streams = new KafkaStreams(builder.build(), props);
        streams.start();

    }
}
```

위 코드를 간단하게 설명하면

1. **product-name**, **product-price** 토픽에서 데이터를 가져옵니다.
2. `join` 을 통해 새로운 레코드 값으로 변환 후 **product-join** 토픽에 저장합니다.

<br>
어플리케이션을 실행 시키고 **product-name** , **product-price** 토픽에 아래와 같이 메세지를 전송해보겠습니다.

```java
//product-name
key : "0"  value : "chair"
key : "1"  value : "desk"

//product-price
key : "0"  value : "5000"
key : "1"  value : "7000"
```

전송 후 **product-join** 토픽에서 메세지를 가져와 로그를 찍어보면

<img src="/assets/images/35/7.png"  width="900" height="150"/>

키를 기준으로 새로운 메세지가 **product-join** 토픽에 저장된 것을 확인할 수 있습니다.

<br>
## KStream , GlobalKTable join

여기서는 **product-name** , **product-price** 각각 KStream ,GlobalKTable로 가져와 **product-join** 토픽에 저장하도록 해보겠습니다. 먼저 아래 명령어를 통해 토픽을 생성합니다.

```java
bin/kafka-topics.sh --create --bootstrap-server localhots:9092 --partitions 3 --topic product-name
bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --partitions 7 --topic product-price
bin/kafka-topics.sh --create --bootstrap-server localhost:9092 --partitions 3 --topic product-join
```

조인하고자 하는 토픽이 코파티셔닝 되어있지 않아도 되기 때문에 **product-price** 토픽의 파티션을 임의로 다르게 설정했습니다.

KStream ,GlobalKTable을 조인하기 위한 어플리케이션 코드입니다.

```java
public class KStreamJoinGlobalKTable {
    private static String APPLICATION_NAME = "product-join";
    private static String BOOTSTRAP_SERVERS = "localhost:9092";
    private static String PRODUCT_NAME = "product-name";
    private static String PRODUCT_PRICE = "product-price";
    private static String PRODUCT_JOIN = "product-join";

    public static void main(String[] args) {

        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, APPLICATION_NAME);
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, BOOTSTRAP_SERVERS);
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        //1
        StreamsBuilder builder = new StreamsBuilder();
        GlobalKTable<String, String> productPriceTable = builder.globalTable(PRODUCT_PRICE);
        KStream<String, String> productNameStream = builder.stream(PRODUCT_NAME);

        //2
        productNameStream.join(productPriceTable,
                        (nameKey, nameValue) -> "n"+nameKey,
                        (name, price) -> name + " :" + price)
                .to(PRODUCT_JOIN);

        KafkaStreams streams;
        streams = new KafkaStreams(builder.build(), props);
        streams.start();
    }
}
```

위 코드를 간단하게 설명하면

1. **product-name**, **product-price** 토픽에서 각각 KSream , GlobalKTable로 데이터를 가져옵니다.
2. `join` 을 통해 새로운 레코드 값으로 변환 후 **product-join** 토픽에 저장합니다. 여기서 `(nameKey, nameValue) -> "n"+nameKey` 를 통해 productNameStream에서 조인에 사용할 레코드키를 정의할 수 있습니다.

<br>
join 어플리케이션을 실행 시키고 **product-name** , **product-price** 토픽에 아래와 같이 메세지를 전송해보겠습니다.

```java
//product-name
key : "0"  value : "chair"
key : "1"  value : "desk"

//product-price
key : "n0"  value : "5000"
key : "m1"  value : "7000"
```
전송 후 **product-join** 토픽에서 메세지를 가져와 로그를 찍어보면
<img src="/assets/images/35/8.png"  width="900" height="70"/>
KStream에 `"n"+nameKey`로 조인해 메세지가 잘 생성된 것을 확인 할 수 있습니다.
