---
layout: post
title: "Elastic Search (1)"
author: "jhkim593"
tags: ELK
---

## Elastic Search란?
Elastic Search는 Apache Lucene( 아파치 루씬 ) 기반의 Java 오픈소스 분산 검색 엔진입니다.

Elastic Search를 통해 루씬 라이브러리를 단독으로 사용할 수 있게 되었으며, 방대한 양의 데이터를 신속하게, 거의 실시간( NRT, Near Real Time )으로 저장, 검색, 분석할 수 있습니다.

Elastic Search는 검색을 위해 단독으로 사용되기도 하며, ELK( Elastic Search / Logstatsh / Kibana )스택으로 사용되기도 합니다.

ELK 스택이란 다음과 같습니다.

- Logstash
  - 다양한 소스( DB, csv파일 등 )의 로그 또는 트랜잭션 데이터를 수집, 집계, 파싱하여 Elastic Search로 전달
  - Elastic Search
Logstash로부터 받은 데이터를 검색 및 집계를 하여 필요한 관심 있는 정보를 획득
- Kibana
  - Elastic Search의 빠른 검색을 통해 데이터를 시각화 및 모니터링


<br>

ElasticSearch사용

- 애플리케이션 검색
- 웹사이트 검색
- 엔터프라이즈 검색
- 로깅과 로그 분석


### Elastic Search와 관계형 DB (RDBMS)의 비교

|RDBMS|Elastic Search|
|:--------:|:--------:|
|schema| mapping |
|database| index|
|table| type |
|row|document|
|column|field|

<br>

### Elastic Search 핵심
 - **클러스터 (cluster)** <br>
   클러스터란 Elastic Search에서 가장 큰 시스템 단위를 의미하며, 최소 하나 이상의 노드로 이루어진 노 드들의 집합입니다. 서로 다른 클러스터는 데이터의 접근, 교환을 할 수 없는 독립적인 시스템으로 유지되며, 여러 대의 서버가 하나의 클러스터를 구성할 수 있고, 한 서버에 여러 개의 클러스터가 존재할수도 있습니다.


- **노드 (node)**<br>
  Elastic Search를 구성하는 하나의 단위 프로세스를 의미합니다.
  그 역할에 따라 Master-eligible, Data, Ingest, Tribe 노드로 구분할 수 있습니다.



- **인덱스 (index)**<br>
  Elastic Search에서 index는 RDBMS에서 database와 대응하는 개념입니다. 이름으로 식별되며 모두 소문자여야 한다.

- **샤딩 (sharding)**<br>
즉, Elastic Search에서 스케일 아웃을 위해 index를 여러 shard로 쪼갠 것인데 콘텐츠 규모를 수평적으로 확장이 가능하도록 합니다.  
기본적으로 1개가 존재하며, 검색 성능 향상을 위해 클러스터의 샤드 갯수를 조정하는 튜닝을 하기도 합니다.


- **리플리카 (replica)**<br>
노드를 손실했을 경우 데이터의 신뢰성을 위해 샤드들을 복제하는 것이며
모든 리플리카에서 병렬 방식으로 검색을 실행할 수 있으므로 검색 볼륨/처리량을 확장할 수 있습니다.


### Elastic Search 특징
- **빠른 검색** <br> Elastic Search는 Lucene을 기반으로 구축되기 때문에, 전체 텍스트 검색에 뛰어납니다. Elastic Search는 또한 거의 실시간 검색 플랫폼이므로  문서가 색인될 때부터 검색 가능해질 때까지의 대기 시간이 아주 짧습니다. 결과적으로, Elastic Search는 보안 분석, 인프라 모니터링 같은 시간이 중요한 사용 사례에 이상적입니다.<br>
또한 Elastic Search는 텍스트를 파싱해서 검색어 사전을 만든 다음에 inverted index 방식으로 텍스트를 저장하기 때문에 RDBMS보다 전문검색( Full Text Search )에서 빠른 성능을 보입니다.

- **분산 저장** <br> Elastic Search는 본질상 분산적입니다. Elastic Search에 저장된 문서는 샤드라고 하는 여러 다른 컨테이너에 걸쳐 분산되며, 이 샤드는 복제되어 하드웨어 장애 시에 중복되는 데이터 사본을 제공합니다.

- **데이터 수집 ,시각화, 보고 간소화** <br>Elastic Stack은 데이터 수집, 시각화, 보고를 간소화합니다. Beats와 Logstash의 통합은 Elastic Search로 색인하기 전에 데이터를 훨씬 더 쉽게 처리할 수 있게 해줍니다. Kibana는 Elastic Search 데이터의 실시간 시각화를 제공하며, UI를 통해 애플리케이션 성능 모니터링(APM), 로그, 인프라 메트릭 데이터에 신속하게 접근할 수 있습니다.

<br>


## Kibana
Kibana는 Elasticsearch를 위한 시각화 및 관리 도구로서, 실시간 히스토그램, 선 그래프, 파이 차트, 지도 등을 제공합니다. Kibana에는 사용자가 자신의 데이터를 기반으로 사용자 정의한 동적 인포그래픽을 만들 수 있는 Canvas, 위치 기반 정보 데이터를 시각화하기 위한 Elastic Maps 같은 고급 애플리케이션도 포함됩니다.



-------
<br>
## Reference

https://choseongho93.tistory.com/231 [TROLL]
