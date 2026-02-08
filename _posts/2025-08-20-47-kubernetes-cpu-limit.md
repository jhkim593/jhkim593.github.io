---
layout: post
title: "Redis Lua Script를 활용한 동시성 처리 로직 구현"
author: "jhkim593"
tags: Kubernetes
---

Kubernetes CPU Limit 해제 실험기: 워커 파드 성능 최적화

쿠버네티스에서 프로덕션 환경을 운영하다 보니 파드의 CPU limit 설정을 어떻게 가져가야 적절할지 의문이 생길때가 많았는데 이번장에서는 파드의 CPU limit 설정을 변경해
성능 개선을 이룬 내용에 대해서 다뤄보겠습니다.

이번 글에서는 SAST / SCA 워커 파드를 대상으로 CPU limit을 해제했을 때 성능이 어떻게 달라지는지 실험한 결과를 공유합니다.

1. CPU Limit, 왜 해제를 고민했을까?
   현재 프로덕션에서 파드의 CPU limit 설정해서 특정 파드의 CPU 독점을 방지하고 있ㅅ

현재 운영 중인 워커 파드 구성은 대략 다음과 같습니다.

Worker 컨테이너: 실제 분석 작업 실행

Tool 컨테이너: 분석 도구 실행 및 서브 프로세스 관리

처음에는 두 컨테이너 모두 CPU request + limit을 동일하게 설정했습니다.
이렇게 하면 안정적으로 동작은 하지만, 문제가 있었습니다.

노드에 여유 CPU가 남아 있어도 limit 값 이상은 절대 못 씀

분석은 CPU 사용량이 높을수록 빠르게 끝날 수 있는데, 인위적 제한 때문에 병목 발생

👉 그래서 CPU limit을 없애면 남는 CPU를 더 활용해 성능이 개선될 수 있지 않을까? 하는 가설을 세웠습니다.

2. CPU Limit 해제 시 주의할 점

모든 컨테이너의 limit을 해제하는 건 위험할 수 있습니다.
왜냐하면 하나의 파드라도 CPU를 무한정 사용해버리면, 같은 노드에 있는 다른 파드들이 피해를 볼 수 있기 때문이죠.

따라서 중요한 건 requests 설정입니다.

requests: 컨테이너가 실행될 때 최소 보장되는 CPU

limit: 컨테이너가 사용할 수 있는 최대치

limit을 해제하더라도 requests를 제대로 잡아주면 최소 자원은 보장되고, 나머지는 남는 CPU를 유동적으로 나눠 쓰는 구조가 됩니다.

3. 실험 환경

노드 스펙:

2 Core / 16 GB

(확장 케이스) 4 Core / 32 GB

대상 파드:

sast-worker

sca-worker

비교 항목:

기존 (limit + request 동일)

limit 해제 후 (request만 낮게 설정)

평가 지표: 분석 시간, CPU 사용량

4. 실험 결과
   (1) SAST 워커

기존 설정

tool: request/limit 1250m

worker: request/limit 250m

분석 시간: 15분

limit 해제 후

tool: request 700m

worker: request 250m

분석 시간: 9분

파드 1개 단독 실행 시: 6분

👉 Worker 컨테이너가 추가 550m CPU를 활용할 수 있었고, 분석 시간이 최대 60% 단축되었습니다.

(2) SCA 워커

기존 설정

tool: request/limit 1000m

worker: request/limit 500m

분석 시간: 35분

limit 해제 후

tool: request 500m

worker: request 250m

분석 시간: 28분

파드 1개 단독 실행 시: 25분

👉 Worker 컨테이너가 추가 750m CPU를 확보했고, 분석 시간이 약 29% 단축되었습니다.

5. 해석

실험 결과는 명확했습니다.

CPU limit 해제 = 유휴 CPU 활용 가능 = 성능 개선

Tool 컨테이너는 request를 낮게 줘도 Worker 보장치가 유지되므로 문제 없음

Worker는 CPU를 더 가져갈 수 있어 분석 속도가 크게 개선

특히 노드 스펙이 커질수록 이 효과는 더 커집니다.
예를 들어 4 Core 노드에서 워커 파드 2개가 동시에 실행되면,
각 워커는 기본 보장치(750~950m) + 남는 CPU 1600~2000m를 나눠 가질 수 있습니다.

6. 인사이트 & 운영 전략

CPU limit은 자원 보호를 위한 안전장치이지만, CPU 집약적인 워크로드에서는 성능 저하 요인이 될 수 있습니다.

운영 환경에서는 다음 전략을 고려할 수 있습니다.

Worker Pod: limit 해제 + request 최소 보장치 설정

Tool/모니터링 Pod: request만 설정, limit 불필요

노드 리소스 모니터링 강화: 특정 파드가 과도하게 점유하지 않도록 감시

7. 결론

이번 실험을 통해 CPU limit 해제가 실질적인 성능 개선으로 이어질 수 있음을 확인했습니다.
특히 분석 워크로드처럼 CPU가 곧 성능과 직결되는 경우,
requests 중심의 리소스 관리가 limit 중심의 관리보다 더 효과적일 수 있습니다.

👉 실제 측정 결과,

SAST 워커는 분석 시간이 15분 → 6분(최대 60% 단축)

SCA 워커는 분석 시간이 35분 → 25분(약 29% 단축)

Kubernetes 자원 관리에서 중요한 건 무조건적인 제한이 아니라, 워크로드 특성에 맞는 자원 활용 전략임을 다시 한번 확인할 수 있었습니다.