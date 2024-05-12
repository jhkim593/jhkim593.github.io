---
layout: post
title: "PostgreSQL - bulk insert"
author: "jhkim593"
tags: PostgreSQL
---
이번장에서는 많은 데이터를 한번에 DB에 저장할 때 bulk insert를 적용해 성능 개선을 했던 내용을 다뤄보도록 하겠습니다.

<br>
## 문제 상황

DB로 **PostgreSQL 12.14버전**을 사용했으며 insert 대상 엔티티는 다음과 같습니다.

#### Issue entity

```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
@DiscriminatorColumn(name = "TYPE")
public abstract class Issue {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
}
```
#### DetailIssue enttiy

```java
@Entity
public class DetailIssue extends Issue {
...
}
```

Issue 엔티티와 Detail_Issue엔티티는 **상속 관계**이며 `InheritanceType.JOINED` 으로 Detail_Issue 테이블 pk는 Issue 테이블 pk를 참조하고 있는 상태입니다.

문제는 DB에 데이터 저장 시 발생했습니다. 특정 시점에 jpa `saveAll` 메소드로  Issue 테이블과 Detail Issue 테이블에  각각 1만개 가량의 데이터를 insert 하고 있었는데 완료되는 시간이 평균 7초정도로 예상보다 오래걸리고 있었습니다.



실행 쿼리 로그를 살펴보니 데이터 한건마다 insert 쿼리가 발생하고 있었고 각 테이블에 1만개씩 **총 2만개의 insert 쿼리**가 발생하고 있었습니다. `saveAll` 메소드를 사용하게 되면 한 트랜잭션내에서 작업이 이루어지지만 매건 마다 insert 쿼리가 발생하기 때문에 대량에 데이터를 넣는 상황에서 성능이 좋지 못했습니다. 그래서 insert시 성능 개선을 위해 **bulk insert**를 고려하게 되었습니다.

<br>
## bulk insert

일반적인 insert 쿼리와 bulk insert 쿼리에 대해 살펴보도록 하겠습니다.

#### insert 쿼리

```java
insert into issue (analysis_id ,risk) values (1 , 1)
insert into issue (analysis_id ,risk) values (1 , 2)
insert into issue (analysis_id ,risk) values (1 , 3)
```

#### bulk insert 쿼리

```java
insert into issue (analysis_id ,risk) values (1 , 1) , (1, 2) , (1,3)
```

기존 insert 쿼리는 매건 마다 쿼리를 날려 데이터를 저장하는 반면에
**bulk insert**는 insert 쿼리를 한번만 날리기 때문에 한번에 서버 접근으로 여러 데이터를 저장할 수 있어 많은 데이터를 저장할 때 성능이 크게 향상됩니다.

<br>
## 테이블 기본키 생성 전략 변경

jpa를 사용하게되면 테이블 기본키 생성 전략으로 보통 `IDENTITY` 방식을 사용합니다. 하지만 해당 방식은 id를 받아오기 위해서 insert를 해야하기 때문에 hibernate 단에서 bulk insert를 지원하지 않습니다.

bulk insert를 사용하기 위해서는 `TABLE` 또는 `SEQUENCE` 방식으로 **기본키 생성 전략을 변경**해야 합니다.

아래는 `SEQUENCE` 방식을 통한 bulk insert 설정 예시입니다.

<br>
#### entity 어노테이션 변경 예시
```java
 @Id
 @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "sequence")
 @SequenceGenerator(name = "sequence", sequenceName = "sequence_generator", allocationSize = 100)
 private Long id;
```
`GenerationType.SEQUENCE` 으로 설정했으며 `allocationSize`는 한번에 받아오는 시퀀스 크기를 나타냅니다.

#### application.yml 예시
```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc.batch_size: 1000
```
`jdbc.batch_size` 로 최대 몇 개의 statement를 묶어서 한번의 쿼리로 처리할지 설정 할 수 있습니다. 예를 들어 jpa `saveAll` 메소드를 사용했을 때 `jdbc.batch_size` 에 설정된 수 만큼 insert 쿼리가 bulk 형식으로 실행됩니다.

해당 값을 설정하지 않았을 때는 매건 마다 insert 쿼리가 발생하기 때문에 bulk insert를 사용을 위해 반드시 설정해야합니다.

또한 PostgreSQL의 경우 bulkinsert를 위해 DB url 쿼리 스트링에 `rewriteBatchedInserts=true` 설정을 반드시 추가해야합니다. 아래 application.yml 예시 중 하나만 설정하면 됩니다.

```yaml
datasource:
    url: 'jdbc:postgresql://localhost:5432/test?rewriteBatchedInserts=true'
-----------------
datasource:
    hikari:
      data-source-properties:
        rewriteBatchedInserts: true
```

하지만 이 방식은 기존에 사용중인 테이블 변경 및 Sequence를 새로 생성해야 하고 batch size 또한 설정해야하기 때문에 기존 설정 변경이 많이 필요하다는 불편함이 있습니다.

<br>
## jdbc 사용을 통한 bulk insert

jdbc를 직접 사용해 bulk insert를 구현 할 수도있습니다. 쿼리를 직접 작성해서 관리해야 한다는 불편함이 있지만 **기존 설정을 변경하지 않고도 bulk insert 적용이 가능**하기 때문에 이 방식을 사용했습니다.

예시 코드는 다음과 같습니다.

이 방식도 동일하게 DB url 쿼리 스트링에 `rewriteBatchedInserts=true` 설정을 반드시 추가해야 합니다.

```yaml
datasource:
    url: 'jdbc:postgresql://localhost:5432/test?rewriteBatchedInserts=true'
-----------------
datasource:
    hikari:
      data-source-properties:
        rewriteBatchedInserts: true
```

BulkinsertRepository 예시
```java
@Transactional
public void saveAll(List<DetailIssue> issues) throws Exception {
    try {
        StringBuilder issueSb = new StringBuilder();
        issueSb.append("INSERT INTO ISSUE (col1,col2,col3,col4,col5) VALUES ");
        for (int i=0; i<issues.size();i++){
            issueSb.append("(?,?,?,?,?)");
            if(i!= issues.size()-1){
                issueSb.append(",");
            }
        }
        issueSb.append(" returning id");
        List<Long> issueIds = new ArrayList<>();
        try (
            PreparedStatement issuePs = DataSourceUtils.getConnection(dataSource).prepareStatement(issueSb.toString())) {
            int idx = 1;
            for (Issue issue : issues) {
                issuePs.setLong(idx++, issue.getCol1());
                issuePs.setString(idx++, issue.getCol2());
                issuePs.setString(idx++, issue.getCol3());
                issuePs.setInt(idx++, issue.getCol4());
                issuePs.setString(idx++, issue.getCol5());
            }

            try(ResultSet rs = issuePs.executeQuery()){
                while (rs.next()) {
                    issueIds.add(rs.getLong(1));
                }
            }
        }
        String detailIssueSql =
            "INSERT INTO DETAIL_ISSUE (col1,col2,col3,col4,col5,col6,col7,id) " +
                "VALUES (?,?,?,?,?,?,?,?)";
        try (
            PreparedStatement issuePs = DataSourceUtils.getConnection(dataSource).prepareStatement(detailIssueSql)) {
            for (int i = 0; i < issues.size(); i++) {
                DetailIssue issue = detailIssues.get(i);
                issuePs.setString(1, issue.getDetailCol1());
                issuePs.setString(2, issue.getDetailCol2());
                issuePs.setInt(3, issue.getDetailCol3());
                issuePs.setString(4, issue.getDetailCol4());
                issuePs.setInt(5, issue.getDetailCol5());
                issuePs.setString(6, issue.getDetailCol6());
                issuePs.setInt(7, issue.getDetailCol7());
                issuePs.setLong(8, issueIds.get(i));
                issuePs.addBatch();
                issuePs.clearParameters();
            }
            issuePs.executeBatch();
        }
    } catch (Exception e){
        throw e;
    }
}
```
단일 테이블 insert 라면 `JdbcTemplate.batchUpdate()` 와 같은 메소드를 사용하시면 더 쉽게 구현이 가능하지만 **detail_issue 테이블 pk 값은 issue pk 값을 참조**하기 때문에 issue 테이블을 먼저 insert 해서 pk 값을 받아와야합니다.

동작 순서는 다음과 같습니다.

- issue 테이블에 먼저 **bulk insert** 하는데 쿼리 가장 마지막에 `returning id`를 추가해 id를 리턴받습니다.
- ResultSet에 `getLong(1)`메소드를 통해 리턴받은 id를 issueIds 리스트에 저장합니다. ResultSet에서 리턴 받은 id는 insert 된 row 순서대로 받기 때문에 항상 순서가 보장됩니다.
- issueIds 저장된 id를 순서대로 Detail_issue 테이블 id 컬럼에 입력해 **bulk insert** 합니다.

<br>
### 쿼리 로그 확인

PostgreSQL 쿼리 로그를 살펴보면 bulk insert로 데이터가 저장 된것을 확인 할 수 있습니다.

<img src="/assets/images/26/1.png"  width="750" height="30"/>

<img src="/assets/images/26/2.png"  width="750" height="30"/>

<br>
### 실행 시간 비교

**기존**

<img src="/assets/images/26/3.png"  width="750" height="30"/>

**bulk insert**  

<img src="/assets/images/26/4.png"  width="800" height="50"/>

실제 각 테이블에 1만건씩 총 2만건에 데이터를 insert 했을 때 기존 로직은 6.54초가 걸렸던 반면에 bulk insert로 쿼리 개선 후 시간은 1.59초로 약 3배 가량 빨라진 것을 확인했습니다.
