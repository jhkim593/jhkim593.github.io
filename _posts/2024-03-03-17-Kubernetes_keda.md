---
layout: post
title: "Kubernetes - KEDA를 활용한 HPA"
author: "jhkim593"
tags: Kubernetes
---
<br>
# KEDA ( Kubernetes Event-driven Autoscaling )

> KEDA는 여러 소스로 부터 이벤트를 받아 파드들을 오토 스케일링 하기 위한 목적으로 개발된 경량의 쿠버네티스 컴포넌트이다.
>

쿠버네티스 hpa는 보통 metric 서버를 설치해서 metric 서버가 제공하는 CPU , Memory 샤용량을 기반으로 pod 개수를 조절합니다.

CPU , Memory외에 외부 지표를 사용해서 오토스케일링을 적용하기 위해서는 번거로운 작업이 많이들어갑니다. (외부 지표를 promethus등의 metric 서버로 수집하고 hpa에서 외부 metric 서버 값을 참조하는 등 )

하지만 KEDA를 사용하면 기본 hpa 지표외 http request rate , queue size 등 여러 외부 지표를 사용한 오토스케일링을 쉽게 구현할 수 있습니다.

현재 KEDA는 연동 가능한 이벤트 타입을 60개 이상 지원하고 있습니다.

- cron
- ActiveMQ
- RabbitMQ
- AWS DynamoDB 등

<br>
# KEDA 동작 및 특징

<img src="/assets/images/17/1.png"  width="800" height="300"/>

위는 쿠버네티스 metric 서버를 이용한 hpa와 KEDA를 이용한 hpa에 대해서 나타낸 그림입니다.
위 그림에서 알 수 있듯이 KEDA는 hpa가 여러 이벤트를 참조하도록 확장해 동작하는 것이며 hpa를 완전 대체하는 것은 아닙니다.

KEDA 요청을위해 **ScaledObject** 리소스를 생성하면 유효성 검증을 거쳐서 유효한 리소스일 경우 hpa가 자동으로 생성되게 됩니다. 이 hpa는 쿠버네티스 metric 서버가 아닌 **keda-metrics-apiserver를** 참조하도록 설정됩니다.

타겟이 되는 deployment와 scaledObject는 **같은 네임 스페이스**에 있어야 합니다.

<br>

# KEDA를 활용한 HPA 적용

쿠버네티스 클러스터에 KEDA를 설치하고 여러 이벤트 소스 중 RabbitMQ queue size를 참조해 hpa를 적용해보도록 하겠습니다.

## 아키텍처

<img src="/assets/images/17/2.png"  width="800" height="400"/>

RabbitMQ 두개의 queue a , queue b 를 만들어서 각 큐에 일정 메세지가 들어가면 a deployment , b deployment에 pod가 오토스케일링 되도록 해보겠습니다.

<br>
## KEDA 설치


KEDA는 helm을 사용해서 클러스터 배포했습니다.

설치 커맨드는 다음과 같으며 keda namespace에 KEDA가 설치되게됩니다.

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace
```
<br>
## RabbitMQ 설치 및 설정

설치는 ubuntu 20.04.5에 진행했으며 rabbitmq server를 설치하는 커맨드는 다음과같습니다.

### RabbitMQ 서버 설치

```bash
sudo apt update
```

```bash
sudo apt install -y rabbitmq-server
```

```bash
sudo rabbitmq-plugins enable rabbitmq_management
```

### RabbitMQ 설정

```bash
//user 생성
rabbitmqctl add_user test test
rabbitmqctl set_user_tags test administrator
rabbitmqctl set_permissions -p / test ".*" ".*" ".*"

//exchange 생성
rabbitmqadmin declare exchange name=exchange type=direct

//queue 생성
rabbitmqadmin declare queue name=a durable=true
rabbitmqadmin declare queue name=b durable=true

//exchange queue 바인딩
rabbitmqadmin declare binding source="exchange" destination_type="queue" destination=a routing_key=akey
rabbitmqadmin declare binding source="exchange" destination_type="queue" destination=b routing_key=bkey
```

test 유저를 생성했으며 exchage와 queue a를 akey로 바인딩 , exchange와 queue b를 bkey로 바인딩했습니다.

<br>
## 타겟 Deployment 및 KEDA ScaledObject생성

deployment.yaml 예시

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: a-nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: a-nginx
  template:
    metadata:
      labels:
        app: a-nginx
    spec:
      containers:
      - name: a-nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: b-nginx-deployment
  labels:
    app: b-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: b-nginx
  template:
    metadata:
      labels:
        app: b-nginx
    spec:
      containers:
      - name: b-nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

```bash
kubectl apply -f deployment.yaml
```

hpa 타겟으로 사용하기위해 간단하게 a-nginx-deployment , b-nginx-deployment 를 생성했습니다.

<br>
scaledObject.yaml 예시

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: a-rabbitmq-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    name: a-nginx-deployment
  minReplicaCount: 0
  maxReplicaCount: 10
  pollingInterval: 30
  triggers:
  - type: rabbitmq
    metadata:
      host: "amqp://test:test@172.30.1.2:5672/"
      protocol: amqp
      queueName: a
      mode: QueueLength
      value: "3"
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: b-rabbitmq-scaledobject
  namespace: default
spec:
  scaleTargetRef:
    name: b-nginx-deployment
  minReplicaCount: 0
  maxReplicaCount: 10
  pollingInterval: 30
  triggers:
  - type: rabbitmq
    metadata:
      host: "amqp://test:test@172.30.1.2:5672/"
      protocol: amqp
      queueName: b
      mode: QueueLength
      value: "3"
```

```yaml
kubectl apply -f scaledObject.yaml
```

- `spec.scaleTargetRef.name`  : 타겟이되는 deployment 자원의 이름

- `spec.pollingInterval` :  설정된 트리거 소스를 확인하는 간격

- `spec.triggers.type` : 이벤트 타입을 설정

- `spec.triggers[0].metadata.mode` : 대기열에 있는 메시지 수 지표 참조를 위한 QueueLength와 대기열에 게시된 속도 지표 참조를 위한 MessageRate가 있음

- `spec.triggers[0].metadata.value` : 오토스케일링을 타겟이 되는 QueueLengh 값

<br>
a queue에 Length 지표 참조를 위한 a-rabbitmq-scaledobject,

b queue에 Length 지표 참조를 위한 b-rabbitmq-scaledobject를 각각 생성했습니다.

여기서 scaledObject가 문제없이 생성됐다면 hpa도 자동적으로 생성되는것을 확인 할 수있습니다.

<img src="/assets/images/17/3.png"  width="800" height="100"/>
<br>

생성된 hpa를 metric 설정을 살펴보면 keda metrics server가 제공하는 지표를 참조하도록 external로 정의된 것을 확인 할 수있습니다.

<img src="/assets/images/17/4.png"  width="650" height="400"/>

<br>
# KEDA HPA 적용 테스트

KEDA를 통한 hpa가 잘 적용되었는지 직접 RabbitMQ에 message를 publish 해서 확인해보도록 하겠습니다.

각 queueu에 메세지를 publish 하는 커맨드는 다음과 같습니다.

```bash
//queue a에 메세지 전송
rabbitmqadmin publish exchange=exchange routing_key=akey payload="test"
//queue b에 메세지 전송
rabbitmqadmin publish exchange=exchange routing_key=bkey payload="test"
```

위 커맨드를 통해 각 큐에 메세지를 임의로 publish 해보겠습니다.

각 큐에 쌓인 메세지 크기를 참조해서 pod가 증가한 것을 확인 할 수있습니다.

<img src="/assets/images/17/5.png"  width="800" height="100"/>

<img src="/assets/images/17/6.png"  width="800" height="300"/>
<br>
다음으로 큐에 메세지를 전부 삭제해서 pod 수를 줄여보도록 하겠습니다.

```bash
rabbitmqctl purge_queue a
```

위 커맨드를 통해 queue a에 저장된 메세지들을 전부 지우게되면

<img src="/assets/images/17/7.png"  width="800" height="100"/>

<img src="/assets/images/17/8.png"  width="800" height="100"/>

일정 시간이 지난 후에 replicas가 0으로 감소된것을 확인 할 수 있었습니다.

---

## Reference

https://keda.sh/
