---
layout: post
title: "Kubernetes 구성 요소 및 동작 흐름"
author: "jhkim593"
tags: Kubernetes

---
쿠버네티스는 구성요소를 크게 분류하면 클러스터 전반을 관리하는 **컨트롤 플레인(Control Plane) 컴포넌트**와 컨트롤 플레인 컴포넌트에 요청을받아 각 노드에서 동작 하는 **노드(Node) 컴포넌트로** 나눌 수 있습니다**.**

이번장에서는 컨트롤 플레인 **컴포넌트**, **Node** 컴포넌트와 각 컴포넌트들이 쿠버네티스에서 어떻게 동작하는지 흐름에 대해 살펴보겠습니다.

<img src="/assets/images/15/1.png"  width="800" height="450"/>

<br>
## 컨트롤 플레인 ( Control Plane )Component

### API server

쿠버네티스 컨트롤 플레인의 프론트엔드로 요청을 유효한지 판별하고 처리합니다. kubectl 인터페이스등을 통해 외부에서 API server에 액세스 할수 있으며 , 내부 모듈의 요청 또한 처리합니다.

<br>
### etcd

etcd는 오직 API server와 통신하며 쿠버네티스 클러스터가 동작하기 위해서 클러스터 및 리소스의 구성 정보 , 상태 정보등을 키 값 쌍 형태로 저장하는 저장소입니다. 노드 추가 , pod 또는 deployment등 클러스터에 대한 모든 변경 사항이 etcd에 업데이트됩니다.

kubectl get 커맨드를 통해 보여지는 데이터는 모두 etcd에서 가져온것입니다.

<br>
### Scheduler

클러스터는 여러 노드로 구성되어있는데 파드는 여러 노드중 특정 노드에 배치되게 됩니다.  이때 파드를 어떤 노드로 배치할지 결정하는 작업을 **스케줄링**이라하고하는데

스케줄러는 노드 상태와 pod 요구 사항등을 체크해서 어떤 워커 노드에 pod를 실행시킬지 **스케줄링**을 담당합니다.

<br>
### Controller

다운된 노드는 없는지 , deployment가 replicas를 잘 유지하고 있는지 등 상태를 감지하고 API server를 통해 그 상태를 유지 관리합니다.

<br>
## 노드 ( Node ) Component

### kubelet

각 노드에서 실행되는 기본 Node Agent입니다. **API server**와 통신하며 podSpec에 기술된 컨테이너들이 동작하도록 합니다. pod를 어디에 배치할지는 Scheduler가 결정하지만 실제 Container runtime에 명령하는것은 kubelet입니다.

만약 워커 노드에서 동작중인 kubelet이 정지되면 pod 생성 , 삭제등 작업이 정상 수행되지 않습니다.

또한 kubelet은 주기적으로 노드와 파드의 상태를 마스터에 보고해서 마스터는 이를 통해 클러스터 전체 상태를 파악 할 수 있습니다.

<br>
### kube-proxy

kube-proxy는 모든 워커노드에 Daemonset 형태로 하나씩 동작하고 있으며 서비스를 만들었을 때 클러스터 ip나 노드 포트로 접근할 수 있도록합니다.

```json
controlplane $ k get daemonsets.apps -A
NAMESPACE     NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-system   kube-proxy   2         2         2       2            2           kubernetes.io/os=linux   6d
```
<br>
### container runtime

파드에 포함된 컨테이너 실행을 담당합니다. 컨테이너를 제어하기 위한 표준 규약인 컨테이너 런타임 인터페이스 (CRI)를 준수하는 어플리케이션을 의미하며 대표적으로 containerd , CRI-O가 있습니다.

<br>
## kubernetes 동작 흐름

쿠버네티스에서 새로운 ReplicaSet을 생성하기위한 흐름은 아래와 같습니다.

<img src="/assets/images/15/2.png"  width="700" height="550"/>

각 컴포넌트들은 서로 통신하지 않고 오직 API server와 통신하게됩니다.

- kubectl등을 통해 API server에 ReplicaSet 생성 명령을 전송
- API server는 etcd에 ReplicaSet을 저장
- controller는 API server를 통해 요청을 감시하고 있다가 ReplicaSet에 정의된 lable selector 조건을 만족하는 pod가 존재하는지 확인
- 해당하는 label이 없을 경우 API server를 통해 etcd에 pod 생성 요청을 저장
- scheduler는 pod 생성 요청이 감지되면 pod 실행 조건에 맞는 node를 체크 후 API server를 통해 etcd에 pod 할당 요청 저장
- 워커노드의 kubelet이 자신의 node에 할당되었지만 생성되지않은 pod를 감지하면 kubelet이 pod 생성 명령 전달
- pod 상태를 API server에 주기적으로 전달


---

## Reference

https://kubernetes.io/docs/concepts/overview/components/
