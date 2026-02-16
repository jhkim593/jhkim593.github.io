---
layout: post
title: "Kubernetes - CPU limit 해제를 통한 성능 개선"
author: "jhkim593"
tags: Kubernetes
---
EKS 환경에서 취약점 분석을 수행하는 파드에 CPU limit을 설정해 운영하고 있었는데, 분석 시 노드에 여유 CPU가 있음에도 limit으로 인해 성능이 제한되는 문제가 있었습니다.

이번 글에서는 CPU limit 해제를 통해 분석 파드의 성능을 개선한 내용에 대해 다뤄보겠습니다.

<br>

## 문제 상황

기존에는 분석 파드가 리소스를 과도하게 사용하는 것을 방지하기 위해 **resource에 requests와 limit을 동일하게 설정**하고 있었습니다. 

<img src="/assets/images/53/5.png"  width="700" height="250"/>

위 그래프처럼 특정 분석에서 CPU를 최대치로 사용하는 것을 확인할 수 있었는데 limit으로 인해 노드의 여유 CPU가 있음에도 이를 사용하지 못해 분석 시간이 길어지는 문제가 있었습니다.

CPU는 Compressible Resource이기 때문에 메모리와 달리 과도하게 사용하더라도 파드가 종료되지 않습니다.

따라서 CPU limit을 여유롭게 설정하거나 해제하면 노드의 여유 CPU를 활용해 분석 성능을 개선할 수 있을 것이라 판단했습니다.

<br>

## 압축 가능 리소스 vs 압축 불가능 리소스

<br>
**압축 가능 리소스 (Compressible Resource) — CPU**

CPU는 대표적인 압축 가능 리소스입니다. CPU가 부족하면 throttling이 발생해 처리 속도가 느려지지만 프로세스가 종료되지는 않습니다.

**압축 불가능 리소스 (Incompressible Resource) — Memory**

메모리는 압축 불가능 리소스입니다. 메모리가 부족하면 CPU처럼 속도를 줄여 조절하는 것이 불가능하고, Out of Memory가 발생하면서 프로세스가 강제 종료됩니다.

<br>
이처럼 CPU는 메모리에 비해 리소스가 부족하더라도 프로세스 종료로 이어지지 않기 때문에 상대적으로 유연하게 관리할 수 있습니다.

<br>

## CPU requests의 동작 방식

파드 `spec.containers[].resources.requests.cpu`에 설정하며, 파드가 실행될 때 최소한으로 보장받는 CPU 양을 의미합니다. requests 동작 방식은 크게 **스케줄링**과 **CPU 할당** 두 가지 측면으로 나눌 수 있습니다.

<br>

#### 스케줄링

Kubernetes 스케줄러는 파드를 배치할 때 각 노드의 allocatable CPU에서 이미 할당된 requests 합을 뺀 나머지를 확인하고, 요청한 requests를 수용할 수 있는 노드에 파드를 배치합니다.

<br>

#### cpu.shares를 통한 CPU 할당

Kubernetes는 requests 값을 리눅스 CFS(Completely Fair Scheduler)의 `cpu.shares` 값으로 변환합니다.  

1000m이 1024 shares에 해당하며 변환식은 다음과 같습니다.
> `cpu.shares = (cpu requests / 1000) * 1024`

예를 들어 requests가 2000m이면 2048 shares, 500m이면 512 shares로 변환됩니다. requests를 설정하지 않으면 기본값인 2 shares가 할당됩니다.

CFS는 **CPU가 부족해 경합이 발생하면 이 shares 값 비율에 따라 CPU를 할당**합니다.

<br>

## CPU requests 동작 테스트

CFS가 requests 기반으로 CPU를 어떻게 분배하는지 확인하기 위해 `stress` 컨테이너를 활용해 테스트했습니다.

<br>

#### 테스트 1 — 모든 파드가 CPU를 최대로 사용하는 경우

requests를 2:1:1 비율(2000m : 1000m : 1000m)로 설정한 파드 세 개를 동시에 같은 노드에 할당해 
CPU 경합 시 requests 비율대로 분배되는지 확인했습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: stress-pod-a
spec:
  containers:
    - name: stress
      image: polinux/stress
      command: ["stress"]
      args: ["--cpu", "5", "--timeout", "120"]
      resources:
        requests:
          cpu: "2000m"
---
apiVersion: v1
kind: Pod
metadata:
  name: stress-pod-b
spec:
  containers:
    - name: stress
      image: polinux/stress
      command: ["stress"]
      args: ["--cpu", "5", "--timeout", "120"]
      resources:
        requests:
          cpu: "1000m"
---
apiVersion: v1
kind: Pod
metadata:
  name: stress-pod-c
spec:
  containers:
    - name: stress
      image: polinux/stress
      command: ["stress"]
      args: ["--cpu", "5", "--timeout", "120"]
      resources:
        requests:
          cpu: "1000m"
```
<img src="/assets/images/53/2.png"  width="700" height="130"/>


세 파드 모두 CPU를 최대치로 사용하려고 할 때, CFS 스케줄러가 requests 비율(2:1:1)에 따라 CPU를 분배하는 것을 확인할 수 있습니다.

<br>

#### 테스트 2 — 일부 파드만 CPU를 많이 사용하는 경우

pod-a는 CPU를 적게 사용하고 나머지 두 파드만 CPU를 최대로 사용하도록 했을 때 어떻게 분배 되는지 확인했습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: stress-pod-a
spec:
  containers:
    - name: stress
      image: polinux/stress
      command: ["stress"]
      args: ["--cpu", "1", "--timeout", "120"]
      resources:
        requests:
          cpu: "2000m"
---
apiVersion: v1
kind: Pod
metadata:
  name: stress-pod-b
spec:
  containers:
    - name: stress
      image: polinux/stress
      command: ["stress"]
      args: ["--cpu", "5", "--timeout", "120"]
      resources:
        requests:
          cpu: "1000m"
---
apiVersion: v1
kind: Pod
metadata:
  name: stress-pod-c
spec:
  containers:
    - name: stress
      image: polinux/stress
      command: ["stress"]
      args: ["--cpu", "5", "--timeout", "120"]
      resources:
        requests:
          cpu: "1000m"
```
<img src="/assets/images/53/3.png"  width="700" height="130"/>

pod-a에 할당된 CPU 외 나머지를 pod-b와 pod-c가 requests 비율(1:1)에 따라 나눠서 사용하는 것을 확인할 수 있습니다.

<br>

#### 테스트 3 — requests를 설정하지 않은 파드가 있는 경우

pod-a는 requests를 설정하지 않고 pod-b와 pod-c는 requests 1000m(1024 shares)를 설정한 상태에서 CPU 경합 발생시 어떻게 분배되는지 확인했습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: stress-pod-a
spec:
  containers:
    - name: stress
      image: polinux/stress
      command: ["stress"]
      args: ["--cpu", "1", "--timeout", "120"]
---
apiVersion: v1
kind: Pod
metadata:
  name: stress-pod-b
spec:
  containers:
    - name: stress
      image: polinux/stress
      command: ["stress"]
      args: ["--cpu", "5", "--timeout", "120"]
      resources:
        requests:
          cpu: "1000m"
---
apiVersion: v1
kind: Pod
metadata:
  name: stress-pod-c
spec:
  containers:
    - name: stress
      image: polinux/stress
      command: ["stress"]
      args: ["--cpu", "5", "--timeout", "120"]
      resources:
        requests:
          cpu: "1000m"
```
<img src="/assets/images/53/4.png"  width="700" height="130"/>

pod-a는 requests가 없어 기본 2 shares만 할당되므로, requests를 설정한 pod-b/c(각 1024 shares)에 비해 CPU를 거의 할당받지 못하는 것을 확인할 수 있습니다.

<br>

## CPU limit 해제 시 주의사항

- CPU limit을 해제하면 파드가 노드의 모든 CPU를 사용할 수 있기 때문에, **같은 노드의 모든 파드에 반드시 requests를 설정**해야 합니다. requests가 없으면 기본 2 shares만 할당되어 CPU 경합 시 CPU를 할당 받지 못합니다.
- 하나의 노드에 많은 파드가 배치되는 환경에서는 CPU limit 해제가 오히려 예측하기 어려운 성능 편차를 유발할 수 있습니다.
- CPU limit을 해제하면 QoS가 Guaranteed에서 Burstable로 변경되므로, 노드 리소스가 부족할 때 **eviction 대상 우선순위가 높아질 수 있습니다.**


<br>

## CPU limit 해제 적용

현재 운영 환경에서는 **하나의 노드에 분석 파드가 최대 2개**까지만 배치되도록 구성되어 있어 CPU 독점으로 인한 영향 범위가 제한적이었습니다.

또한 노드에 여유 CPU가 있음에도 limit으로 인해 CPU 사용이 제한되고 있었기 때문에 **여유 CPU를 활용하면 분석 성능을 개선**할 수 있다고 판단했습니다.

같은 노드에서 실행되는 **데몬셋 파드들에는 CPU requests를 설정해 경합 시에도 최소 CPU를 보장**받도록 했습니다.

<br>

#### limit 해제 적용 전
<img src="/assets/images/53/5.png"  width="700" height="250"/>
<img src="/assets/images/53/7.png"  width="780" height="250"/>

limit으로 인해 최대 CPU 1000m 까지만 사용 가능했고 분석 시간이 약 13분이 걸리는 것을 확인할 수 있었습니다.

<br>

#### limit 해제 적용 후
<img src="/assets/images/53/6.png"  width="700" height="250"/>
<img src="/assets/images/53/8.png"  width="750" height="250"/>
limit을 해제해 노드의 여유 CPU를 활용할 수 있게 되면서 2000m 이상으로 늘어났고
분석 시간이 약 13분에서 9분으로 30% 가량 개선된 것을 확인할 수 있었습니다.

