---
layout: post
title: "PostgreSQL - VACUUM"
author: "jhkim593"
tags: PostgreSQL
---
**VACUUM** 이란 Postgresql에만 존재하는 특수한 개념이며 Postgresql의 MVCC 구현 방식이 다른 DBMS 구현 방식과 달라서 생기는 문제를 처리하기 위해 등장했습니다.

Postgresql은 VACUUM을 통해 아래와 같은 작업을 수행합니다.

- Dead Tuple을 정리해 디스크 공간 확보
- Transaction ID Wraparound 방지
- visibility map을 갱신하여 index scan 성능 향상

이번장에서는 위 3가지 작업 및 VACUUM 동작에 대해 살펴볼 것인데 그전에 Postgresql의 MVCC 동작 방식에 대해 살펴보도록 하겠습니다.

> Postgresql 12.15 버전을 대상으로 테스트 했습니다.

<br>
## Postgresql MVCC

MVCC(Multi Version Concurrency Control)란 다중 버전 동시성 제어를 뜻하며 다양한 버전 데이터들에 대한 관리가 가능한 기능입니다.

대부분의 DBMS는 동시성 처리를 위해 MVCC가 구현되어있으며 기본적으로 트랜잭션이 시작된 시점의 ID와 같거나 작은 트랜잭션 ID를 가지는 데이터를 읽도록 동작합니다.

Postgreqsql에서는 변경되기 이전 튜플과 변경된 신규 튜플을 같은 페이지에 저장하고 튜플별로 특정 시점의 데이터를 추출하기 위해 **XMIN / XMAX 값을 기록**하는 것으로 MVCC를 구현합니다.

#### XMIN

- Insert의 경우 insert된 신규 튜플 XMIN에 수행 시점의 Tracnscation ID를 저장
- Update의 경우 update된 신규 튜플 XMIN에 수행 시점의 Tracnsaction ID를 저장

#### XMAX

- Update의 경우 변경 이전 튜플의  XMAX에 수행 시점의 Tracnscation ID를 저장
- Delete의 경우 변경 이전 튜플의 XMAX에 수행 시점의 Tracnsaction ID를 저장

예시는 다음과 같습니다.

> XID 999 시점에 insert 쿼리로 A Row가 저장되었고
> XID가 1000인 시점에 select 쿼리가 계속 수행 중이며 동시에
> XID 1001 시점에 A Row를 A1 Row로 변경하는 update 쿼리가 완료되었다고 가정해보겠습니다.


A Row는 조회 시점보다 과거( (XMIN = 999 )에 입력되었고 미래 시점 (XMAX = 1001)에 변경되었기 때문에 조회 대상에 포함됩니다.

A1 Row는 조회 시점보다 미래 (XMIN = 1001 )에 입력되었기 때문에 조회 대상에 포함되지 않습니다.

<br>
이제 앞서 언급한 **VACUUM이 처리하는 작업** 3가지에 대해 알아보겠습니다.

<br>
## Dead Tuple 정리

앞서 말씀드린 것처럼 Postgresql의 MVCC는 변경 혹은 제거된 튜플들이 제거되는 것이 아니고 XMIN, XMAX 값을 통해 읽기 가능한 데이터를 구별하는 방식으로 동작합니다. 이러한 동작방식 때문에 **변경 이전의 값을 저장하고 있는 튜플**이 생기는데 이를 **Dead Tuple** 이라고합니다.

Dead Tuple들은 FSM에 반환되지 않고 메모리를 차지하고 있기 때문에 제거해주지 못하면 **메모리 낭비가 발생**하며 Dead Tuple 때문에 더 많은 페이지를 읽게되고 이는 **쿼리 성능 저하**에도 영향을 미칠 수 있습니다.

Postgresql은 데이터를 읽을 때 페이지 단위로 읽게되는데 MVCC 동작 방식 특성상 페이지에 Live Tuple과 Dead Tuple이 같이 포함됩니다. 페이지 크기는 기본 8KB로 한정되어 있기 때문에 Live Tuple을 읽기 위해서 더 많은 페이지를 읽게된다면 **디스크 I/O 가 더 발생하므로 쿼리 성능 저하**에 영향을 줄 수 있습니다.

이런 문제를 방지하기 위해 Postgresql 에서는 VACUUM 동작을 수행해 쌓인 Dead Tuple 들을 정리합니다.

<br>
## **Transaction ID Wraparound 방지**

Postgresql의 경우 Row 단위로 4 byte의 TransactionID로 XID를 할당해 데이터 입력 및 변경 시점을 저장하는데 4byte 인 XID는 표현 할 수 있는 값이 43억개 정도로 한정되어있습니다.

만약 표현 가능한 Transaction을 모두 돌고 난뒤 1이라는 XID 값을 할당받은 새로운 Transaction은 ID값이 2인 Transaction 보다 최신임에도 모든 데이터가 미래 데이터가 되어 어떤 값도 읽을 수 없게됩니다. 이러한 상황을 **Transaction ID Wraparound** 라고합니다.

**Transaction ID Wraparound** 문제는 운영 환경 발생시에 치명적인 문제가 되기 때문에 Postgresql은 VACUUM 작업을 통해 오래된 트랜잭션에 **FrozenXID를 부여**하고 FrozenXID를 부여받은 튜플은 Old 버전이라는 의미로서 항상 읽기가 가능하도해 이 문제를 해결했습니다.

위처럼 FrozenXID를 부여하는 작업을 Data Freezing이라고 하는데
Data Freezing 대상을 선정하기 위해 생성 후 얼마나 오래되었는지 확인해야하는데 이때 **Age** 라는 개념을 사용합니다.

Age는 생성 시점의 **XID와 Current XID의 차이**를 의미하며 대상은 크게 Table 과 Tuple입니다.
DB에서 트랜잭션이 발생할 때마다 age가 1씩 증가합니다.Age가 계속 증가하다가 설정된 임계치에 도달하게 되면 **VACUUM시 Date Freezing**이 발생합니다.

<br>
Tuple은 Data freezing 대상이며 `vacuum_freeze_min_age` 파라미터 보다 age가 높은 Tuple에 VACUUM시 Date Freezing이 발생합니다.

Table은 Tuple과 다르게 Date Freezing 대상은 아니지만 **Tuple들의 Age중 가장 높은 값이 설정**되기 때문에 Date Freezing을 필요로하는 Tuple의 존재 여부를 쉽게 파악할 수 있게 합니다.

<br>
## visibility map 갱신

visibility map ( VM )이란 테이블을 구성하는 개별 페이지의 상태를 2개의 bit값으로 표현하는 파일이며 페이지가 포함하는 Row들의 상태 정보를 담고있습니다.

VM은 VACUUM에 의해서만 설정되며 페이지 직접 접근 여부를 빠르게 판담함으로써 불필요한 I/O 작업을 줄여주는 역할을 합니다.

VM은 Postgresql 9.6 이후 버전 부터 페이지 당 2bit의 Flag 값을 표현합니다.

<img src="/assets/images/27/1.png"  width="750" height="500"/>

#### VM에 저장되는 데이터

첫번째 bit는 `ALL_VISIBLE` 으로  페이지에 포함된 **Row가 모든 트랜잭션에 보이는 상태**인지를 나타냅니다. 값이 1( True )인 경우 페이지 Row가 모두 Visible 한 상태이며 0( False )인 경우 페이지에서 정리해야 할 튜플이 포함되어있음을 의미합니다.

VACUUM 수행시 `ALL_VISIBLE` 값이 False인 페이지만 대상으로 하기 때문에 페이지 접근을 줄일수 있어 **VACUUM 성능이 향상**됩니다. 또한 Index Only Scan시 페이지의 `ALL_VISIBLE` 값이 True인 경우 테이블에 접근하지 않아도 되기때문에 **쿼리 성능이 향상**됩니다.

두번째 bit는 `ALL_FROZEN` 으로 `ALL_VISIBLE` 값이 True 인 경우에만 의미가 있으며 모든 Row가 Freeze되었는지 여부를 나타냅니다. 여기서 Freeze는 FrozenXID를 부여하는 작업을 의미합니다. `ALL_FROZEN` 값이 True인 경우 VACUUM 수행시 Row를 Freezing하는 작업을 스킵하게되어 **VACUUM 성능이 향상**됩니다.

<br>
#### VM 동작

<img src="/assets/images/27/2.png"  width="600" height="400"/>

- VACUUM을 수행하지 않은 최소 생성된 페이지 VM 값은 (0, 0 )
- VACUUM 작업을 수행한 이후 `ALL_VISIBLE` (1, 0) 되는데 추가로 FREEZE 작업까지 했다면 (1,1)
- DML로 데이터 업데이트 시 (0, 0)
- VACUUM FULL 수행시 ( 0, 0)

visibility_map은 다음 쿼리로 조회 할 수 있습니다.
```sql
CREATE EXTENSION IF NOT EXISTS pg_visibility;
SELECT blkno, all_visible, all_frozen from pg_visibility('table_name')
```

<br>
## VACUUM 종류

VACUUM은 사용 명령어에 따라 크게 `VACUUM` , `VACUUM FULL` , `VACUUM FREEZE` 등으로 나뉩니다.

`VACUUM` : OS에 디스크 공간 반환까지는 처리되지 못하고 FSM에만 반환되어 Dead Tuple이 있던 공간을 **재사용 할 수 있게 처리**됩니다. Postgresql 12버전 이후 부터 TRUNCATE가 디폴트로 동작해 디스크 공간이 확보될 수 있습니다.

`VACUUM FULL`  : Old 버전 데이터의 물리적인 공간을 **OS에 반환함으로써 디스크 공간을 확보**하는 역할을 수행합니다. 이 때 Live Tuple들을 모아서 새로운 테이블을 구성하고 기존 테이블을 삭제하도록 동작하며 디스크 공간을 확보하는데 이때 Table에 대한 Exclusive Lock을 획득해 동시성 작업에 문제가 생길 수 있으며 새로운 테이블을 생성하는 과정에서 일정량 디스크 공간이 존재해야합니다.

`VACUUM FREEZE` : 트랜잭션 ID 제한으로 인한 오래된 자료의 손실을 방지하고자 FrozenXID를 부여합니다.

<br>
아래에서는 예시를 통해 `VACUUM` , `VACUUM FULL` 동작에 대해 살펴 보도록 하겠습니다.

```sql
ALTER TABLE issue SET (autovacuum_enabled = OFF);
EXPLAIN ANALYZE SELECT COUNT(*) FROM issue;
```

<img src="/assets/images/27/3.png"  width="200" height="30"/>
테스트를 위해 AutoVacuum은 수행되지 않도록 했으며 약 250개의 Row가 저장된issue 테이블에 count 쿼리 수행 시간은 175ms 소요되었습니다.

<br>
이제 Delete 쿼리를 통해 Row를 하나만 남기고 모두 삭제한 뒤 다시 count 쿼리를 날려보도록하겠습니다.

```sql
DELETE FROM issue WHERE id != 16714426;
EXPLAIN ANALYZE SELECT COUNT(*) FROM issue;
```

<img src="/assets/images/27/4.png"  width="200" height="30"/>

issue 테이블에 Row는 하나밖에 남지 않았지만 Row를 삭제하기 전과 count 쿼리 수행 시간이 비슷한 것을 알 수 있습니다.

<br>
실제 테이블 크기를 확인해보면 데이터는 하나 밖에 없는데 300MB 가량 되는 것을 확인 할 수있습니다..
```sql
SELECT pg_size_pretty(pg_relation_size('issue'))
```

<img src="/assets/images/27/5.png"  width="180" height="70"/>

<br>
여기서 테이블에 Live , Dead Tuple 확인해보면

```sql
SELECT relname, n_live_tup, n_Dead_tup
FROM pg_stat_user_tables;
```

<img src="/assets/images/27/6.png"  width="200" height="30"/>

Live Tuple은 1이고 Dead Tuple은 약 250만개 인것을 알 수있습니다. Dead Tuple이 정리되지않아 count 쿼리에 수행에 영향이 있었기 때문에 VACUUM을 통해 Dead Tuple을 처리해보겠습니다.

```sql
VACUUM ISSUE;
SELECT relname, n_live_tup, n_Dead_tup
FROM pg_stat_user_tables;
```

<img src="/assets/images/27/7.png"  width="200" height="30"/>

issue 테이블에 VACUUM을 수행시킨 뒤에 Dead Tuple이 모두 처리된 것을 확인 할 수있습니다.

<br>
다시 동일한 count 쿼리를 날려보면

```sql
EXPLAIN ANALYZE SELECT COUNT(*) FROM issue;
```

<img src="/assets/images/27/8.png"  width="200" height="30"/>

쿼리 수행시간도 0.028ms로 감소된것을 확인할 수 있습니다.

<br>
하지만 테이블 사이즈를 확인해보면

```sql
SELECT pg_size_pretty(pg_relation_size('issue'))
```

<img src="/assets/images/27/9.png"  width="180" height="60"/>

Row는 한개인데 256MB로 기존과 크게 변함이 없는 것을 알 수있습니다. Postgresql 12버전 이후 부터 VACUUM 수행시 TRUNCATE이 기본적으로 동작해 테이블 끝에 있는 빈 페이지를 잘라내고 잘린 페이지에 대한 디스크 공간을 OS에 반환하기 때문에 테이블 크기 변화가 생기는 것을 볼수 있습니다.

<br>
마지막으로 VACUUM FULL을 수행해보면

```sql
VACUUM FULL ISSUE
SELECT pg_size_pretty(pg_relation_size('issue'))
```

<img src="/assets/images/27/10.png"  width="150" height="60"/>

사용되지 않는 공간을 모두 반환해 테이블 사이즈가 8192 byte로 감소한 것을 확인 할 수있습니다.

<br>
## Autovacuum

VACUUM 동작을 자동을 수행하는것을 autoVacuum 이라고 하며 크게 두가지 상황에서 수행됩니다.

- Dead Tuple의 개수가 임계치에 도달했을 때 → Dead Tuple을 정리
- Table , Tuple의 age가 임계치에 도달했을 때 → Transaction ID Wraparound 방지를 위해 Data Freezing

<br>
### Dead Tuple의 개수가 임계치에 도달했을 때

해당 케이스는 쿼리 성능 문제 또는 디스크 공간 부족 문제를 방지하기 위해 수행되며 다음식을 통해 임계치가 계산됩니다.

```sql
vacuum threshold = autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor * 튜플수
```

계산된 임계치보다 Dead Tuple이 쌓이게되면 AutoVacuum이 호출되며 아래 쿼리로 확인 가능합니다.

```sql
select name,setting from pg_settings where name in ('autovacuum_vacuum_scale_factor', 'autovacuum_vacuum_threshold');
```

<img src="/assets/images/27/11.png"  width="300" height="80"/>

디폴트는 autovacuum_vacuum_scale_factor 0.2  , autovacuum_vacuum_threshold 50 으로 설정되어 있으며 디폴트 값을 기준으로 테이블 Dead Tuple 수가 모든 튜플 (LIVE + DEAD) * 0.2 + 50 개를 초과하는 경우 AutoVacuum이 호출됩니다.

VACUUM 수행에 문제가 되는 테이블이 있다면 직접 AutoVacuum 임계치를 설정할 수 있습니다.

```sql
ALTER TABLE issue SET (autovacuum_vacuum_scale_factor = 0.2, autovacuum_vacuum_threshold= 0 );
```

현재 issue 테이블에 5개 Row가 있고 위와 같이 설정했다고 설정했다고 가정해보겠습니다. 5개 중 2개 Row를 delete 하게되면 임계치인 5*0.2 = 1를 Dead Tuple 수 (= 2)가 초과하기 때문에 **AutoVacuum**이 동작합니다.

<br>
### **Table , Tuple의 age가 임계치에 도달했을 때**

기본적으로 테이블의 Age가 `AUTOVACUUM_FREEZE_MAX_AGE` 를 초과한 시점에 AutoVacuum이 수행되며 기본값은 2억입니다.

---

## Reference
<https://postgresql.kr/docs/9.6/routine-vacuuming.html>
