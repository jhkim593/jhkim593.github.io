---
layout: post
title: "Kubernetes Taints , Tolerations"
author: "jhkim593"
tags: Kubernetes
---
# Taints , Tolerations

**taint**는 **노드**에 적용되며 임의의 파드가 할당되는 것을 방지합니다.

**toleration**는 **파드**에 적용되며 **toleration과 일치하는 taint**를 가진 노드에 스케줄링 되도록 합니다.

taint와 toleration은 항상 함께 동작하여 파드가 부적절한 노드에 스케줄 되지 않게 합니다.

<br>
쿠버네티스 환경에 파드를 동작시킬 때 **마스터 노드**에는 파드가 스케줄링 되지않는데 이유는 마스터 노드에는 기본적으로 **taint**가 설정되어있기 때문입니다.

```bash
[node1 ~]$ kubectl describe node node1  | grep -i taint
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
```

<br>
# Taints

테인트 설정은 `taint`커맨드를 통해 설정 할 수있습니다.

```bash
kubectl taint nodes node1 key1=value1:NoSchedule
```
node1 노드에 테인트를 설정한 것이며 설정값에는 key , value 및 effect (NoSchedule) 가 있으며

위와 같이 설정하면 node1은 테인트와 일치하는 톨러레이션이 있는 파드만 스케줄링 될 수있습니다.

테인트를 **제거**할 때는 다음과 같이 실행하면 됩니다.

```bash
kubectl taint nodes node1 key1=value1:NoSchedule-
```

<br>
## effect 설정

effect 는 **NoSchedule** , **PreferNoSchedule** ,  **NoExecute** 세가지가 있습니다.

- **NoSchedule** : taint와 toleration이 없으면 파드가 배치되지 않으며 기존에 동작 중인 파드는 종료되지 않는다.
- **PreferNoSchedule** : taint와 toleration이 없으면 파드가 배치되지않으나 클러스터 리소스가 부족하면 taint가 설정되어 있어도 파드가 배치 될 수있다. ( toleration이 필수는 아님 )
- **NoExecute** :  taint와 toleration이 없으면 파드가 배치되지 않으며 , 기존에 실행 중인 파드도 toleration이 없으면 종료된다.

<br>
# Tolerations

톨러레이션은 파드 spec에 정의 할 수 있습니다. 정의 예시는 다음과 같습니다.

```yaml
spec:
  tolerations:
  - key: "key1"
    operator: "Equal"
    value: "value1"
    effect: "NoSchedule"
  - key: "key2"
    operator: "Equal"
    value: "value2"
    effect: "NoExecute"
    tolerationSeconds: 3600
```

`spec.tolerations` 항목에 설정하며 **key** , **value** , **operator** , **effect**를 포함합니다

**key , value , effect**는 테인트 설정과 동일합니다.

<br>
## operator

지정하지않으면 기본값은 `Equal` 입니다.

- Equal : key에 해당하는 value가 같은지 확인
- Exists : key가 존재하는지 확인 ( value 설정 필요없음)

> operator는 몇가지 특별 케이스가 있습니다.
>
> 모든 taint 허용을 의미합니다.
>
>
> ```yaml
> tolerations:
> - operator: Exists
> ```
>
> key가 key1인 모든 effect taint를 허용합니다.
>
> ```yaml
> tolerations:
> - key: key1
>   operator: Exists
> ```
>

taint와 toleration이 일치하기 위해서는 **operator를 통한 key ,value** 및 **effect**가 모두 동일해야합니다.

<br>
## tolerationSeconds

toleration은 추가로 `tolerationSeconds` 속성을 지정할 수 있습니다.

**effect가 NoExecute인 테인트가 노드에 추가되면** 이미 동작중인 파드여도 테인트와 일치하지 않는다면 즉시 종료됩니다. 하지만 tolerationSeconds을 설정 하게된다면 해당 테인트와 **일치하는 파드**는 설정된 시간 후에 종료되게 되며 그전에 테인트가 제거된다면 파드는 종료 되지않습니다.

적용 예시는 아래 테스트를 참고하시면 됩니다.

<br>
# Taint , Toloeration 테스트

실제 테인트와 톨러레이션을 적용해서 케이스별로 어떻게 동작하는지 테스트를 진행해보겠습니다.

## Test 1

테스트를 위해 마스터노드 **node1** 과 워커노드 **node2** , **node3** 를 세팅했습니다.

```bash
kubectl taint nodes node2 key2=value2:NoExecute
kubectl taint nodes node3 key3=value3:NoSchedule
```

node2 노드에 key2=value2 이고effect가 NoExecute인 테인트를

node3 노드에 key3=value3 이고effect가 NoSchedule인 테인트를 설정했습니다.

pod 배포를 위한 deployment yaml을 살펴보겠습니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
      tolerations:
      - key: "key2"
        operator: "Equal"
        value: "value2"
        effect: "NoSchedule"
```

파드 toleration을 key2 = value2 , effect를 NoSchedule 로 설정해서 배포를 진행했습니다.

배포 후 결과를 확인해보면

<img src="/assets/images/19/1.png"  width="800" height="100"/>

node2 와 key , value 값이 일치하지만 **effect가 일치하지 않기 때문에** 스케줄링 되지않는 것을 확인 할 수있었습니다.

<br>
## Test 2

테스트를 위해 마스터노드 **node1**과 워커노드 **node2**를 세팅했습니다.

먼저 파드 실행을 위한 yaml을 살펴보겠습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx1
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: "key2"
    value: "value2"
    operator: "Equal"
    effect: "NoExecute"
    tolerationSeconds: 30
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx2
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: "key2"
    value: "value2"
    operator: "Equal"
    effect: "NoExecute"
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx3
spec:
  containers:
  - name: nginx
    image: nginx
  tolerations:
  - key: "key2"
    value: "value2"
    operator: "Equal"
    effect: "NoSchedule"
```

**첫번째 파드** ( nginx1 ) toleration을 key2 = value2 , effect를 NoExecute로 설정했으며 추가로 tolerationSeconds을 30초로 설정했습니다.

**두번째 파드** ( nginx2 )는 toleration을 key2 = value2 , effect를 NoExecute로 설정했습니다.

**세번째 파드** ( nginx3 )는 toleration을 key2 = value2 , effect를 NoSchedule로 설정했습니다.

<br>
yaml을 통해 파드를 실행시키면 워커 노드는 node2만 있기때문에 node2에서 모두 동작 중인 것을 확인할 수 있습니다.

<img src="/assets/images/19/2.png"  width="800" height="100"/>

<br>
이후에 node2에 key2 = value2 , effect를 NoExecute로 **테인트**를 설정하게되면

```bash
kubectl taint nodes node2 key2=value2:NoExecute
```

테인트와 일치하지 않는 파드 nginx3은 바로 종료된 것을 확인 할 수 있습니다.

<img src="/assets/images/19/3.png"  width="800" height="100"/>

<br>
**테인트가 일치하지만** effect가 NoExecute이고 **tolerationSeconds가 30초**로 설정된 nginx1는 30초 뒤에 종료된 것을 확인 할 수 있습니다.

<img src="/assets/images/19/4.png"  width="800" height="100"/>

<br>
만약 node2에 추가된 테인트의 **effect가 NoSchedule 였다면** 새로운 파드가 스케줄링되지 않는것일 뿐 기존 파드를 종료시키지는 않기 때문에 **세 파드 모두 종료되지 않습니다.**

---

## Reference
<https://kubernetes.io/ko/docs/concepts/scheduling-eviction/taint-and-toleration/>
