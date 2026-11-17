---
layout: post
title: "Kubernetes - RabbitMQ 지표를 통한 KEDA ScaledJob 적용"
author: "jhkim593"
tags: Kubernetes
---
앞선장에서는 KEDA ScaledObject를 사용한 HPA에 대해서 다뤘었었습니다. 하지만 특정 상황에서 HPA를 통해 파드를 관리하기 여러운 부분이 있었습니다. 이번장에서는 HPA 적용시 있었던 **문제 상황을 KEDA ScaledJob을 통해 해결**한 내용에 대해 다뤄보겠습니다.  

<br>

# 문제 상황

현재 프로젝트에는 취약점 분석을 위한 분석 파드가 존재하고 이 분석 파드는 **종료 시간을 예측 할 수 없는 긴 시간 동안 분석을 수행**했습니다.

HPA를 적용해봤지만 Scale In시에 HPA가 파드를 **임의로 종료시켜 분석 중인 파드가 종료될 때가 있었습니다.**   
분석 중인 파드가 죽게되면 처음 부터 분석을 진행해야 했기때문에 분석 시간이 오래 걸리는 문제가 발생했었습니다.


<br>
# KEDA ScaledJob이란?

Kubernetes Job은 **파드의 작업 완료를 보장**하는 리소스입니다.  
Deployment가 파드를 계속 실행 상태로 유지하는 것과 달리, Job으로 실행된 파드는 임의로 종료되지 않고, 컨테이너 프로세스가 정상 종료되어야 Job이 완료됩니다.  
이런 특성 때문에 한 번 실행되고 끝나는 작업에 주로 사용됩니다.

KEDA ScaledJob은 이벤트를 감지해 이 Job을 생성하는 리소스로, ScaledObject와 마찬가지로 RabbitMQ, Kafka 등 약 60개 이상의 이벤트 소스를 지원합니다.

ScaledObject가 HPA를 통해 파드 수를 늘리고 줄여 스케일링한다면, ScaledJob은 **처리할 이벤트 수만큼 Job을 생성**하는 방식입니다.  
생성된 Job은 작업이 끝나야 종료되기 때문에, HPA처럼 **작업 중인 파드가 중간에 종료되는 일이 없습니다.**

<br>
이러한 특성 때문에 ScaledJob이 문제 상황 해결에 적절하다고 판단했습니다.

<br>
# KEDA ScaledJob 적용

테스트용 프로젝트를 통해 ScaledJob 적용 과정을 살펴보겠습니다. 전체 아키텍처는 아래와 같습니다.

<img src="/assets/images/24/1.png"  width="650" height="400"/>

KEDA ScaledJob이 RabbitMQ Queue에 처리 대기 중인 메세지를 감지하면 Job을 생성하고, 생성된 Job의 파드가 Queue에서 메세지를 가져와 분석을 진행합니다.
분석이 끝나면 파드가 종료되며 Job도 완료됩니다.

파드에서 동작할 테스트 프로젝트는 [링크](https://github.com/jhkim593/blog_code/tree/master/keda_scaledJob)를 참고해주세요.


<br>
## 분석 파드 동작 흐름

Job은 파드가 종료되어야 완료되기 때문에, 분석 파드는 **분석이 끝난 시점에 스스로 종료**되도록 만들었습니다. 흐름은 다음과 같습니다.

```
메시지 수신 → 분석 진행(ING) → 분석 완료(COMPLETED) → entrypoint 루프 탈출 → 컨테이너 종료 → Job 완료
```

- 분석 파드는 IDLE 상태일 때만 Queue에서 메세지를 하나 가져오고, 가져오면 **ING** 상태가 되어 더 이상 메세지를 받지 않습니다.
- 분석이 끝나면 **COMPLETED** 상태가 됩니다. (분석은 15초 Sleep으로 대체했습니다.)
- entrypoint.sh는 `/get/status` API로 상태를 확인하다가 COMPLETED가 되면 종료되고 Job이 완료됩니다.

entrypoint.sh 코드는 아래와 같습니다.

```bash
#!/bin/bash

java -jar analysis.jar &

while true; do
    status=$(curl -s http://$MY_POD_IP:8082/get/status)
    if [[ $status == "COMPLETED" ]]; then
        echo "Job is completed"
        break
    fi
    sleep 2
done
```
`$MY_POD_IP`는 파드 생성시 Downward API로 주입되는 파드 IP입니다.

<br>

## KEDA 설치

KEDA는 helm을 사용해서 배포했습니다.

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace
```

<br>

## KEDA ScaledJob 생성

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: scaledjob
spec:
  jobTargetRef:
    parallelism: 1
    completions: 1
    activeDeadlineSeconds: 600
    backoffLimit: 6
    template:
      spec:
        containers:
        - name: test
          image: jhkim593/scaled_job_test:9
          env:
            - name: MQ_HOST
              value: "172.30.1.2"
            - name: MY_POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
  pollingInterval: 30
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 5
  minReplicaCount: 2
  maxReplicaCount: 40
  rollout:
    strategy: gradual
    propagationPolicy: foreground
  scalingStrategy:
    strategy: custom
    customScalingQueueLengthDeduction: 0
    customScalingRunningJobPercentage: "0"
  triggers:
  - type: rabbitmq
    metadata:
      host: "amqp://test:test@172.30.1.2:5672/"
      protocol: amqp
      queueName: a
      mode: QueueLength
      value: "1"
```

이 문제를 해결하는 데 중요한 설정은 다음과 같습니다.

- **`scalingStrategy`** : 실행 중인 Job 수와 관계없이 **Queue에 메세지가 쌓인 만큼 Job이 생성**되도록 custom 전략을 쓰고 `customScalingQueueLengthDeduction`, `customScalingRunningJobPercentage`를 0으로 설정했습니다.
- **`rollout.strategy: gradual`** : 이미지 등 ScaledJob 스펙이 변경되어도 **진행 중인 Job은 그대로 두고** 새로 생성되는 Job부터 변경 사항을 적용합니다.
- **`minReplicaCount`** : 메세지가 없어도 기본으로 유지할 최소 Job 수입니다. 여기서는 2개로 설정했습니다.
- **`maxReplicaCount`** : polling 주기 한 번에 새로 생성할 수 있는 최대 Job 수입니다. 여기서는 최대 40개로 설정했습니다.
- **`successfulJobsHistoryLimit`** : 완료된 Job을 몇 개까지 남겨둘지 정합니다. 여기서는 5개로 설정해 최근 완료된 Job 5개의 기록을 확인할 수 있습니다.

그 외 자세한 스펙은 [공식 문서](https://keda.sh/docs/2.13/concepts/scaling-jobs/)를 참고하시면 됩니다.

<br>

# KEDA ScaledJob 적용 테스트

## ScaledJob 생성

ScaledJob을 생성하면 `minReplicaCount`에 따라 Job과 파드가 2개 생성됩니다.

```bash
kubectl apply -f scaledJob.yaml
```

<img src="/assets/images/24/2.png"  width="600" height="250"/>

<br>

## 메세지 publish 테스트

Queue에 메세지를 하나 publish 해보겠습니다.

<img src="/assets/images/24/3.png"  width="600" height="270"/>

메세지를 받은 Job이 15초 뒤 완료되고, `minReplicaCount`에 맞게 새로운 Job이 다시 생성됩니다.

<br>
메세지를 여러 개 보내면 Queue에 쌓인 만큼 Job이 늘어납니다.

<img src="/assets/images/24/4.png"  width="500" height="250"/>

<br>

Job이 모두 완료되면 `successfulJobsHistoryLimit`만큼 히스토리가 남고, `minReplicaCount`에 맞게   
2개의 Job이 대기하고 있는 것을 확인할 수 있습니다.

<img src="/assets/images/24/5.png"  width="500" height="350"/>

---

## Reference
<https://keda.sh/docs/2.13/concepts/scaling-jobs/>