---
layout: post
title: "Kubernetes Affinity"
author: "jhkim593"
tags: Kubernetes
---
# Affinity

쿠버네티스에서 pod를 특정 노드에 스케줄링 하기 위한 방법으로 `nodeSelector`와 유사하지만 더 상세하게 제한을 설정 할 수 있습니다.

Affinity는 두가지 종류로 구분됩니다.

- Node Affinity
- Pod Affinity & antiAffinity

<br>
# Node Affinity

노드의 레이블을 기반으로 파드가 스케줄링 될 수 있도록 제한 할 수있습니다.

Node Affinity에는 두 종류가 있습니다.

- **requiredDuringSchedulingIgnoredDuringExecution**
- **preferredDuringSchedulingIgnoredDuringExecution**

<br>
## **requiredDuringSchedulingIgnoredDuringExecution**
pod가 실행되기 위한 필수 조건을 설정 할 수있습니다. 해당 조건이 만족되지 않으면 파드를 스케줄링 할 수 없습니다.

 **yaml 예시**

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
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: disktype
                operator: In
                values:
                - ssd
```

여기서는 disktype이 ssd인 레이블이 존재하는 node에만 pod가 실행 될 것입니다.

사용가능한 `operator` 는 다음과 같습니다.

- In : values 필드에 설정한 값 중 일치하는 것이 하나라도 있는지 확인
- NotIn : values 필드에 설정한 값 모두와 일치하지 않는지 확인
- Exists : key 필드에 설정한 값이 있는지 확인
- DoesNotExist : key 필드에 설정한 값이 없는지 확인
- Gt : values에는 값이 하나만 있어야하며 더 큰 값인지 확인
- Lt : values에는 값이 하나만 있어야하며 더 작은 값인지 확인


<br>
### 테스트
node에 레이블을 부여해서 테스트를 진행해보도록 하겠습니다

```bash
kubectl label nodes node2 disktype=ssd
```

label 커맨드를 통해 node2에 disktype=ssd 레이블을 지정했습니다.

배포 후 결과를 살펴보면

<img src="/assets/images/18/1.png"  width="800" height="100"/>

레이블이 disktype = ssd로 설정된 node2에만 pod가 스케줄링 되는 것을 확인할수 있습니다.

<br>

## preferredDuringSchedulingIgnoredDuringExecution
노드 선정을 위한 가중치인 weight를 명시 할 수있습니다. 1부터 100까지 설정 할 수있으며 matchExpressions과 일치한다면 weight 값을 더하고 합계가 가장 큰 노드를 선택합니다.

필수 조건은 아니기 때문에 조건에 해당되는 노드가 없더라도 스케줄러는 작동합니다.

<br>
**yaml 예시**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 6
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
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            preference:
              matchExpressions:
              - key: disktype
                operator: In
                values:
                - hdd
          - weight: 2
            preference:
              matchExpressions:
              - key: disktype
                operator: In
                values:
                - ssd
```

여기서는 레이블이 disktype = hdd 일 때 가중치 1 , disktype = ssd 일 때 가중치 2를 부여하고 있습니다.  
가중치가 높은 노드를 우선으로 pod가 스케줄링 되며 **해당되는 노드가 없더라도 스케줄링은 동작**합니다.

<br>
### 테스트
노드에 레이블을 부여해서 테스트를 진행해보겠습니다.

```bash
kubectl label nodes node2 disktype=ssd
kubectl label nodes node3 disktype=hdd
```
label 커맨드를 통해 node2 에 disktype=ssd , node3에 disktype=hdd 레이블을 부여했습니다.

<br>
배포 후 결과를 살펴보면

<img src="/assets/images/18/2.png"  width="800" height="100"/>

가중치가 더높은 node2에 더 많은 pod가 배치된 것을 확인 할 수있습니다.

<br>
# Pod affinity & **antiAffinity**

pod간 affinity와 antiAffinity를 사용해서 **각 노드에 이미 실행 중인 다른 파드의 레이블을 기반**으로 파드가 스케줄링 되도록 설정 할 수있습니다.

각 동작은 다음과 같습니다.

- **affinity** : 특정 레이블에 pod가 실행 중인 node에 스케줄링

- **antiAffinity** : 특정 레이블에 pod가 실행 중인 node를 피해서 스케줄링

pod간 affinity와 antiAffinity에는 많은 양의 프로세싱이 요구되므로 대규모 클러스터에서는 스케줄링 속도가 크게 느려 질 수있어 주의해야합니다.

<br>
Pod affinity & antiAffinity 에는 두 종류가 있습니다

- **requiredDuringSchedulingIgnoredDuringExecution**
- **preferredDuringSchedulingIgnoredDuringExecution**

<br>
## **requiredDuringSchedulingIgnoredDuringExecution**

이미 실행중인 pod 레이블을 기반으로 스케줄링 노드 선정을 위한 필수 조건을 설정합니다.

**yaml 예시**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx1
  template:
    metadata:
      labels:
        app: nginx1
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: security
                operator: In
                values:
                - s1
            topologyKey: kubernetes.io/hostname
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment2
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx2
  template:
    metadata:
      labels:
        app: nginx2
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: security
                operator: In
                values:
                - s1
            topologyKey: kubernetes.io/hostname
```

여기서는

**nginx-deployment1**으로 생성되는 pod는 label이 **security = s1**인 pod가 실행된 node에 스케줄링 하도록 하는 필수 조건을 설정했으며,

**nginx-deployment2**으로 생성되는 pod는 label이 **security = s1**인 pod가 실행된 node에는 스케줄링 되지않도록 필수 조건을 설정했습니다.

 operator 설정은 node affinity 설정과 동일하므로 생략하겠습니다.

<br>
### topologyKey

노드 레이블 key를 확인해서 affinity를 적용하는 설정이며 **podAffinity**에 해당하는 node가 여러개일때 추가 서브 조건으로 사용가능합니다.

스케줄링시 먼저 pod의 레이블을 기준으로 실행할 노드를 탐색하고 **topologyKey**를 사용해서 원하는 노드인지 확인합니다.

**즉 클러스터의 모든 노드는 topologyKey 매칭되는 적절한 레이블을 가지고 있어야하며** 없는 경우 의도하지 않는 동작이 발생 할 수있습니다.
예를들어 pod affinity에 조건과는 일치하지만 topologyKey를 레이블로 가지고 있지 않는 노드는 스케줄링 되지않습니다.

<br>
### 테스트
테스트를 위해 label이 security = s1인 nginx pod를 생성했습니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    security: s1
spec:
  containers:
  - image: nginx
    name: nginx
    ports:
    - containerPort: 80
```

<img src="/assets/images/18/3.png"  width="800" height="100"/>

레이블이 security=s1인 pod가 현재 node2에서 실행중이기 때문에

**nginx-deployment1**을 통해 생성된 pod가 모두 node2에 스케줄링 된 것을 확인 할 수있으며
**nginx-deployment2**을 통해 생성된 pod는 모두 node2를 피해 다른 노드로 스케줄링 된 것을 확인 할 수있습니다.


<br>
## preferredDuringSchedulingIgnoredDuringExecution

노드 선정을 위한 가중치인  weight를 명시 할 수있습니다.

필수 조건은 아니기 때문에 조건에 해당되는 노드가 없더라도 스케줄러는 작동합니다.

### yaml 예시

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx1
  template:
    metadata:
      labels:
        app: nginx1
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
      affinity:
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: security
                  operator: In
                  values:
                  - s1
              topologyKey: kubernetes.io/hostname
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment2
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx2
  template:
    metadata:
      labels:
        app: nginx2
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: security
                  operator: In
                  values:
                  - s1
              topologyKey: kubernetes.io/hostname
```

---

## Reference

https://kubernetes.io/ko/docs/concepts/scheduling-eviction/assign-pod-node/
