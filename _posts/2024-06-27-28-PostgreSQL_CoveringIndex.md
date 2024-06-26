---
layout: post
title: "PostgreSQL - Covering Index 적용을 통해 페이징 성능 개선"
author: "jhkim593"
tags: PostgreSQL
---
# Covering Index

쿼리를 충족하는데 필요한 데이터를 모두 갖는 인덱스를 뜻합니다.

일반적인 인덱스 스캔시에는 row를 찾기 위해 **index 영역**과 메인 데이터가 저장된 **heap 영역**모두에 접근해 데이터를 탐색해야합니다.
하지만 covering index를 사용하면 **실제 데이터 영역에 접근할 필요 없이 인덱스만으로 빠르게 데이터를 조회** 할 수 있습니다.

covering index를 이용해 heap 영역 접근 없이 인덱스 내에서만 데이터를 조회하는 것을`Index Only Scan`이라 부릅니다.

<br>
## Covering Index 사용

실행 쿼리에 인덱스로 설정된 컬럼만 사용되어야합니다.

a , b 컬럼에 인덱스가 설정되어 있고 c 컬럼에는 인덱스가 설정되어있지 않은 상태라고 한다면

```sql
SELECT a, b FROM issue i WHERE i.a= 3;
SELECT a FROM issue i WHERE i.a='2' AND b="test";
```
해당 쿼리는 **index only scan**이 가능합니다.

하지만 이 쿼리는 인덱스가 설정되지 않은 c 컬럼이 **select , 조건절**에 사용되었기 때문에 **index-only scan이 불가능**합니다.
```sql
SELECT a, c FROM issue i WHERE i.a=3;
SELECT a FROM issue i WHERE i.a='2' AND c ="test";
```

<br>
커버링 인덱스가 적용된 쿼리 플랜 예시는 다음과 같습니다.

<img src="/assets/images/28/1.png"  width="750" height="120"/>

쿼리플랜을 보면 Index Only Scan이 적용된 것을 확인 할 수있습니다.

<br>
## Covering Index 생성

쿼리 실행시에 사용되지 않지만 단순히 `Index only scan`을 위해 특정 칼럼들을 인덱스에 놓고 싶을때는 `include` 를 사용하면 됩니다. 예시는 다음과 같습니다.

```sql
SELECT b FROM issue WHERE a = 'key';
```
위와 같은 경우 `a`가 탐색의 조건으로 사용되었고, `b` 컬럼을 조회하고 있습니다. 커버링 인덱스 적용을 위해 단순하게  `b`에도 인덱스를 걸면 되지만  **`b`가 검색의 조건으로 사용되지 않는다는 점에서는 비효율적**입니다.

이런 상황에서는 `include` 키워드를 이용해서 인덱스를 생성하면됩니다.

```sql
CREATE INDEX index ON issue(a) INCLUDE (b);
```
위와 같이 인덱스를 생성하면  `b` 가 인덱스 테이블에 저장되지만 탐색에는 사용되지 않습니다. 하지만 사이즈가 큰 컬럼을 인덱스 테이블에 저장하는 것은 주의해야합니다. 인덱스 테이블 크기가 증가해 **탐색시 성능 저하**가 일어날 수 있고 인덱스 유형이 허용하는 최대 사이즈를 초과하면 **insert 예외가 발생** 할 수있기 떄문입니다.

<br>
## visibility

기본적으로 위에서 설명드린 조건에 부합하면 **Index Only Scan**이 가능하지만 검색 결과 각 Row가 쿼리의 **MVCC 스냅샷에 보이는지 여부**에 따라 동작이 달라질 수 있습니다.

**visibilty 정보**는 인덱스 영역이 아닌 heap 영역에 저장되기 때문에 **visibilty를 확인**하기 위해서는 인덱스 영역외 추가로 heap 영역을 조회해야합니다.

하지만 매번 heap을 조회하는 것은 아닙니다. Postgresql은 페이지 상태를 visibilty map에 저장하고 있는데 visibilty map bit에 데이터가 **모두 visible 하다면 추가로 heap에 접근하지 않고 visible 하지않은 데이터가 있다면 heap에 접근**합니다.

visibilty map은 heap에 접근하는 것보다 디스크 I/O가 적게들어 빠르기 때문에 **visibilty map 업데이트에 따라 index only scan 성능이 달라질 수 있습니다.**


<br>
위 내용을 간단하게 테스트를 해보도록 하겠습니다.

`checker_key`, `id`컬럼에 인덱스를 설정하고 `autovacuum_enabled = OFF` 로 autoVacuum을 비활성화 한뒤 테이블에 update 쿼리를 날려보겠습니다. 테스트 대상 테이블에는 약 250만 Row가 저장돼있습니다.

```sql
CREATE INDEX INDEX ON issue(checker_key) include(id)

ALTER TABLE issue SET (autovacuum_enabled = OFF);
UPDATE issue  SET risk = 6  where id < 2500000
```
update 쿼리가 실행됐기 때문에 VM `ALL_VISIBLE`은 FALSE인 상태입니다.
이제 쿼리를 날려 index only scan을 통해 데이터를 조회해보겠습니다.

```sql
EXPLAIN ANALYZE
SELECT id from issue where checker_key = 'test.key'
```

<img src="/assets/images/28/2.png"  width="750" height="120"/>

index only scan을 통해 조회를 했지만 heap Fetches가 327,417로 해당 row 수만큼 heap에 접근한 것을 알 수있습니다.

<br>
여기서 **VACUUM을 통해 visibility map을 업데이트**하고 다시 같은 쿼리로 조회해보도록 하겠습니다.

```sql
VACUUM  issue;

EXPLAIN ANALYZE
SELECT id from issue where checker_key = 'test.key'
```

<img src="/assets/images/28/3.png"  width="750" height="120"/>

heap Fetches가 0이고 실행시간이 269ms에서 64ms로 감소된 것을 확인 할 수 있습니다.  테스트 상황에서는 미미한 차이였지만 heap Fetches 값이 커질수록 추가적인 I/O가 많이 발생해 성능이 저하될 수 있습니다.

<br>
# Covering Index를 적용해 페이징 성능 개선

<br>
## LIMIT-OFFSET 방식의 문제점
페이징 처리를 하기위해 흔히 `LIMIT - OFFSET` 방식을 사용합니다. 하지만 `LIMIT - OFFSET` 방식은 `OFFSET` 값만큼 Row를 읽은 후 `LIMIT` 값만큼의 Row를 가져오는 방식으로 동작합니다.

`OFFSET` 값이 크지 않을 때는 괜찮지만 테이블 크기가 커지면서 `OFFSET` 값이 커진다면 그만큼 데이터 조회를 위해 디스크 I/O가 많이 발생하기 때문에 조회 성능이 저하될 수 있습니다.

약 250만개 Row가 저장된 테이블 대상으로 간단하게 확인해보도록 하겠습니다.

```sql
EXPLAIN ANALYZE SELECT * from issue OFFSET 30 LIMIT 10
```

offset을 30정도로 설정하게되면

<img src="/assets/images/28/4.png"  width="750" height="120"/>

40개 Row를 순차적으로 스캔해서 0.136ms가 걸린것을 알 수있습니다.

<br>
여기서 offset만 250만으로 올려서 같은 쿼리를 실행해 보도록 하겠습니다.

<img src="/assets/images/28/5.png"  width="750" height="105"/>

250만개 Row를 순차적으로 스캔해서 2712 ms가 걸렸으며 offset 30일 때와 비교해 시간이 오래걸린것을 확인할 수 있었습니다.


<br>
## 서브 쿼리를 통해 Covering Index 적용
일반적인 `LIMIT - OFFSET` 방식은 OFFSET이 커지게 되면 실제 데이터 영역에 더 접근해야하기 때문에 성능 저하에 영향을 주는 것을 확인했습니다. 하지만 `LIMIT - OFFSET` 방식에 추가로 커버링 인덱스를 적용하게되면 **실제 데이터에 접근하지 않고 인덱스 데이터만으로 결과를 구성하기 때문에 디스크 I/O를 줄일 수 있어 성능이 향상**이 가능합니다.

`WHERE` , `ORDER BY`를 추가해 쿼리를 다시 구성해보도록 하겠습니다.

```sql
SELECT * from issue where checker_key ='test.key' ORDER BY id OFFSET 100000 LIMIT 10
```
`WHERE` 와 `ORDER BY` 를 추가해 조회 쿼리를 구성했습니다. 만약 해당 쿼리에 커버링 인덱스를 적용하려고 한다면 **`SELECT` 절에 모든 컬럼이 인덱스에 포함**되어 있어야합니다. 하지만 테이블의 모든 컬럼을 가지고 있는 인덱스를 생성하게 되면 큰 **오버헤드가 발생**할 것입니다.

이런 상황에서 커버링 인덱스를 통해 페이징 하는 부분을 **서브쿼리**로 나눠서 해결할 수 있습니다.

```sql
select *
from (
      select id
      from issue
      where checker_key = 'test.key'
      ORDER BY id
      OFFSET 100000
			LIMIT 10
) b join issue a on b.id = a.id
```

먼저 서브쿼리를 통해 커버링 인덱스로 페이징합니다. 서브쿼리 select 절에는 **id만 포함**이 되어있기 떄문에 모든 컬럼을 포함하는 인덱스를 생성하지 않아도 커버링 인덱스가 적용됩니다. 이후 join을 통해서 10개 Row에 대해서만 **실제 데이터 영역에 접근해 조회**합니다.

서브 쿼리로 조회 후 join하게되면 모든 컬럼이 아닌 10개 Row에 대해서만 데이터 영역에 접근하기 때문에 성능상 이점을 얻을 수 있습니다.

<br>
인덱스를 생성해서 테스트를 해보겠습니다.

```sql
CREATE INDEX TEST_INDEX ON issue(checker_key, id)
```

checker_key , id 컬럼에 대해 index를 생성했습니다.

첫번째 쿼리의 쿼리 플랜을 살펴보면

<img src="/assets/images/28/6.png"  width="900" height="250"/>

3039 ms 가 걸리는 것을 확인 할 수 있습니다.

<br>
하지만 **서브 쿼리를 적용**한 두번째 쿼리의 쿼리 플랜을 살펴보면

<img src="/assets/images/28/7.png"  width="800" height="220"/>

77ms로 조회 되는 데이터는 같지만 실행시간은 크게 차이나는 것을 확인 할 수 있습니다.

<br>
## 주의사항
페이징 쿼리에 커버링 인덱스를 적용할 때 주의 사항은 다음과 같습니다.

- 인덱스가 너무 커질 수 있음
    - 커버링 인덱스를 적용하기 위해서는 `WHERE` 절 외에도 `ORDER BY` , `GROUP BY` , `HAVING` 등에 들어가는 컬럼들이 인덱스에 포함되어야 합니다. 하지만 인덱스 크기가 너무 커지게 되면 성능상 이슈가 발생할 수 있습니다.
- 주기적인 VACUUM 필요
    - 커버링 인덱스는 실제 데이터 영역인 heap에 접근하기전 VM을 확인하는데 **VM은 VACUUM시에 업데이트 되기 때문에** VACUUM이 잘 동작하지 않았다면 커버링 인덱스로 인한 성능 향상을 기대하기 어렵습니다.
