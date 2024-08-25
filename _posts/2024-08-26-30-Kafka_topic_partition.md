---
layout: post
title: "Kafka - Topic , Partition"
author: "jhkim593"
tags: Kafka
---
# Topic

프로듀서가 전송하고 컨슈머가 가져가는 **메세지를 보관하는 역할**을 합니다. 파일시스템으로 비유하면 폴더와 유사합니다.

여기서

- 프로듀서는 특정 토픽에 메시지를 publish 하는 역할을 합니다.
- 컨슈머는 특정 토픽에서 메시지를 subscribe하여 읽어들이는 역할을 합니다.

**각 토픽은 1개 이상의 파티션으로 구성**되는데 파티션을 통해 처리할 데이터 양을 늘릴 수 있고, 데이터를 종류별로 나눠 처리 할수도 있습니다.

<br>
# Partition
<img src="/assets/images/30/1.png"  width="900" height="250"/>

토픽 내부에서 데이터를 처리하는 단위입니다.

파티션에 있는 데이터는 컨슈머가 가져가도 **삭제되지 않습니다**. 한번 파티션 개수가 설정되며 늘릴 수는 있으나 **줄이는 것은 불가능**하기 때문에 파티션 늘릴 때 주의가 필요합니다.

FIFO 방식을 통해 들어온 순서대로 메세지가 저장되게되며 파티션 마다 고유 **오프셋값이 할당**됩니다. 여기서 오프셋은 메세지 위치를 나타내는데 추후에 컨슈머가 데이터를 어디까지 읽었는지 알기위해 사용합니다.

프로듀서는 파티션에 데이터 전송시 **메세지 키 값을 지정할 수 있는데 키를 지정**해서 전송하게 되면 **같은 키를 가진 메세지는 같은 파티션에 저장**되는데 이로인해 같은 키를 가진 메세지는 순서가 유지되는 것을 보장받을 수 있습니다.

만약 메세지 키를 지정하지 않으면 Round-Robin 방식으로 특정 파티션에 메세지를 저장합니다.

<br>
# Replication

<img src="/assets/images/30/2.png"  width="600" height="250"/>

파티션의 복제본을 생성해서 특정 브로커에 장애가 발생하더라도 **데이터 유실을 방지** 할 수있습니다. 토픽 단위로 설정가능하며 기본값은 1입니다.

Replication 설정을 통해 리더 , 팔로워 파티션이 생성되는데 **리더 파티션**은 프로듀서, 컨슈머와 실제로 통신하는 역할을 하고 **팔로워 파티션**은 복제한 데이터를 저장하는 역할을 합니다.

`--replication.factor`를 통해 복제본 개수를 지정 할 수 있으며 현재 운용중인 브로커 개수만큼 최대값을 설정할 수 있습니다. 복제로 인해 저장 용량이 증가하지만 안정성을 위해 2개이상 복제본수를 설정하는 것이 일반적입니다.
예시로 `--replication.factor`가 3이라면 리더 파티션 1개 , 팔로워 파티션이 2개 생성되게 됩니다.

<br>
만약 브로커에 장애가 발생해 리더 파티션이 속한 브로커가 다운되면 **팔로워 파티션중 하나가 리더 파티션으로 승급**해서 컨슈머 , 프로듀서와 데이터를 주고받아 서비스 연속성을 유지 할 수 있습니다.

이 때 일시적으로 리더 , 팔로워 파티션간 동기화가 완벽히 이루어지지 않아 데이터 일관성이 깨질 수가 있기 때문에 기본적으로 리더로 승급되는 팔로워 파티션 선정시 **ISR**에 해당하는지 확인합니다. 여기서 ISR은 리더 파티션에 **모든 데이터를 동기화한 상태**를 의미합니다.

해당 옵션은 `unclean.leader.election.enable` 인데 기본값이 `false`이기 때문에 ISR에 해당하는 팔로워 파티션이 없다면 해당 브로커가 복구될 때까지 **데이터 처리가 중단**됩니다. 데이터 유실이 절대 발생하면 안될 때 사용하게됩니다.

만약 일부 데이터가 유실이 되도 괜찮다면 `unclean.leader.election.enable`  를 `true`로 설정해 ISR에 해당하지 않는 팔로워 파티션도 리더 파티션으로 승급할 수 있습니다.

<br>
**생성 예시**

토픽 생성시 파티션 수와 복제 개수를 지정할 수 있는데 각 의미가 혼동될 것 같아 생성 예시를 들어보겠습니다

만약 브로커3개, 파티션2개, 복제 개수를 3으로 B토픽을 생성하면 다음과 같이 셋팅될 수 있습니다. 위 그림을 참고해주세요

- **B토픽**
    - 브로커0 : B토픽(파티션0-팔로워), B토픽(파티션1-팔로워)
    - 브로커1 :  B토픽(파티션0-팔로워), B토픽(파티션1-리더)
    - 브로커2 : B토픽(파티션0-리더) , B토픽(파티션1-팔로워)

토픽 생성시 파티션 수는 **리더 파티션의 수를 의미합니다.** 위 예시에서 파티션 0 - 리더  , 파티션 1 -리더가 생성됐습니다.

복제본인 **팔로워 파티션은 리더 파티션이 속한 브로커와 다른곳에 생성**되게 됩니다. 위 예시에서 파티션 0 - 리더가 브로커2가 생성됐으므로 브로커2가 아닌 브로커 0과 브로커1에 파티션 0 - 팔로워 가 생성됐습니다.

<br>
# Kafka CLI를 통한 테스트

Kafka 다운로드 후 bin 경로에 CLI를 사용해 테스트가 가능합니다. Kafka 바이너리는 아래 경로에서 다운로드 가능하며 3.8.0 버전으로 테스트를 진행했습니다.

> https://kafka.apache.org/downloads
>

토픽 테스트를 위해서 kafka-topics.sh를 사용합니다.

<br>
### 토픽 생성

```bash
 bin/kafka-topics.sh \
--bootstrap-server localhost:9092 \
--replication-factor 1 \
--partitions 2 \
--config retention.ms=172800000 \
--topic topic-test \
--create
```

- `--bootstrap-server` : 토픽을 생성할 브로커들의 IP와 port를 입력합니다.
- `--replication-factor`  : 파티션을 복제 개수를 지정합니다. 만약 2이면 1개의 복제본을 생성하게됩니다.
- `--partitions` : 파티션의 개수를 지정합니다.
- `--config`  : 추가적인 설정을 할 수 있습니다. 여기서 retention.ms는 데이터 유지기간을 의미합니다.
- `--create`  : Topic을 생성할 때 사용하는 옵션입니다.

<br>
### 토픽 상세 정보 조회

`--describe` 옵션을 통해 토픽 상제 정보를 조회할 수 있습니다.

```bash
bin/kafka-topics.sh  --bootstrap-server localhost:9092 --topic topic-test --describe
```

```bash
Topic: topic-test       TopicId: 9-m7hT6VQjOBZvLFK7sXVg PartitionCount: 2       ReplicationFactor: 1    Configs: retention.ms=172800000
        Topic: topic-test       Partition: 0    Leader: 0       Replicas: 0     Isr: 0  Elr: N/A        LastKnownElr: N/A
        Topic: topic-test       Partition: 1    Leader: 0       Replicas: 0     Isr: 0  Elr: N/A        LastKnownElr: N/A

```

<br>
### 토픽 설정 변경

 `--alter` 옵션을 통해 토픽 설정을 변경할 수있습니다.

```bash
bin/kafka-topics.sh --bootstrap-server localhost:9092 --topic topic-test --alter --partitions 10
```

해당 커맨드는 토픽 파티션 개수를 10으로 변경할 때 사용합니다.
