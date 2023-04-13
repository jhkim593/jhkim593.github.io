---
layout: post
title: "@Transactional 동작 원리"
author: "jhkim593"
tags: Spring

---
Spring에서 트랜잭션 처리를 위해 제공해주는 @Transactional 어노테이션을 프로젝트를 진행하면서 자주 사용하는데 사용하다 보면 예상과 다르게 동작하는 부분이 많았습니다. 그래서 이번장에서는 @Transactional이 어떻게 동작하는지 자세하게 확인해보도록 하겠습니다.

>관련 코드는 [github](https://github.com/jhkim593/blog_code/tree/master/jpa_entity_query)를 참고해주세요

## @Transactional 이란?
비즈니스 로직이 트랜잭션 처리를 필요로 할 때 트랜잭션 처리 코드가 비즈니스 코드와 같이 섞여있다면 코드 중복이 발생하고 비즈니스 핵심 로직에 집중하기 힘들어 질수 있습니다. 부가 기능인 트랜잭션 처리 분리를 위해 Spring에서는 **@Transactional** 어노테이션을 제공하는데 큰 설정없이 해당 어노테이션을 명시해주기만 하면 AOP를 통해 내부적으로 트랜잭션 코드가 수행됩니다.
<br>
<br>
클래스 , 메소드단 모두 @Transactional 선언이 가능하며 메소드 레벨의 @Transactional이 우선 순위를 갖습니다.

<br>
## @Transactional은 내부적으로 프록시 객체를 통해 동작한다.

예시로 Item Entity가 있고 아래와 같이 Item 생성을 위한 서비스 클래스가 있습니다.

Entitiy
~~~java
@Entity
@Getter @Setter
public class Item {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "member_id")
    private Long id;

    private String name;

    private int count;

    private String code;

    private String status;

}
~~~
<br>
ItemService
~~~java
@Service
@RequiredArgsConstructor
public class ItemService {
    private final ItemRepository itemRepository;

    /**
     * DB insert 메소드
     **/
    @Transactional
    public void insertItem(Item item) {                           //1
        item.setStatus("CREATED");
        itemRepository.save(item);
    }

    public void createItem1() {                                  //2
        for (int i = 0; i < 4; i++) {
            Item item = new Item();
            item.setName("item"+i);
            item.setCount(10);
            item.setCode("test");

            insertItem(item);
        }
    }

    @Transactional
    public void createItem2() {                                 //3
        for (int i = 0; i < 4; i++) {
            Item item = new Item();
            item.setName("item"+i);
            item.setCount(10);
            item.setCode("test");

            insertItem(item);
        }
    }
}
~~~
1. insertItem 메소드는 item를 insert 하기위한 메소드이고 @Transactional이 적용되어 있습니다.
2. createItem1 메소드는 insertItem 메소드를 4번 호출하고 @Transactional이 적용되어 있지않습니다.
3. createItem2 메소드는 insertItem 메소드를 4번 호출하고 @Transactional이 적용되어 있습니다.

createItem1 , createItem2 메소드는 정상 동작하게 될 시 모두 Item을 4개 insert 할 것입니다.

그러면 여기서 Item 4개를 모두 insert 한 뒤 **예외**가 발생했다고 가정해보겠습니다.

이 경우 insertItem 메소드에는 @Transactional이 적용되어 있기 때문에 두 메소드 모두 Item 4개가 정상적으로 생성될 것이라 예상됩니다.
해당 상황을 쿠폰이 모두 insert 된 후 **RuntimeException**을 던져 테스트를 진행해보겠습니다.

~~~java

public void createItem1() {                                  
    for (int i = 0; i < 4; i++) {
        Item item = new Item();
        item.setName("item"+i);
        item.setCount(10);
        item.setCode("test");

        insertItem(item);
    }
    throw new RuntimeException("insert exception");      //추가
}

@Transactional
public void createItem2() {                                 
    for (int i = 0; i < 4; i++) {
        Item item = new Item();
        item.setName("item"+i);
        item.setCount(10);
        item.setCode("test");

        insertItem(item);
    }
    throw new RuntimeException("insert exception");     //추가
}
~~~

아래의 테스트 코드를 통해 각 메소드 수행 후 insert된 Item 수가 4개가 맞는지 확인해보겠습니다.
~~~java
@SpringBootTest
class ItemServiceTest {

    @Autowired
    ItemService itemService;

    @Autowired
    ItemRepository itemRepository;

    /**
     * createItem1 메소드 테스트
     **/
    @Test
    public void createItem1() {

        assertThatThrownBy(() -> itemService.createItem1()).isInstanceOf(RuntimeException.class).hasMessage("insert exception");
        assertThat(itemRepository.findAll().size()).isEqualTo(4);
    }

    /**
     * createItem2 메소드 테스트
     **/
    @Test
    public void createItem2() {
        assertThatThrownBy(() -> itemService.createItem2()).isInstanceOf(RuntimeException.class).hasMessage("insert exception");
        assertThat(itemRepository.findAll().size()).isEqualTo(4);

    }
}
~~~

첫번째 테스트의 경우 예상대로 성공하고 DB를 확인해보면 Item 4개가 insert 된 것을 확인할 수 있습니다.
<img src="https://user-images.githubusercontent.com/53510936/231760545-bdd0ee1a-c92b-4c75-bcff-d90e444bb9d4.png"  width="600" height="200"/>

<br>
하지만 두번째 테스트의 경우 테스트는 실패하게 되고 DB를 확인해보면 Item이 insert되지않은 것을 확인할 수 있습니다.
<img src="https://user-images.githubusercontent.com/53510936/231738724-c0783683-3855-400b-b89b-cfd43754cc7b.png
"  width="600" height="200"/>

분명 **insertItem** 메소드에 @Trasnsactinal이 설정되어있고 예상대로 작동했다면 예외가 터져도 DB는 롤백되지 않아야 할 것입니다.

이와 같이 동작한 이유는 @Transactional이  **프록시**로 동작하기 때문입니다.


테스트 메소드에 break point를 찍어서 itemService 객체를 살펴보겠습니다.
<img src="https://user-images.githubusercontent.com/53510936/231752100-edec3231-0843-4637-b286-9846a1ed5ae3.png
"  width="600" height="100"/>
위와 같이 itemService 실제 객체가 아닌 CGLIB를 통해 프록시 객체가 생성돼 사용되고 있는 것을 확인할 수 있습니다.

생성된 프록시 객체는 다음과 같이 구성되어 있을 것이라고 예상할 수 있습니다.
~~~java
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

import javax.persistence.EntityManager;
import javax.persistence.EntityTransaction;

@Service
@RequiredArgsConstructor
public class ItemServiceProxy extends ItemService {
    private final EntityManager em;

    public void insertItem() {
        EntityTransaction tx = em.getTransaction();
        tx.begin();

        super.insertItem();

        tx.commit();
    }

    public void createItem1() {
        super.createItem1();
    }

    public void createItem2(int index) {
        EntityTransaction tx = em.getTransaction();
        tx.begin();

        super.createItem2(index);

        tx.commit();
    }
}
~~~
프록시 객체는 ItemService를 상속받고 @Transactional 선언된 메소드에는 트랜잭션 처리를 위한 부가 기능이 실제 객체 메소드 호출사이에 포함되어 있을 것입니다.


위 내용을 토대로 몇가지 특이사항에 대해 알 수 있습니다.

<br>
#### 1. private 메소드는  @Transactional이 적용되지 않음
@Transactional은 프록시 형태로 동작하기 때문에 외부에서 접근 가능한 메소드만 @Transactional 설정이 가능합니다.

<br>
#### 2. 같은 클래스 내에서 다른 메소드로 @Transactional이 호출되어도 최초 트랜잭션을 기준으로 동작합니다.
위 예시에서 createItem2 메소드가 예상대로 작동하지 않았는데
~~~java
public void createItem2(int index){
    EntityTransaction tx = em.getTransaction();
    tx.begin();

    super.createItem2(index);

    tx.commit();    
}
~~~
위와 같이 트랜잭션이 걸린 상태로 실제 객체의 createItem2를 호출하고 내부적으로 insertItem이 호출되기 때문에 상위 메소드인 createItem2에 @Transactional만 동작하게 됩니다.

<br>

#### 3. @Transactional이 선언되지않은 메소드가 @Transactional이 선언된 메소드를 호출하더라도 트랜잭션 적용이 되지 않는다.
프록시에서 내부에서 실제 객체를 호출하기 때문에 진입 메소드에 @Transactional이 선언되어있지 않으면 트랜잭션이 적용 되지 않습니다


## @Transactional 전파 레벨
1. Propagation.REQUIRED (기본 값)
~~~java
@Transactional(propagation = Propagation.REQUIRED)
public void test() { ... }
~~~
부모 메소드 (자식을 호출)에 이미 트랜잭션이 적용되어 있다면 기존 트랜잭션을 내에서 로직을 실행하며 트랜잭션이 적용되어 있지 않다면 새로운 트랜잭션을 생성합니다.
부모 메소드에서 예외가 발생하게되면 롤백 처리되고 자식으로도 롤백이 전파됩니다. 마찬가지로 자식 메소드에서 예외가 발생하면 롤백 처리되고 부모 메소드로 롤백이 전파됩니다.
해당 설정은 명시하지 않아도 **default**로 적용되어 있습니다.

2. Propagation.REQUIRES_NEW
~~~java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void test() { ... }
~~~
Propagation.REQUIRES_NEW로 설정되었을 때에는 매번 새로운 트랜잭션을 시작합니다.  부모 메소드에 이미 트랜잭션이 설정되어 있다면 기존의 트랜잭션은 메소드가 종료할 때까지 대기 상태가 되고 자식의 트랜잭션을 생성해 실행합니다. 각 트랜잭션은 완전히 독립적인 단위로 작동하기 때문에 예외가 발생해도 호출한 곳에는 롤백이 전파되지 않습니다.









동작예시
