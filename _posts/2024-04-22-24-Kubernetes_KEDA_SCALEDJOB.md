---
layout: post
title: "Kubernetes - RabbitMQ 지표를 통한 KEDA ScaledJob 적용"
author: "jhkim593"
tags: Kubernetes
---
앞선장에서는 KEDA ScaledObject를 사용한 HPA 적용에 대해서 다뤘었었습니다. 하지만 특정 상황에서 HPA를 통해 파드를 관리하기 여러운 부분이 있었습니다. 이번장에서는 HPA 적용시 있었던 **문제 상황을 KEDA ScaledJob을 통해 해결**한 내용에 대해 다뤄보겠습니다.  

<br>

# 문제 상황

현재 프로젝트에는 취약점 분석을 위한 분석 파드가 존재하고 이 분석 파드는 **종료 시간을 예측 할 수 없는 긴 시간 동안 분석을 수행**했습니다.

분석 파드는 HPA를 통해 관리되고 있었는데 ScaleDown시에 HPA가 파드를 **임의로 종료시키기 때문에 분석 중인 파드가 죽을때가 있었습니다.** 분석 중인 파드가 죽게되면 처음 부터 분석을 진행해야 했기때문에 분석 시간이 오래 걸리는 문제가 생기게되었습니다.  

HPA 설정을 통해 해결해보려고 했지만 분석 중인 파드를 감지해 ScaleDown시 제외하는 기능은 제공하지 않고 있었습니다.

<br>
# KEDA ScaledJob

ScaledObject와 마찬가지로 약 60개 이상의 이벤트 타입을 지원합니다. 이벤트를 감지해 Kubernetes Job이 생성되는데 생성된 Job은 실행 중일때 종료되지 않기 때문에 **작업을 수행 중인 파드의 동작 완료를 보장하며** 프로세스가 완료됐을때 Job이 종료되게 됩니다.

세부적인 설정에 관한 부분은 아래에서 다루도록 하겠습니다.

<br>
# KEDA ScaledJob 적용

Kubernetes Job은 **파드 작업 수행 완료**를 보장하기 때문에 문제 상황 해결에 적절하다고 판단했고 이제 테스트용 분석 서버 프로젝트를 생성해 ScaledJob을 어떻게 적용했는지 살펴보도록 하겠습니다. 아키텍처는 아래와 같습니다.

<img src="/assets/images/24/1.png"  width="600" height="400"/>

KEDA ScaledJob을 통해 RabbitMQ Queue에 처리 대기 중인 메세지가 있다면 추가 Job을 생성해 메세지를 읽어 분석을 진행하고
Job이 완료되면 완료된 파드는 종료되도록 해보겠습니다.

파드에서 동작할 테스트 **분석 서버** 프로젝트는 [링크](https://github.com/jhkim593/blog_code/tree/master/keda_scaledJob)를 참고해주세요.


<br>
## 분석 서버 예시

분석 서버는 Spring Boot를 기반으로 하며 RabbitMQ Queue에서 메세지를 읽어 분석을 수행합니다.

**테스트에 사용될 이미지는 현재 DockerHub에 jhkim593/scaled_job_test:9 로 업로드**된 상태입니다.

<br>
### MQListener.class

```java
@Component
@RequiredArgsConstructor
@Getter
@Setter
@Slf4j
public class MQListener {
    private static final String QUEUE_NAME = "a";
    private final RabbitTemplate rabbitTemplate;
    private Status status = Status.IDLE;

    public enum Status {
        ING, IDLE, COMPLETED
    }

    @Scheduled(fixedDelay = 5000)
    public void getMessage() throws InterruptedException {
        if(!status.equals(Status.IDLE)) return;
        Object message = rabbitTemplate.receiveAndConvert(QUEUE_NAME);
        if(message == null) return;
        log.info("job start !");
        this.setStatus(Status.ING);

        Thread.sleep(15000);
        this.setStatus(Status.COMPLETED);
    }
}

```

RabbitMQ에서 Queue에 메세지가 있다면 분석 요청 메세지를 읽어 분석을 실행하는 부분입니다.

Status는 **ING** , **IDLE** , **COMPLETED** 가 존재하는데 IDLE 상태일 때 분석 요청을 받을 수 있고 받은 뒤 ING 상태로 전환되면 분석 요청을 받을 수 없습니다. 이후 분석 종료시 **COMPLETED** 상태로 전환됩니다.

**COMPLETED 상태로 전환되게 되면 JOB이 종료**되게됩니다.

분석은 15초 쓰레드 Sleep으로 대체 했습니다.


<br>
### TestController.class

```java
@Controller
@RequiredArgsConstructor
@Slf4j
public class TestController {
    private final MQListener mqListener;
    @GetMapping("/get/status")
    public ResponseEntity status(){
        return new ResponseEntity(mqListener.getStatus().name(),HttpStatus.OK);
    }
}
```

/get/status url을 통해 분석 상태를 확인 할 수있도록했습니다. COMPLETED가 됐을 때 분석이 종료됩니다.

<br>
### application.yml

```yaml
spring:
  rabbitmq:
    host: ${MQ_HOST}
    port: 5672
    username: test
    password: test
    virtual-host: /
server:
  port: 8082
```

RabbitMQ 연결을 위한 설정을 추가했으며  `spirng.rabbitmq.host` 값을 ${MQ_HOST}로 할당 받도록 했습니다.

<br>
### Dockerfile

```Dockerfile
FROM openjdk:17-jdk-slim

#jar
ARG JAR_FILE=build/libs/scaledJob.jar
COPY ${JAR_FILE} scaledJob.jar

#curl
RUN apt-get update && apt-get install -y \
curl

#entrypoint
COPY entrypoint.sh /entrypoint.sh
RUN ["chmod", "+x", "/entrypoint.sh"]

ENTRYPOINT ["/entrypoint.sh"]
```

이미지 빌드를 위한 도커 파일 설정입니다. jdk17을 base로 하며 curl을 install 후 **entrypoint.sh** 스크립트를 실행합니다.

<br>
### entrypoint.sh

```bash
#!/bin/bash

java -jar scaledJob.jar &

while true; do
    status=$(curl -s http://$MY_POD_IP:8082/get/status)
    echo $status
    if [[ $status == "COMPLETED" ]]; then
        echo "Job is completed"
        break
    else
        echo "Job is $status"
        sleep 2
    fi
done
```

컨테이너에서 실행될 스크립트 정의 부분이며 jar 실행 후 `http://$MY_POD_IP:8082/get/status` 에 요청을 보내 분석 상태를 확인하고 상태가 **COMPLETED 일 때 Job을 종료**시킵니다.

여기서 **$MY_POD_IP**는 파드의 IP를 의미하며 파드 생성시 동적으로 할당될 것입니다.


<br>
## KEDA 설치

KEDA는 helm을 사용해서 배포했습니다.

```yaml
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace
```

<br>
## RabbitMQ 설치 및 설정

설치는 ubuntu 20.04.5에 진행했으며 다음 커맨드를 통해 rabbitmq-server를 설치했습니다.

```bash
sudo apt update
```

```bash
sudo apt install -y rabbitmq-server
```

```bash
sudo rabbitmq-plugins enable rabbitmq_management
```

rabbitmq-server 설치 후 rabbitmq_management plugin을 활성화 시켰습니다.

```bash
//user 생성
rabbitmqctl add_user test test
rabbitmqctl set_user_tags test administrator
rabbitmqctl set_permissions -p / test ".*" ".*" ".*"

//exchange 생성
rabbitmqadmin declare exchange name=exchange type=direct

//queue 생성
rabbitmqadmin declare queue name=a durable=true

//exchange queue 바인딩
rabbitmqadmin declare binding source="exchange" destination_type="queue" destination=a routing_key=akey
```

이후 test 유저와 exchange , queue를 생성했으며 exchage와 queue a를 akey로 바인딩했습니다.

<br>
## KEDA ScaledJob생성

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
 name: scaledjob
spec:
  jobTargetRef:
    parallelism: 1                            # [max number of desired pods](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/#controlling-parallelism)
    completions: 1                            # [desired number of successfully finished pods](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/#controlling-parallelism)
    activeDeadlineSeconds: 600                #  Specifies the duration in seconds relative to the startTime that the job may be active before the system tries to terminate it; value must be positive integer
    backoffLimit: 6                           # Specifies the number of retries before marking this job failed. Defaults to 6
    template:
      spec:
        containers:
        - name: test
          image: jhkim593/scaled_job_test:8
          env:
            - name: MQ_HOST
              value: "172.30.1.2"
            - name: MY_POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
  pollingInterval: 30                         # Optional. Default: 30 seconds
  successfulJobsHistoryLimit: 5               # Optional. Default: 100. How many completed jobs should be kept.
  failedJobsHistoryLimit: 5                   # Optional. Default: 100. How many failed jobs should be kept
  minReplicaCount: 2   
  maxReplicaCount: 10                        # Optional. Default: 100
  triggers:
  - type: rabbitmq
    metadata:
      host: "amqp://test:test@172.30.1.2:5672/"
      protocol: amqp
      queueName: a
      mode: QueueLength
      value: "2"
```

ScaledJob 설정은 다음과 같습니다.

- `spec.jobTargetRef.parallelism`  : 병렬 처리를 위해 동시 작업할 파드 수치
- `spec.jobTargetRef.completions` : default는 1이며 해당 수치만큼 성공할 때까지 Job이 실행
- `spec.jobTargetRef.activeDeadlineSeconds` : 해당 수치이후에 job을 강제로 종료
- `spec.jobTargetRef.backoffLimit` : Job을 실패로 표시하기전 재시도 횟수
- `spec.pollingInterval` : 각 트리거를 확인하는 시간을 의미 default는 30초
- `spec.successfulJobsHistoryLimit` : Job의 성공 내역 저장 제한 수치
- `spec.failedJobsHistoryLimit` : Job의 실패 내역 저장 제한 수치
- `spec.minReplicaCount`: 기본적으로 생성되는 최소 Job수를 의미
- `spec.maxReplicaCount` : 생성되는 최대 pod수를 의미

더 자세한 스팩은 아래 링크를 참고하시면 됩니다.
> https://keda.sh/docs/2.13/concepts/scaling-jobs/

<br>
`containers.env`에는 컨테이너 실행시 사용하기 위한 **MQ_HOST**, **MY_POD_IP**를 정의했습니다. **MQ_HOST**는 분석 서버 실행시에, **MY_POD_IP**는 entrypoint.sh 실행시에 사용됩니다.

<br>
# KEDA ScaledJob 적용 테스트

KEDA ScaledJob이 잘 적용되었는지 확인하기 위해 ScaledJob을 생성하고 RabbitMQ 에 messgae를 publish 해보겠습니다.

먼저 scaledJob을 생성해보겠습니다.

```bash
kubectl apply -f scaledJob.yaml
```

생성후 job , pod를 확인해보면 job이 `minReplicaCount`를 따라 2개 생성되고 pod 또한 2개 생성된 것을 확인 할 수 있습니다.

<img src="/assets/images/24/2.png"  width="600" height="250"/>

<br>
이제 Queue에 message를 하나 publish 해보겠습니다.

```bash
//queue a에 메세지 전송
rabbitmqadmin publish exchange=exchange routing_key=akey payload="test"
```

<img src="/assets/images/24/3.png"  width="600" height="300"/>

Queue에 메세지를 읽어 15초뒤에 job 하나가 완료되고`minReplicaCount` 에 맞게 새로운 job이 하나더 생기는 것을 확인 할 수있습니다.

<br>
message를 추가로 더 날려보면 Queue에 message가 쌓이는 만큼 동작 중인 파드가 늘어난 것을 확인 할 수 있습니다.

<img src="/assets/images/24/4.png"  width="500" height="250"/>

이후 Job이 모두 완료되면

<img src="/assets/images/24/5.png"  width="500" height="350"/>

`successfulJobsHistoryLimit` 만큼 히스토리가 남아있고 `minReplicaCount` 에 맞게 2개 Job은 실행되고 있는 것을 확인 할 수있습니다.

---

## Reference
<https://keda.sh/docs/2.13/concepts/scaling-jobs/>
