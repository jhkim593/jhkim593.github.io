---
layout: post
title: "JPA Entity 조회만 했는데 update가 발생한 이유"
author: "jhkim593"
tags: JPA
---
프로젝트에서 JPA를 통해 데이터를 조회하는 로직이 있었는데 조회후 데이터를 수정하지 않았는데 **update 쿼리**가 날라가는 경우가 있었습니다. 예상과 다른 결과였기 때문에 어떤 문제였고 어떻게 해결했는지 다뤄보았습니다.

<br>
## 문제 상황

엔티티 및 서비스 코드는 다음과 같습니다.

### Request

```java
@Entity
@Getter
@Setter
public class Request {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    private String type;

    @Convert(converter = RequestTextConverter.class)
    @Column(columnDefinition="TEXT")
    private RequestText requestText;

    @Column(updatable = false)
    @CreationTimestamp
    private Timestamp insertTime;


    @Getter
    @Setter
    public static class RequestText{
        private String type;
        private String target;
    }
}
```

### RequestTextConverter

```java

public class RequestTextConverter implements AttributeConverter<RequestText, String> {

    private ObjectMapper objectMapper = new ObjectMapper();

    @Override
    public String convertToDatabaseColumn(RequestText requestDto) {
        if (requestDto != null) {
            try {
                return objectMapper.writeValueAsString(requestDto);
            } catch (JsonProcessingException e) {
                return null;
            }
        } else {
            return null;
        }
    }

    @Override
    public RequestText convertToEntityAttribute(String str) {
        if (str != null) {
            try {
                return objectMapper.readValue(str, new TypeReference<RequestText>() {});
            } catch (JsonProcessingException jpe) {
                return null;
            }
        } else {
            return null;
        }
    }
}

```

### RequestService

```java
@Service
@RequiredArgsConstructor
public class RequestService {
    private final RequestRepository requestRepository;

    @Transactional
    public Request find(Long id){
        return requestRepository.findById(id).orElse(null);
    }
}
```

Request는 id , name , type , requestText , insertTime 으로 구성된 엔티티입니다. reuqestText는 @Convert 어노테이션을 사용해서 DB 저장 , 조회시 데이터를 변환하고 있었습니다.

여기서  RequestService에 find 메소드를 호출하게되면

<img src="/assets/images/22/1.png"  width="400" height="500"/>

트랜잭션이 종료 후 DB에 select 쿼리와 **update 쿼리**가 같이 날라간 것을 확인 할 수 있습니다. 서비스 로직을 보면 조회만 하고 데이터를 수정하는 로직은 없었기 때문에 예상과 다른 동작이었습니다.

<br>
## 문제 파악

문제를 파악해보기위해 Request 엔티티에 `@DynamicUpdate` 어노테이션을 붙여 다시 테스트해보았습니다. `@DynamicUpdate`을 붙이게되면 수정된 필드에 대해서만 **update 쿼리**가 나가기 때문에 어떤 필드가 수정되고 있는지 알 수있었습니다.

```java
@Entity
@Getter
@Setter
@DynamicUpdate
public class Request {
	...
}
```

추가후 테스트를 해보았습니다.
<img src="/assets/images/22/2.png"  width="400" height="450"/>

쿼리를 살펴보니 requestText 필드가 변경돼 **더티체킹**으로 인해 update 쿼리가 발생했다는 것을 알 수있었습니다. 데이터를 따로 변경하지 않았는데도 **더티체킹**이 발생한 이유는 다음과 같습니다.

JPA 구현체인 **Hibernate**는 트랜잭션내에서 엔티티 조회시 엔티티의 **복사본**을 생성합니다. 그리고 트랜잭션이 끝나는 시점에 복사본과 조회한 엔티티를 비교해서 update 쿼리를 날리는데 이 때 Objects.equals를 사용해 모든 필드가 전부 같은지 비교합니다.

Objects.equals는 해당 객체의 equals 메소드를 호출하는데 오버라이딩 하지않으면 **레퍼런스**를 비교하게됩니다. **레퍼런스**를 비교하게되면 같은 값을 가지고 있는 객체여도 새롭게 생성된 객체와 비교시 false를 반환하게됩니다. String , Long 등은 동등성 검사를 위해 내부적으로 equals를 재정의해서 사용하고 있습니다.

 RequestText에는 equals가 오버라이딩 되어있지 않았고 Request 엔티티 조회시 생성된 복사본과 조회된 엔티티간 레퍼런스가 달랐기 때문에 **더티체킹**으로 update가 발생한 것이었습니다.


<br>
## 문제 해결

RequestText에 equals 오버라이딩을 위해 `@EqualsAndHashCode` 을 추가했습니다. `@EqualsAndHashCode` 는 객체에 포함된 필드를 각각 비교할 수 있도록 equasl , hashcode 메소드를 재정의 해줍니다.

```java
@Getter
@Setter
@EqualsAndHashCode
public static class RequestText{
    private String type;
    private String target;
}
```

추가후 다시 테스트를 해보면

<img src="/assets/images/22/3.png"  width="400" height="400"/>

RequestText의 레퍼런스가 아닌 필드 값을 비교하므로 update쿼리가 발생하지 않는것을 확인 할 수있었습니다. `@Convert` 와 같은 어노테이션을 사용시 equals 재정의 하지않으면 **더티체킹**이 발생하기 때문에 이부분을 유의해야 할 것같습니다.
