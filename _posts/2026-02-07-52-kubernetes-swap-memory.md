---
layout: post
title: "EKS 환경에서 Swap 메모리를 활용해 OOM 방지"
author: "jhkim593"
tags: Kubernetes
---

현재 EKS 환경에서 어플리케이션 취약점 분석 서비스를 운영하고 있는데, 분석 작업 특성상 메모리 사용량 예측이 어려워 OOM이 빈번하게 발생하는 문제가 있었습니다.

이번 글에서는 노드 스펙업 없이 Swap 메모리를 활용해 OOM에 유연하게 대응한 방법에 대해 다뤄보겠습니다.

<br>

## 문제 상황

EKS 운영 환경에서 분석 파드는 취약점 분석 작업을 수행하고 있었는데, 분석 대상의 규모나 복잡도에 따라 메모리 사용량 편차가 컸습니다.

편차로 인해 메모리 사용량 예측이 어려워 노드 스펙을 여유롭게 잡고, 메모리 과도한 사용을 방지하고자 컨테이너에 메모리 limit을 설정해 관리하고 있었습니다.

하지만 limit을 여유롭게 설정하더라도 메모리를 많이 사용하는 작업이 간혹 발생했고, 이를 초과하면 OOM Kill이 발생했습니다.

OOM을 방지하려면 limit을 늘려야 하는데, 그러려면 다시 노드 스펙업이 필요했습니다. 노드 스펙업은 추가 비용이 발생하므로 매번 스펙을 올리는 것은 한계가 있었습니다.

<br>

## Swap 메모리로 해결할 수 있을까?

Swap 메모리를 미리 설정해놓으면, 물리 메모리가 부족할 때 디스크를 메모리처럼 사용할 수 있습니다.

디스크 기반이기 때문에 물리 메모리 보다 속도가 느리지만 **노드 스펙업 없이도 limit을 늘릴 수 있기 때문에 OOM을 더 유연하게 대응할 수 있습니다.**

<br>

## Swap 적용 전 확인 사항
EKS 환경에서 Swap을 사용하려면 별도 설정이 필요한데 Swap을 적용하기 전에 몇 가지 확인해야 할 사항을 정리했습니다.

<br>

#### 1. 컨테이너 limit 설정

Swap을 사용하더라도 컨테이너는 여전히 limit만큼만 메모리를 사용 할 수 있기 때문에 Swap을 사용하려면 **limit을 기존보다 여유롭게 설정**해야 합니다.

<br>

#### 2. QoS 설정

쿠버네티스에서 Swap을 사용하려면 파드의 QoS가 **Burstable**이어야 합니다. 여기서 QoS(Quality of Service)는 쿠버네티스가 파드의 리소스 설정을 기반으로 자동으로 부여하는 등급이며 총 세가지가 있습니다.

- **Guaranteed** : 모든 컨테이너의 CPU, Memory request = limit → **Swap 사용 불가**
- **Burstable** : request와 limit이 다르거나, 한쪽만 설정된 경우 → **Swap 사용 가능**
- **BestEffort** : request, limit 모두 미설정 → **Swap 사용 불가**

파드를 Guaranteed로 설정하면 Swap을 사용하지 않기 때문에 성능을 보장 할 수 있습니다.  

<br>



#### 3. vm.min_free_kbytes와 kubelet eviction.hard

kubelet의 `evictionHard` 설정은 사용 가능한 메모리가 임계값 이하로 떨어지면 파드를 강제 제거합니다.

```json
"evictionHard": {
    "memory.available": "100Mi"
}
```

eviction이 발생하기 전에 Swap을 사용하려면 `vm.min_free_kbytes` > `evictionHard`로 설정해야합니다.

`vm.min_free_kbytes`가 더 높으면 커널이 eviction 임계값에 도달하기 전에 미리 Swap으로 메모리를 확보하기 때문입니다.

<br>

#### 4. Instance Storage를 사용

Swap은 디스크를 사용하기 때문에, Instance Storage(로컬 NVMe 디스크)를 사용하면 EBS보다 성능이 향상됩니다.
하지만 Instance Storage를 지원하는 인스턴스 타입은 EBS만 지원하는 타입에 비해 비용이 더 발생합니다.

<br>

## Swap 설정 코드

Karpenter EC2NodeClass의 `userData`를 활용해 노드 시작 시 Swap을 구성하도록 설정했습니다. 

```yaml
  apiVersion: karpenter.k8s.aws/v1
  kind: EC2NodeClass
  spec:
    amiSelectorTerms:
      - alias: al2023@latest
  ...
  userData: |
    MIME-Version: 1.0
    Content-Type: multipart/mixed; boundary="//"

    --// 
    Content-Type: text/x-shellscript; charset="us-ascii"

    #!/usr/bin/env bash

    SWAPFILE="/swapfile"

    sudo fallocate -l 30G "$SWAPFILE"
    sudo chmod 600 "$SWAPFILE"
    sudo mkswap "$SWAPFILE"
    sudo swapon "$SWAPFILE"
    echo "Swap file has been created and enabled."

    sysctl -w vm.min_free_kbytes=204800
    
    --//
    Content-Type: application/node.eks.aws

    apiVersion: node.eks.aws/v1alpha1
    kind: NodeConfig
    spec:
      kubelet:
        config:
          failSwapOn: false
          featureGates:
            NodeSwap: true
          memorySwap:
            swapBehavior: LimitedSwap

```

주요 설정을 살펴보면 다음과 같습니다.

- `al2023@latest` : 기본 AL2는 Swap에 필요한 cgroupv2를 지원하지 않기 때문에 AL2023을 사용
- `fallocate -l 30G` : 30GB 크기의 Swap 파일 생성
- `mkswap` / `swapon` : Swap 포맷 초기화 및 활성화
- `vm.min_free_kbytes` : kubelet.evictionHard 기본값이 100Mi이므로, 더 높게 설정
- `failSwapOn: false` : Swap이 활성화된 상태에서도 kubelet이 정상 실행되도록 허용
- `featureGates.NodeSwap: true` : 노드 Swap 기능 활성화
- `swapBehavior: LimitedSwap` : 파드가 Swap을 제한적으로 사용하도록 허용

<br>
`LimitedSwap` 모드에서 각 컨테이너가 사용할 수 있는 Swap 양은 다음 공식으로 결정됩니다.

> **컨테이너 Swap 한도 = (컨테이너 Memory Request / 노드 전체 물리 RAM) x 노드 전체 가용 Swap 양**

이를 통해 쿠버네티스는 **특정 컨테이너가 전체 Swap을 독점하는 것을 방지**합니다.

<br>

## Instance Storage에 Swap 설정 코드

Instance Storage가 있는 인스턴스를 사용한다면, Swap 파일을 Instance Storage에 생성해 I/O 성능을 높일 수 있습니다.

다만 Instance Storage는 RAID 마운트 이후에 사용 가능하기 때문에, systemd 서비스로 마운트 완료 후 Swap을 설정하도록 구성해야 합니다.

```yaml
  apiVersion: karpenter.k8s.aws/v1
  kind: EC2NodeClass
  spec:
    amiSelectorTerms:
      - alias: al2023@latest
  instanceStorePolicy: RAID0
  ...
  userData: |
      MIME-Version: 1.0
      Content-Type: multipart/mixed; boundary="//"
      
      --//
      Content-Type: text/x-shellscript; charset="us-ascii"
      
      #!/usr/bin/env bash
      cat << 'EOF' > /etc/systemd/system/setup-swap.service
      [Unit]
      Description=Setup swap on instance storage
      After=mnt-k8s\x2ddisks-0.mount
      Before=kubelet.service
      
      [Service]
      Type=oneshot
      ExecStart=/usr/local/bin/setup-swap.sh
      RemainAfterExit=yes
      
      [Install]
      WantedBy=multi-user.target
      EOF
      
      cat << 'EOF' > /usr/local/bin/setup-swap.sh
      #!/bin/bash
      if mountpoint -q /mnt/k8s-disks/0; then
      SWAPFILE="/mnt/k8s-disks/0/swapfile"
      else
      SWAPFILE="/swapfile"
      fi
      
      fallocate -l 30G "$SWAPFILE"
      chmod 600 "$SWAPFILE"
      mkswap "$SWAPFILE"
      swapon "$SWAPFILE"
      
      sysctl -w vm.min_free_kbytes=204800
      EOF
      
      chmod +x /usr/local/bin/setup-swap.sh
      systemctl enable setup-swap.service

```

- `instanceStorePolicy: RAID0` : Instance Storage 디스크들을 RAID0으로 묶어 마운트
- `setup-swap.service` : Swap 설정을 systemd 서비스로 등록해 노드 시작 시 자동 실행
- `After=mnt-k8s\x2ddisks-0.mount` : Instance Storage RAID 마운트가 완료된 후 실행되도록 설정
- `Before=kubelet.service` : kubelet이 시작되기 전에 Swap이 준비되도록 보장
- `Type=oneshot` / `RemainAfterExit=yes` : 스크립트를 한 번 실행하고 서비스 상태를 활성으로 유지
- `mountpoint -q /mnt/k8s-disks/0` : Instance Storage 마운트 여부를 확인해, 마운트되어 있으면 Instance Storage에, 실패하면 기본 EBS에 Swap을 생성하도록 fallback 처리
- `systemctl enable setup-swap.service` : 노드 재시작 시에도 Swap 설정이 자동으로 적용되도록 서비스 등록

<br>

## 테스트 결과

기존에 OOM이 발생하던 분석 작업에 Swap을 적용하여 테스트를 진행했습니다. 요금 정책상 Instance Storage는 사용하지 않고 EBS에 Swap을 설정했습니다.

`system.memory.swap.used.bytes` 메트릭을 통해 물리 메모리 부족 시 Swap이 정상적으로 사용되는 것을 확인했으며, OOM 없이 작업이 정상 완료되었습니다.

<img src="/assets/images/52/1.png"  width="900" height="280"/>

<br>

---

## Reference
<https://kubernetes.io/docs/concepts/cluster-administration/swap-memory-management/>
<https://builder.aws.com/content/38j5cjxVTorJaNTu37HkPcDVcUs/how-to-enable-swap-memory-on-eks-using-karpenter>