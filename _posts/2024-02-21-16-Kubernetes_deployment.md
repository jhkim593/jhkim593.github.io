---
layout: post
title: "Kubernetes Deployment"
author: "jhkim593"
tags: Kubernetes
---
## kubernetes Deployment

쿠버네티스 클러스터에서 파드는 컨터이너로 집합으로 이루어져있고 **레플리카셋 ( ReplicaSet )**은 파드 집합의 실행을 유지하는것을 보장하기위해 사용됩니다.

프로덕션 환경에서 개발된 서비스를 배포한뒤에 그래도 방치하는 일은 없을 것입니다. 버그가 터지거나 새로운 기능이 추가될때마다 서비스를 업데이트 하게될텐데 레플리카셋은 동작 파드의 수를 유지할 뿐 업데이트에 관한 기능은 제공하지 않습니다.

하지만 **디플로이먼트 ( deployment )**는 **레플리카셋**에 상위 개념으로 **RollingUpdate** 또는 변경 사항 **버전관리** 등 서비스 배포 및 관리를 더욱 편하게 할 수 있습니다.

<br>
### deployment.yaml 예시

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
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
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

`spec.template.metadata.labels`는 `spec.matchLabels`에 해당되어있어야합니다.

replicas는 3 , container image는 nginx:1.14.2로 설정되어있는것을 확인 할 수 있습니다.

<br>
### deploy

```bash
controlplane $ kubectl get deployments.apps                  
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           50s
```

### replicaSet

```bash
controlplane $ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-86dcfdf4c6   3         3         3       25s
```

### pod

```bash
controlplane $ kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-86dcfdf4c6-57g6z   1/1     Running   0          3m11s
nginx-deployment-86dcfdf4c6-bfbn2   1/1     Running   0          3m11s
nginx-deployment-86dcfdf4c6-mxfm4   1/1     Running   0          3m11s
```

네이밍이 deployment - rs - pod 로 된것을 확인 할 수있습니다.

만약 여기서 레플리카셋을 지워서 모든 파드를 지운다고 해도 디플로이먼트는 레플리카셋에 상위 오브젝트이기 때문에 새로운 레플리카셋을 만들어 파드를 재생성하게됩니다.

즉 **deployment**가 **replicaSet**을 만들고 **replicaSet**이 **pod**를 만듭니다.

<br>
## Deployment rollingUpdate

<img src="/assets/images/16/1.png"  width="800" height="400"/>

롤링 업데이트 배포 전략은 **무중단 배포 전략**으로 기존 버전의 pod를 순차적으로 제거 , 새 버전 pod를 순차적으로 생성해서 교체하는 방법입니다. 업데이트 시 이전 버전 pod와 새 버전 pod가 공존하기 때문에 다운 타임이 발생하지 않는다는 장점이 있습니다. Deployment에 따로 지정하지 않았을때 rollongUpdate가 기본으로 설정됩니다.


<br>
`set` 커맨드를 통해 디플로이먼트를 업데이트 할 수 있습니다.

```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1
```
<br>

다른 방법으로는 `edit` 커맨드를 통해 containers.image를 nginx:1.16.1로 교체해서 업데이트 할 수도 있습니다.
```bash
kubectl edit deployment/nginx-deployment
```
<br>

rollout status 커맨드를 통해 rollout 상태를 확인 할 수 있습니다.

```bash
kubectl rollout status deployment/nginx-deployment
```

```bash
controlplane $ kubectl rollout status deployment/nginx-deployment
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "nginx-deployment" successfully rolled out
```

업데이트 진행 상황 로그가 출력되며 최종적으로 success 된 것을 확인할 수 있습니다.

여기서 레플리카셋을 확인해보면 새로운 레플리카셋을 생성해서 3개 파드를 실행하고 기존 레플리카셋으로 동작중인 파드는 없는것을 확인 할 수있습니다.

```bash
controlplane $ k get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-58449dbcd5   0         0         0       15m
nginx-deployment-5f7ff877fb   3         3         3       4m8s
```
<br>
## Deployment RollingUpdate 추가 스팩 정의

롤링 업데이트시 maxSurge , maxUnavailable을 사용해서  프로세스를 제어 할 수 있습니다.

<br>
### maxSurge

업데이트시 생성할 수 있는 최대 파드의 수를 지정 할 수 있습니다.

기본값은 25%이며 예를들어 replica = 3 이면 25퍼인 0.75를 반올림한 3+1만큼 pod가 실행될수 있습니다.

maxUnavailable값이 0이면 0이 될 수없습니다. (업데이트가 불가능하기 때문)

### maxUnavailable

업데이트시 사용 할 수없는 최대 파드의 수를 지정 할 수 있습니다.

마찬가지로 기본값은 25%이며 maxSurge가 0이면 0이 될 수 없습니다.

<br>
### deployment.yaml 예시

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  progressDeadlineSeconds: 600
  revisionHistoryLimit: 10
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
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
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

<br>
## Deployment rollback

디플로이먼트에 **파드 템플릿**이 변경되는 경우에만 새로운 디플로이먼트 버전이 생성됩니다. 기존 버전 상태로 이동이 가능합니다.

`rollout history` 커맨드를 통해 디플로이먼트 변경 기록을 확인 할 수 있습니다.

```bash
controlplane $ k rollout history deployment nginx-deployment
deployment.apps/nginx-deployment
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```



`rollout undo` 커맨드를 통해서 바로 전 버전으로 롤백 할 수있습니다.

```bash
kubectl rollout undo deployment nginx-deployment
```

`--to-revision` 옵션을 통해서 특정 버전으로 롤백 할 수있습니다

```bash
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

```bash
controlplane $ kubectl rollout history deployment nginx-deployment
deployment.apps/nginx-deployment
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
```

rollback 하게되면 rollback 된 버전이 삭제되고 +1되어서 최신 상태로 변경됩니다.

<br>
## Deployment rollout resume / pause

`pause` 커맨드를 사용해서 롤아웃을 일시정지하게되면 변경 사항이 생겨도 롤아웃이되지 않습니다.

이를 통해 불필요한 롤아웃을 줄이고 여러 수정 사항을 적용 할 수 있습니다.

```bash
kubectl rollout pause deployment/nginx-deployment
```

리소스 및 이미지 변경을 위해 `set`  커맨드를 사용했지만

```bash
kubectl set resources deployment/nginx-deployment -c=nginx --limits=cpu=200m,memory=512Mi
kubectl set image deployment/nginx-deployment nginx=nginx:1.16.1
```

아래와 같이 레플리카셋은 추가로 생성되지 않은 것을 확인 할 수 있습니다.

```bash
controlplane $ k get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-86dcfdf4c6   3         3         3       8s
```

`rollout resume`  커맨드를 통해서 롤아웃을 재개하게되면

```bash
kubectl rollout resume deployment/nginx-deployment
```

아래와 같이 수정 사항이 한번에 적용되는 것을 확인 할 수 있습니다.

```bash
controlplane $ kubectl get rs
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-548c777b7b   3         3         3       15s
nginx-deployment-86dcfdf4c6   0         0         0       2m54s
```

```bash
controlplane $ k describe deployment nginx-deployment
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Sun, 18 Feb 2024 18:22:37 +0000
Labels:                 app=nginx
Annotations:            [deployment.kubernetes.io/revision:](http://deployment.kubernetes.io/revision:) 3
Selector:               app=nginx
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
Labels:  app=nginx
Containers:
nginx:
Image:      nginx:1.16.1
Port:       80/TCP
Host Port:  0/TCP
Limits:
cpu:        200m
memory:     512Mi
Environment:  <none>
Mounts:       <none>
Volumes:        <none>
...
```
---

## Reference

https://kubernetes.io/ko/docs/concepts/workloads/controllers/deployment/
