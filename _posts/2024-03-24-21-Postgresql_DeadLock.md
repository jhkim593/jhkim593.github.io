---
layout: post
title: "PostgreSQL - DeadLock"
author: "jhkim593"
tags: PostgreSQL
---
이번 장에서는 Spring boot , JPA , PostgreSQL를 사용 중인 프로젝트에서 DB 업데이트시 **DeadLock**이 발생하는 것을 확인해서 DeadLcok이 무엇을 의미하는지 , 어떤 상황이었는지, 문제 상황을 어떻게 해결했는지 다루도록 하겠습니다.

<br>
# Lock이란?

PostgreSQL는 테이블의 데이터에 대한 동시 액세스를 제어하기위해서 Lock 기능을 제공합니다.

동시성 제어를 통해 여러 트랜잭션이 작업 수행시 테이블의 데이터에 동시에 쓰기 , 수정, 삭제를 제한해 데이터 일관성을 유지 할 수있습니다.

여러 트랜잭션이 동시에 동일한 데이터를 조회 또는 수정하기 위해 Lock을 요청하게 되면 요청간 “충돌”이 발생하게 됩니다.

충돌한다는 의미는 두 트랙잭션이 동시에 잠금을 가질 수 없는 것을 의미합니다.

<br>
# 테이블 수준의 잠금

Lock에는 여러 타입이 있습니다. 여기서는 테이블 수준의 잠금 중 몇가지만 알아보도록 하겠습니다.

> <https://www.postgresql.org/docs/current/explicit-locking.html>
>

<br>
### **Access Share ( AccessShareLock )**

- `Access Exclusive` 를 제외하고 다른 모든 잠금과는 충돌하지 않습니다.
- 일반적으로 테이블을 읽기만 하고 수정하지 않는 쿼리가 획득합니다.
- 조회 진행 중 DDL로 요청을 동시에 수행하지 못하도록합니다.

`select` 쿼리를 날리게되면
```sql
begn;
select * from item;
```
`begin`명령어를 실행하면, 이후 실행하는 transaction으로 묶이게 되어 `commit` 또는 `rollback`을 하게 될 때 까지 DB에 반영되지 않습니다.

다음 해당 쿼리를 통해 Lcok을 확인해보면

```sql
select locktype, relation::regclass, mode, transactionid tid, pid, granted from pg_catalog.pg_locks order by pid;
```

`AccessShareLock`을 획득하는 것을 확인 할 수있습니다.
<img src="/assets/images/21/1.png"  width="700" height="150"/>

<br>
이 상태에서 만약 DDL 쿼리를 날리게된다면

```sql
begin;
alter table item drop column name;
```

<img src="/assets/images/21/2.png"  width="700" height="210"/>

AccessExclusiveLock granted가 Fasle로 조회 쿼리에서 얻은 잠금이 해제되지않아서 잠금을 획득하지 못하는 것을 확인 할 수있습니다.


<br>
### Row Exclusive ( **RowExclusiveLock** )

- `UPDATE`, `DELETE`, `INSERT` 등 테이블을 수정하는 명령에 의해 획득됩니다.

<br>
### Share Lock

- `ROW EXCLUSIVE`, `SHARE UPDATE EXCLUSIVE`, `SHARE ROW EXCLUSIVE`, `EXCLUSIVE` 및 `ACCESS EXCLUSIVE` 잠금 모드와 충돌합니다.
- 동시 데이터 변경시 테이블을 보호하기 위해 사용됩니다.

session1과 session2에서 순서대로 item_id 1에 상태를 변경 시키려고한다면

```sql
UPDATE item SET NAME='item1' WHERE item_id=1;
```

<img src="/assets/images/21/3.png"  width="700" height="310"/>

한 transaction이 `ShareLock`을 요청하는 것을 알 수있습니다. `ShareLock`은 동시에 데이터를 변경할 때 문제를 방지하기 위해 먼저 Lock을 획득한 transaction에게 공유를 요청하는 Lock입니다. session1에 transaction에서 `ExclusiveLock`을 획득했기때문에 `ShareLock`과 **충돌**하는 것을 확인 할 수있습니다.

이후 session1에 transaction이 종료되면 session2에 transaction이 작업을 수행하게됩니다.

<br>
# DeadLock

여러 트랜잭션이 서로 Lock을 얻기위해서 작업을 끝내지 못하고 **무한정 대기**하는 상태를 말합니다.

예를들어 한 **트랜잭션 1**이 테이블 A에 대한 Lock을 획득하고 테이블 B에 대한 Lock을 획득하려고 할 때 이미 테이블 B에 대한 Lock을 획득한 **트랜잭션 2**가 테이블 A에 대한 Lock을 획득하려는 경우 서로 Lock을 획득하기 위해 무한정 대기하게되는데 이것을 DeadLock 이라고합니다.

postgreSQL은 DeadLock이 발생하면 **자동으로 감지하고 관련된 트랜잭션 중 하나를 중단 후 다른 트랜잭션이 완료됟록 허용**하여 문제를 해결합니다. 하지만 어떤 트랜잭션이 종료될지는 모르기 때문에 이것에 의존해서는 안됩니다.

<br>
DeadLock 발생하는 상황을 의도적으로 만들어서 확인해보도록 하겠습니다.

session1에서 item_id 1에 대한 update 쿼리를 날립니다.

```sql
begin;
UPDATE item SET NAME='item1' WHERE item_id=1;
```

이후 session2에서 item_id 2에 대한 update 쿼리를 날립니다.

```sql
begin;
UPDATE item SET NAME='item2' WHERE item_id=2;
```

다음과 같은 결과를 얻을 수 있습니다.

<img src="/assets/images/21/4.png"  width="700" height="300"/>

데이터를 수정하려고 할 때 transactionId가 할당이 되는데 현재 **transaction ( 5082 )** 와 **transation (5083)** 이 item_id 1,2의 Lock을 획득 한것을 확인 할 수있습니다.

여기서 DeadLock을 발생 시키기 위해서는

- transaction( 5082 ) 에서 transaction( 5083 )이 가지고 있는 Lock을 요청
- transaction( 5083 ) 에서 transaction( 5082 )이 가지고 있는 Lock을 요청

서로 가지고 있는 Lock을 획득하려 한다면 DeadLock이 발생 할 것입니다.

session1 에서 다음 쿼리를

```sql
UPDATE item SET NAME='item2' WHERE item_id=2;
```

session2에서 다음 쿼리를 날리게되면

```sql
UPDATE item SET NAME='item1' WHERE item_id=1;
```

다음과 같이 DeadLock이 발생하는 것을 확인 할 수있으며

<img src="/assets/images/21/5.png"  width="400" height="250"/>

이 후 transaction (5083)이 강제 중단 된것을 확인 할 수있습니다.

<img src="/assets/images/21/6.png"  width="700" height="210"/>

<br>
# 문제 상황

<img src="/assets/images/21/7.png"  width="550" height="300"/>

백엔드에서 분석 모듈로 분석 요청을 보낸 뒤 분석 모듈이 백엔드로 **분석 결과**와 **분석 상태**를 **주기적으로 업데이트**하기 위한 API로 요청을 보내고 있습니다.

분석결과 , 분석 상태 업데이트 API 모두 **ANALYSIS** , **REQUEST** 테이블을 수정하는 로직이 포함되어있습니다.

예시 코드는 다음과 같습니다.

```java
//분석 결과 업데이트
@Transactional
public void resultUpdate(Long analysisId, Integer count) throws Exception {
        Analysis analysis = findByAnalysisId(analysisId);
        analysis.issueCount(count);
        analysisRepository.save(analysis);

        Request request = analysis.getRequest();
        request.setResult("ISSUE" + count);
        requestRepository.save(request);
    }

//분석 상태 업데이트
@Transactional
public void progressUpdate(Long analysisId, Integer progress) throws Exception {
        Analysis analysis = findByAnalysisId(analysisId);
        analysis.setProgress(progress);
        analysisRepository.save(analysis);

        Request request = analysis.getRequest();
        request.setResult("ING" + progress);
        requestRepository.save(request);
    }
```

한 트랜잭션이 ANALYSIS , REQUEST 테이블을 모두 수정하기 때문에 백엔드에서는 주기적으로 오는 요청들을 처리하던 중 **DeadLock**이 발생했습니다.

<br>
## 문제 해결

한 트랜잭션에서 ANALYSIS , REQUEST 테이블 수정을 모두 진행해 문제가 발생했기 때문에

```java
public void resultUpdate(Long analysisId, Integer count) throws Exception {
        analysisService.resultUpdate(analysisId, count);
        requestService.requestUpdate(requestId);
    }

//분석 상태 업데이트
public void progressUpdate(Long analysisId, Integer progress) throws Exception {
        analysisService.progressUpdate(analysisId, count);
        requestService.progressUpdate(requestId);
    }
```

테이블 업데이트 마다 Transaction을 분리해 **DeadLock** 문제를 해결 할 수있었습니다.


---

## Reference
<https://www.postgresql.org/docs/current/explicit-locking.html>
