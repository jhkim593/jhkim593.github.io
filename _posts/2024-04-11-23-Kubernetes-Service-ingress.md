---
layout: post
title: "Kubernetes Service 와 Ingress"
author: "jhkim593"
tags: Kubernetes
---
이번장에서는 Kubernetes Service와 Ingress에 대해서 살펴보도록 하겠습니다.

<br>
# kubernetes Service

쿠버네티스 Service는 파드에서 통해 실행되는 어플리케이션들을 **외부 네트워크에 노출시키기위한 컴포넌트**입니다.

쿠버네티스 환경에서 동작하는 파드는 비영구적인 일회성 리소스입니다. 새로 생성될때마다 새로운 IP 할당 받게 되므로 통신을 유지하기 어렵기 때문에 쿠버네티스 Service는 **고정 IP**를 통해서 파드로 접근할 수 있게 합니다.

또한 서비스는 deployment와 같은 파드 집합에서 단일 네트워크 진입점을 부여해 여러 파드에 대한 **요청을 분산하는 로드밸런서 기능**을 수행합니다.

쿠버네티스 **Service 유형은 크게 4가지**로 분류됩니다.

1. ClusterIp
2. NodePort
3. LoadBalancer
4. ExternalName

아래에서는 ClusterIP, NodePort, LoadBalancer만 살펴보도록 하겠습니다.

<br>
## ClusterIP

<img src="/assets/images/23/1.png"  width="500" height="500"/>
클러스터 내부에서 Service에 요청을 보낼 때, Service가 관리하는 Pod들에게 **로드밸런싱**하는 역할을 하며 **오직 클러스터 내부에서만 접근 가능**합니다. Service 타입을 명시하지 않았을 때 **기본값**입니다.

<br>
### clusterIp.yaml 예시

```yaml
apiVersion: v1
kind: Service
metadata:
  name: clusterip-service
spec:
  type: ClusterIP
  clusterIP: 10.100.100.100
  selector:
    app: webui
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

`selector`에 label이 동일한 파드들을 그룹으로 묶어 단일진입점이 생성됩니다.

`port`는 클러스터 내부에서 사용할 Service의 포트를 의미하며

`targetPort` 는 Service 객체로 전달된 요청을 Pod로 전달할 때 사용하는 포트를 의미합니다.

<br>
Service를 생성 후 확인해보면

```yaml
[node1 ~]$ kubectl describe service clusterip-service
Name:              clusterip-service
Namespace:         default
Labels:            <none>
Annotations:       <none>
Selector:          app=webui
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.100.100.100
IPs:               10.100.100.100
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.5.1.2:80,10.5.1.3:80,10.5.1.4:80 + 2 more...
Session Affinity:  None
Events:            <none>
```

Endpoints 항목에 label 이름이 일치하는 pod들이 묶이게됩니다.

<br>
### NodePort

<img src="/assets/images/23/2.png"  width="500" height="500"/>

각 노드에 외부 접속 가능한 **포트를 노출**시킵니다. 포트는 생략하면 30000 ~ 32767 대역의 포트를 랜덤하게 사용하게됩니다.
각 노드에 동일한 포트를 노출해 트래픽을 받고 ClusterIP를 통해 다시 파드로 분산되는 방식이기 때문에 ClusterIp가 자동으로 생성되게 됩니다.

<br>
### nodePort.yaml 예시

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nodeport-service
spec:
  type: NodePort
  clusterIP: 10.100.100.200
  selector:
    app: webui
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30200    
```

`spec.ports` 에 추가로 `nodePort` 를 설정해 외부에서 노드의 포트를 통해 서비스로 접근할 수 있습니다. 마스터 노드를 포함해 모든 노드에 대해 포트가 노출되기 때문에 **어떤 노드에서든 접근이 가능**합니다.

<br>
직접 nodePort에 해당하는 요청을 날리게되면

```bash
k get nodes -o wide
NAME           STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
controlplane   Ready    control-plane   38d   v1.29.0   172.30.1.2    <none>        Ubuntu 20.04.5 LTS   5.4.0-131-generic   containerd://1.7.13
node01         Ready    <none>          37d   v1.29.0   172.30.2.2    <none>        Ubuntu 20.04.5 LTS   5.4.0-131-generic   containerd://1.7.13
```

```bash
controlplane $ curl 172.30.1.2:30200
hello!
```

파드로 요청이 전송되는 것을 확인할 수있습니다.

<br>
## LoadBalancer

<img src="/assets/images/23/3.png"  width="500" height="500"/>

외부 로드 밸런서를 지원하는 **클라우드 (AWS , Azure, GCP 등 )상**에서, type 필드를 LoadBalancer로 설정하면 서비스에 대한 로드 밸런서를 프로비저닝합니다. 예를 들어 AWS EKS의 경우 다음과 같이 loadbalancer 타입 서비스 생성하게되면 NLB가 프로비저닝돼 포드를 외부 네트워크 트래픽에 노출할 수 있습니다.

예시 코드는 다음과 같습니다.

### loadbalancer.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: loadbalancer-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
spec:
  type: LoadBalancer
  selector:
    app: webui
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

LoadBalancer로 노출하는 각 서비스는 자체 IP 주소를 갖게 되고, **노출된 서비스마다 LoadBalancer에 대한 비용을 지불**해야 하기 때문에 사용시 체크가 필요합니다.

<br>
## Ingress

<img src="/assets/images/23/4.png"  width="700" height="500"/>

Ingress는 IP기반이 아니라 **URL 기반**으로 라우팅해 파드를 **외부 트래픽에 노출**하며 트래픽 **로드밸런싱 기능**을 제공합니다.

앞서 설명드린 kubernetes 서비스의 한 종류가 아니며 서비스들을 묶는 상위 오브젝트입니다. 여러 서비스로 라우팅할 수 있기때문에 **동일한 IP 주소로 여러 서비스를 노출하려는 경우 유용**합니다.

<br>
### ingress.yaml 예시

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    alb.ingress.kubernetes.io/load-balancer-name: alb
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/target-type: ip
  name: ingress
spec:
  ingressClassName: alb
  rules:
  - http:
      paths:
      - backend:
          service:
            name: backend
            port:
              number: 80
        path: /api
      - backend:
          service:
            name: frontend
            port:
              number: 80
        path: /front
status:
  loadBalancer:
    ingress:
    - hostname:
```

인그레스 자체는 이런 규칙들을 정의해둔 자원이고 이런 규칙들을 실제로 동작하게 해주는게 **인그레스 컨트롤러(ingress controller)**입니다.

클라우드 서비스를 사용하지 않고 직접 쿠버네티스 클러스터를 구축해서 사용하는 경우라면 인그레스 컨트롤러를 직접 인그레스와 연동해 주어야 합니다.

클라우드 서비스를 사용하게 되면 큰 설정없이 각 클라우드 서비스에서 인그레스를 사용할 수 있게 됩니다. **AWS EKS 환경에서 예시 코드와 같이 ingress를 생성하면 AWS ALB가 프로비저닝**됩니다.

---

## Reference
<https://kubernetes.io/ko/docs/concepts/services-networking/service/>
<https://kubernetes.io/ko/docs/concepts/services-networking/ingress/>
