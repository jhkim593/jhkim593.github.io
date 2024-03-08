---
layout: post
title: "Kubernetes - HPA (Horizontal Pod Autoscaling)"
author: "jhkim593"
tags: Kubernetes

---
## **HPA (Horizontal Pod Autoscaling) 란**

Horizontal Pod Autoscaling은 평균 CPU 사용율 , 평균 메모리 사용률 또는 다른 커스텀 메스틱등과 같은 지표를 관측해 pod수를 유동적으로 더 배치하는 것을 의미합니다. 크기 조절이 불가능한 Daemonset과 같은 pod에는 적용되지 않습니다.

HPA를 위해 사용량 모니터링 하기위해서는 **Metrics Server**가 별도로 실행되어야합니다.

<br>

## HPA 적용


HPA 적용을 위해 먼저 Metrics-Server를 설치해야합니다.
<br>
<br>
### Metrics-Server

HPA(Horizontal Pod Autoscaler) 및 VPA(Vertical Pod Autoscaler)가 활용할 수 있도록 리소스 메트릭을 수집합니다. `kubectl top` 명령어를 사용해서 수집된 메트릭을 확인 할 수 있습니다.

> [Metrics-Server Repository](https://github.com/kubernetes-sigs/metrics-server)

<br>
kubelet-insecure-tls를 설정

```bash
kubectl edit deploy -n kube-system metrics-server
```

```yaml
spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls   << 추가
```
<br>

### deployment , service

HPA 적용을 위해 php-apache deployment를 php-apache service로 노출합니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  replicas: 3
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
  selector:
    run: php-apache
```

### HPA 생성

`autoscale` 커맨드를 통해 HorizontalPodAutoscaler를 생성합니다.

- auto scaling target cpu : 50%
- 최소 pod 1개  , 최대 pod 10개

```bash
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

hpa.yaml 파일로 명시해서 생성하는 방법입니다.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache2
spec:
  maxReplicas: 10
  metrics:
  - resource:
    type: Resource
      name: cpu
      target:
        averageUtilization: 50
        type: Utilization
  minReplicas: 1
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
```

CPU Utilization 50퍼센트는  **pod resources에 requests** 값을 기준으로 계산합니다.

<br>
`kubectl get hpa` 명령어를 통해 생성된 HPA를 확인 할 수있습니다.

```json
controlplane $ kubectl get hpa
NAME         REFERENCE               TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%   1         10        1          2m55s
```

<br>
## HPA 테스트

부하 증가에 따라 autoscailing이 동작하는지 확인하기 위해 서비스에 요청을 보내 보내보겠습니다.

```bash
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

다시 HPA를 확인해보면

```json
NAME         REFERENCE                     TARGET      MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache/scale   110% / 50%  1         10        7          3m
```

autoscale이 동작해서 replicas가 7로 증가한 것을 확인 할 수 있습니다.

<br>
### HPA 알고리즘

pod가 추가 생성되는 알고리즘은 다음과 같습니다.

```yaml
원하는 레플리카 수 = ceil[현재 레플리카 수 * ( 현재 메트릭 값 / 원하는 메트릭 값 )]
```

만약 여기 처럼 초기 레플리카수는 3이고 cpu request가 200m이라고한다면 총 requests는 600m가 됩니다.

부하가 걸려서 총 cpu사용량이 800m이 되었다고 한다면 ( (800/600)* 100 = 130%가 될 것이고 여기서 타겟 cpu 퍼센트가 50% 라고한다면 3 * (130% / 50% ) = 7.8을 **ceil (올림)** 해서 8로 증가하게될 것입니다.

사용량과 타겟 사용량은 HPA에 엮인 pod에 **평균을 기준**으로합니다.<br>
만약 3개에 pod가 실행중인 상황에서 특정 한 pod 하나에 200m에 부하를 준다고하면
( 200 / 600 ) * 100 = 33% 이므로 pod 개수가 늘어나지 않습니다.

<br>
부하 발생을 중단하고 시간이 지난 후 scale down도 정상 동작한것을 확인할 수 있습니다.

```json
NAME         REFERENCE                     TARGET       MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache/scale   0% / 50%     1         10        1          15m
```

---

## Reference

<https://kubernetes.io/ko/docs/tasks/run-application/horizontal-pod-autoscale/>
