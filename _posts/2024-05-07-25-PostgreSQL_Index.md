---
layout: post
title: "PostgreSQL - Index"
author: "jhkim593"
tags: PostgreSQL
---
DB에 데이터가 점점 쌓이면서 한 테이블에 약 200만건이 넘는 데이터가 저장되었었습니다. 이로 인해 조회 속도가 느려지는 문제가 발생했었지만 테이블에 **Index**를 적용해 해결 할 수 있었습니다.
이번 장에서는 조회 속도 향상을 위한 데이터 베이스 **Index**에 대해 살펴보도록 하겠습니다.

<br>
# Index

테이블의 **조회** 속도를 높여주는 자료구조입니다. 인덱스가 설정되지 않았다면 **Table Full Scan**이 일어나 테이블에 저장된 데이터가 많을 경우 성능이 저하될 수 있습니다.

`SELECT`의 경우 성능이 향상되지만 `INSERT`의 경우 데이터 추가로 인해 기존 탐색 위치가 수정되기 때문에 효율이 좋지 않습니다. `UPDATE` , `DELETE` 의 경우 업데이트 할 데이터를 찾을 때의 속도는 빨라지지만 `INSERT`와 마찬가지로 탐색 위치 수정으로 인해 효율이 좋지 않습니다.

인덱스가 설정되어 있다고해도 반드시 사용되는 것은 아닙니다. 인덱스를 통해 읽어야할 데이터가 일정량을 넘는 순간 옵티마이저는 인덱스 대신 **Table Full Scan**을 사용하는데 이는 인덱스에서 확인한 ROWID로 테이블 접근시 **랜덤 액세스 방식**으로 인해 많은 디스크 I/O가 발생할 수 있기 때문입니다.

기본적으로 PostgreSQL은 인덱스가 생성되는 동안에도 **읽기 작업**을 허용하고 **쓰기 작업(Insert, Update, Delete)**은 인덱스 생성 까지 블록됩니다.

인덱스는 하나 (**단일 컬럼 인덱스**) 혹은 여러 컬럼 (**다중 컬럼 인덱스**)에 대해 설정 할 수 있습니다.

<br>
## Index 생성 및 삭제

```sql
CREATE INDEX INDEX1 ON issue(analysis_id, determinant)
```

issue 테이블에 analysis_id, determinant 컬럼을 기준으로 INDEX1을 생성합니다.

```sql
DROP INDEX INDEX1
```

INDEX1인 인덱스를 삭제합니다.

<br>
# B-tree Index

B-Tree는 동등 연산이나 범위 연산을 수행할 때 적합합니다. 생성시 인덱스 유형을 명시하지 않으면 디폴트로 설정됩니다.

구조 예시는 다음과 같습니다.

<img src="/assets/images/25/1.png"  width="750" height="400"/>

B-tree는 최상위에 노드인 **rootNode**, 중간 노드인 **non-leaf nodes** 최하위 노드인 **leaf nodes**로 구분되며 하나의 노드에서 파생되는 branch의 개수는 해당 노드에 포함된 key의 개수+1개입니다.

이진트리와 다르게 하나의 노드에 많은 값을 가질 수 있으며, 인덱스 키를 바탕으로 데이터가 **항상 정렬된 상태로 유지**됩니다. 또한 **균형 트리**이기 때문에 root로부터 leaf까지의 거리가 일정해 어떠한 값을 조회하더라도 같은 시간이 걸립니다.

위의 그림에서 key 8을 찾는 예시는 다음과 같습니다.
- 8은 root노드에 모든 key보다 크기 때문에 가장 오른쪽 포인터를 탑니다.
- 첫번째 key인 8과 같기 때문에 왼쪽 포인터를 탑니다.
- 6보다는 값이 크기 때문에 오른쪽 포인터를 탑니다.
- leaf노드에 도달했기 때문에 해당 leaf노드부터 오른쪽으로 8보다 큰 값이 나올 때 까지 탐색합니다.

<br>
# Index  생성 시 유의 사항

<br>
### 인덱스 생성 기준

한 테이블에 너무 많은 인덱스를 설정하게되면 새로운 row가 저장될때마다 인덱스 업데이트가 필요하기 때문에 성능상 문제가 발생할 수있습니다.

보통 한 테이블당 3 ~4 개 적당하며 update가 많이 일어나지 않고 조회 시 주로 사용되는 컬럼이 적합합니다.

<br>
### 인덱스 컬럼 구성 기준

단일 컬럼 인덱스의 경우 **카디널리티**가 가장 높은 컬럼을 선택하는 것이 유리합니다.

다중 컬럼 인덱스는 구성 컬럼의 순서는 **카디널리티**가 높은 순에서 낮은 순으로 구성하는 것이 효율적입니다. **카디널리티**는 특정 컬럼 기준 중복도가 낮으면 높아집니다.

<br>
# Index 사용 시 유의 사항

<br>
### 조회 시 `WHERE`절을 사용
`WHERE` 을 통해 **인덱스가 걸린 컬럼을 조회**해야 성능 향상을 기대할 수 있기 때문에 `WHERE`절을 통해 실제 쿼리가 구성되는 컬럼을 선정해야 합니다.

`AND` 연산자는 읽어와야 할 row수를 줄이지만 `OR` 연산자는 비교해야 할 row 수가 늘어나기 때문에 Table 풀 스캔이 발생할 확률이 높기 때문에 주의가 필요합니다.

<br>
### 다중 컬럼 인덱스를 사용시 인덱스 가장 왼쪽 컬럼 `WHERE`절 사용

다중 컬럼 인덱스를 사용할 때는 **인덱스로 설정한 제일 왼쪽 컬럼**이 `WHERE` 절에 사용되야 합니다.

analysis_id, determinant 컬럼 기준으로 인덱스를 생성했을 때

```sql
CREATE INDEX INDEX1 ON issue(analysis_id, determinant)
```

첫번째 쿼리는 인덱스가 적용되지만 두번째 쿼리는 인덱스가 적용되지 않습니다.

```sql
SELECT * FROM issue i WHERE i.analysis_id= 1674  // 인덱스 적용O
SELECT * FROM issue i WHERE i.determinant= 'test'  //인덱스 적용X
```

<br>
### 문자열 패턴 와일드 카드 위치

문자열 패턴 매칭의 경우 문자열 시작에 와일드 카드가 위치해 있으면 인덱스가 적용되지 않습니다.

```sql
SELECT * FROM table WHERE name LIKE 'T%';  // 인덱스 적용O

SELECT * FROM table WHERE name LIKE '%T';  // 인덱스 적용X
```

<br>
# Index 적용 테스트

테스트에 사용된 issue 테이블은 다음과 같으며 약 200만개 row를 대상으로 테스트를 진행해보겠습니다.

```sql
CREATE TABLE "issue" (
	"id" BIGINT ,
	"analysis_id" BIGINT,
	"determinant" VARCHAR(1024) ,
	PRIMARY KEY ("id")
);
```

처음에는 **Index 생성 없이** 쿼리 플랜을 통해 어떤 방식으로 쿼리를 수행할지 확인해보겠습니다.

```sql
EXPLAIN analyze
SELECT * FROM issue i
WHERE i.analysis_id= 1674
AND i.determinant = 'test'
```

<img src="/assets/images/25/2.png"  width="700" height="220"/>

**Sequential Scan**의 경우 모든 데이터를 하나씩 확인 하는 것을 의미합니다. 현재 analysisId와 determinant에는 인덱스가 걸려있지 않기 때문에 Worker 프로세스 2개를 이용해 병렬적으로 테이블을 순차 스캔했음을 확인 할 수 있었습니다.

**Rows Removed by Filter**는 테이블의 row를 조회했으나 조건에 맞지 않아 버려진 row수를 의미합니다. 루프 한번당 755,450개를 버리고 세번의 루프를 돌았으니 한개 row를 찾기위해 약 200만개 row를 모두 조회한셈입니다.

마지막 **Execution Time**에서 실제 쿼리 실행에 소요된 시간을 확인 할 수있습니다.

<br>
이제 analysisId와 determinant 컬럼에 index를 걸어보겠습니다.

```sql
CREATE INDEX INDEX1 ON issue(analysis_id, determinant)
```

INDEX1로 인덱스를 생성했고 같은 조회 쿼리 동작을 쿼리 플랜을 통해 확인해보겠습니다.

```sql
EXPLAIN analyze
SELECT * FROM issue i
WHERE i.analysis_id= 1674
AND i.determinant = 'test'
```

<img src="/assets/images/25/3.png"  width="750" height="115"/>

쿼리 플랜을 통해 Index Scan이 이루어진 것을 확인 할 수 있고 실행 시간 또한 0.027ms로 대폭 감소된 것을 확인 할 수 있습니다.

<br>
추가로 **다중 컬럼 인덱스의 경우 제일 왼쪽 컬럼을 조회 시 사용**해야 인덱스를 타는데 이를 테스트로 통해 확인해보겠습니다. 가장 왼쪽 analysis_Id 컬럼을 `WHRER` 절에 포함 시키지 않고 쿼리를 날려보겠습니다.

```sql
EXPLAIN analyze
SELECT * FROM issue i
WHERE i.determinant = 'test'
```

<img src="/assets/images/25/4.png"  width="700" height="220"/>

예상대로 인덱스가 사용되지않고 **Sequential Scan으로** 모든 데이터를 하나씩 확인 한 것을 확인 할 수있습니다.

반대로 analysis_Id만 `WHERE` 절에 포함시켜 쿼리를 날려보겠습니다.

```sql
EXPLAIN analyze
SELECT * FROM issue i
WHERE i.analysis_id= 1674
```

<img src="/assets/images/25/5.png"  width="750" height="115"/>

위와 같이 Index Scan이 동작한 것을 확인 할 수있습니다.

추가로 인덱스를 통해 많은 데이터를 조회해보도록 하겠습니다. 여기서는 `<` 를 사용해서 약 백만개 ROW를 조회했습니다.

```sql
EXPLAIN ANALYZE
SELECT * from issue i WHERE analysis_id < 1600
```

<img src="/assets/images/25/6.png"  width="650" height="115"/>

조회되는 데이터가 일정량을 넘게되면 옵티마이저는 인덱스가 아닌 테이블 풀 스캔을 사용하는 것을 확인 할 수있었습니다.

---

## Reference
<https://www.postgresql.org/docs/current/indexes-intro.html>
