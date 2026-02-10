---
layout: post
title: "Kubernetes CPU Limit 해제를 통한 워커 파드 성능 최적화"
author: "jhkim593"
tags: Kubernetes
hidden : true
---

현재 EKS 환경에서 워커 파드에 CPU limit을 설정해 운영하고 있는데, 분석 작업 시 노드에 여유 CPU가 있음에도 limit으로 인해 성능이 제한되는 문제가 있었습니다.

이번 글에서는 CPU limit 해제를 통해 워커 파드의 성능을 개선한 내용에 대해 다뤄보겠습니다.

<br>

## 문제 상황

워커 파드는 취약점 분석 작업을 수행하고 있으며, CPU 사용량이 높을수록 분석이 빠르게 완료됩니다.

하지만 CPU limit이 설정되어 있으면 노드에 여유 CPU가 남아 있어도 limit 이상은 사용할 수 없기 때문에, 불필요한 병목이 발생하고 있었습니다.

<br>

## 압축 가능 리소스 vs 압축 불가능 리소스

Kubernetes 리소스는 부족할 때의 동작 방식에 따라 두 가지로 나뉩니다.

**압축 가능 리소스 (Compressible Resource) — CPU**
- CPU가 부족하면 throttling이 발생해 속도가 느려질 뿐, 프로세스가 종료되지 않음
- 따라서 limit을 해제해도 다른 파드에 치명적인 영향을 주지 않음

**압축 불가능 리소스 (Incompressible Resource) — Memory**
- 메모리가 부족하면 throttling이 불가능하고, OOM Kill로 프로세스가 강제 종료됨
- 따라서 메모리는 limit 설정이 필요함

CPU는 압축 가능 리소스이기 때문에 limit을 해제하더라도 안전하게 운영할 수 있습니다.

<br>

## CPU Request의 동작 방식

CPU limit을 해제하면 파드가 노드의 여유 CPU를 자유롭게 사용할 수 있게 됩니다. 이때 CPU 분배를 담당하는 것이 리눅스의 CFS(Completely Fair Scheduler)이며, request 값을 기반으로 동작합니다.

`request`는 파드가 실행될 때 최소한으로 보장받는 CPU 양입니다. CFS는 이 request 값을 가중치로 사용해 CPU를 분배합니다.

**여유 CPU가 있는 경우**
- 다른 파드가 CPU를 사용하지 않으면 request 이상의 CPU를 자유롭게 사용할 수 있음

**CPU 경합이 발생하는 경우**
- 모든 컨테이너가 CPU를 최대치로 사용하면 CFS가 **request 비율에 따라 공정하게 분배**
- 예를 들어 노드에 4 Core가 있고, 파드 A의 request가 1000m, 파드 B의 request가 500m인 경우 A는 약 2.67 Core, B는 약 1.33 Core를 사용하게 됨

따라서 CPU limit을 해제하더라도 모든 파드에 request를 설정하면 최소 CPU는 보장되고, 여유분은 유동적으로 나눠 쓰는 구조가 됩니다.

<br>

## CPU Request 동작 테스트

실제로 CFS가 request 기반으로 CPU를 분배하는지 확인하기 위해 `stress` 컨테이너를 활용해 테스트했습니다.

<br>

#### 테스트 1 — 모든 파드가 CPU를 최대로 사용하는 경우

request를 2:1:1 비율(500m : 250m : 250m)로 설정한 파드 세 개를 동시에 실행해, CPU 경합 시 request 비율대로 분배되는지 확인합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: stress-pod-a
spec:
  containers:
    - name: stress
      image: progrium/stress
      command: ["stress"]
      args: ["--cpu", "4", "--timeout", "120"]
      resources:
        requests:
          cpu: "500m"
---
apiVersion: v1
kind: Pod
metadata:
  name: stress-pod-b
spec:
  containers:
    - name: stress
      image: progrium/stress
      command: ["stress"]
      args: ["--cpu", "4", "--timeout", "120"]
      resources:
        requests:
          cpu: "250m"
---
apiVersion: v1
kind: Pod
metadata:
  name: stress-pod-c
spec:
  containers:
    - name: stress
      image: progrium/stress
      command: ["stress"]
      args: ["--cpu", "4", "--timeout", "120"]
      resources:
        requests:
          cpu: "250m"
```

세 파드 모두 CPU를 최대치로 사용하려고 할 때, CFS 스케줄러가 request 비율(2:1:1)에 따라 CPU를 분배하는 것을 확인할 수 있습니다.

<br>

#### 테스트 2 — 일부 파드만 CPU를 많이 사용하는 경우

테스트 1과 동일한 request 설정에서, pod-a는 CPU를 적게 사용하고 나머지 두 파드만 CPU를 최대로 사용합니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: stress-pod-a
spec:
  nodeName: node01
  containers:
    - name: stress
      image: progrium/stress
      command: ["stress"]
      args: ["--cpu", "200m", "--timeout", "120"]
      resources:
        requests:
          cpu: "500m"
---
apiVersion: v1
kind: Pod
metadata:
  name: stress-pod-b
spec:
  nodeName: node01
  containers:
    - name: stress
      image: progrium/stress
      command: ["stress"]
      args: ["--cpu", "4", "--timeout", "120"]
      resources:
        requests:
          cpu: "250m"
---
apiVersion: v1
kind: Pod
metadata:
  name: stress-pod-c
spec:
  nodeName: node01
  containers:
    - name: stress
      image: progrium/stress
      command: ["stress"]
      args: ["--cpu", "4", "--timeout", "120"]
      resources:
        requests:
          cpu: "250m"
```

pod-a가 CPU를 적게 사용하면 여유 CPU가 발생하고, pod-b와 pod-c가 request 비율(1:1)에 따라 여유분을 나눠서 사용하는 것을 확인할 수 있습니다.

<br>

## 정리

- CPU는 압축 가능 리소스이기 때문에 limit을 해제해도 프로세스가 종료되지 않고 안전하게 운영할 수 있습니다.
- CPU limit을 해제하면 노드에 여유 CPU가 있을 때 워커 파드가 더 많은 CPU를 활용할 수 있어 분석 성능이 개선됩니다.
- limit 해제 시 모든 파드에 request를 설정하면 CFS 스케줄러가 request 비율대로 CPU를 분배하므로 특정 파드의 CPU 독점을 방지할 수 있습니다.

---

## Reference

<https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/>

<https://blog.outsider.ne.kr/1653>
