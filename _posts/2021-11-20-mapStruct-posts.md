---
layout: post
title: "MapStruct를 이용해 Entity <-> Dto 변환"
author: "jhkim593"
tags: JPA

---

## MapStruct
MapStruct는 Dto < -- > Entity 간 객체 Mapping을 편하게 도와주는 라이브러리이며 컴파일 시점에 구현체를 만들어내기 떄문에 리플렉션이 발생하지 않습니다. 매핑해줄 클래스(target)에는 setter가 있어야 하고 매핑이 되는 클래스(source)에는 getter가 있어야 사용 가능합니다.

### ModelMapper , MapStruct
MapStruct가 ModelMapper보다 좋은 이점은 다음과 같은 것들이 있습니다.
- 매핑 속도가 빠름
- 명시적인 변수 매핑
- 컴파일 단계에서 에러 확인 가능


### 의존성 추가
~~~gradle
implementation "org.mapstruct:mapstruct:${mapstructVersion}"
	compileOnly 'org.projectlombok:lombok-mapstruct-binding:0.2.0'
	compileOnly 'org.projectlombok:lombok'
	annotationProcessor 'org.projectlombok:lombok'
	annotationProcessor "org.mapstruct:mapstruct-processor:${mapstructVersion}"
~~~
Lombok 사용시 MapStruct가 먼저오게 작성해야 에러가 발생하지 않습니다.
Entity와 Dto를 생성해서 MapStruct를 사용해보겠습니다.

<br>

### Entity , Dto 생성
~~~java
@Getter
@Setter
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    private String orderNum;

    @ManyToOne
    @JoinColumn(name = "person_id")
    Person person;

    public void addPerson(Person person){
        this.person=person;
        person.getOrders().add(this);
    }
}

~~~

~~~java
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
public class OrderDto {

    private String name;

    private String orderId;

}

~~~
간단하게 주문정보에 대한 Entity와 Dto를 생성했으며 ,
OrderDto --> Order 변환시 orderNum값을 orderId로 받을 것이며 id값은 설정하지 않습니다.
Order--> OrderDto 변환시 orderId값을 orderNum값으로 받도록 할 것입니다.

<br>

~~~java
@Getter
@Setter
@Entity
public class Person {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    private String birth;

    private String email;

    private String group;

    @OneToMany(mappedBy = "person")
    Set<Order> orders=new HashSet<>();


}
~~~


~~~java
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
public class PersonDto {

    private String name;

    private String email;

    private String birth;

    private String team;

    private LocalDateTime localDateTime;

    private Set<OrderDto>orders=new HashSet<>();
}
~~~
주문자 정보를 위한 Entity와 Dto를 생성했으며 ,
Person --> PersonDto 변환시 team을 group으로 받을 것이며 localDateTime을 추가합니다.
PersonDto --> Person 변환시 group을 team으로 받을 것이며 id값은 생성하지 않습니다.

<br>

### Mapper 설정

~~~java
@Mapper
public interface PersonMapper {

    PersonMapper MAPPER = Mappers.getMapper(PersonMapper.class);  //1

    @Mappings({
            @Mapping(target = "id", ignore = true),        //2
            @Mapping(target = "group", source = "team")    //3
    })
    Person personDtoToPerson(PersonDto personDto);


    @Mappings({
            @Mapping(target = "localDateTime" ,expression = "java(java.time.LocalDateTime.now())"), //4
            @Mapping(target = "team", source = "group")  //5
    })
    PersonDto personToPersonDto(Person person);



    @Mappings({
            @Mapping(target = "orderNum", source = "orderId"), //6
            @Mapping(target = "id", ignore = true)   //7
    })
    Order orderDtoToOrder(OrderDto orderDto);


    @Mapping(target = "orderId", source = "orderNum") //8
    OrderDto orderToOrderDto(Order order);

}

~~~
1. 해당하는 MAPPER가 PersonMapper 상속받아서 메서드와 실제 로직들이 PersonMapperrImpl로 구현되는 것을 명시
2. PersonDto -- > Person 변환시 id값 제외
3. PersonDto -- > Person 변환시 PersonDto team을 Person group으로 매핑
4. Person --> PersonDto 변환시 localDateTime에 현재시간 매핑
5. Person --> PersonDto 변환시 Person group을 PersonDto team으로 매핑
6 , 7 ,8 위와 동일


<br>

~~~java
public interface GenericMapper<D, E> {
    D toDto(E entity);

    E toEntity(D dto);
}
~~~
~~~java
@Mapper
public interface DataMapper extends GenericMapper<DataDto, DataEntity> {
}
~~~
별다릉 임의 매핑이 필요없다면 , 공통 interface를 선언해두고 , 상속 받는다면 추가적인 코드를 작성하지 않아도 된다.

<br>

### MapperImpl 생성
프로젝트를 빌드 하게 되면 PersonMapperImpl가 생성되는 것을 확인할 수있습니다.

<br>

<img src="https://user-images.githubusercontent.com/53510936/142766525-0b8fc51a-11fc-40e2-a106-6d49092d6bd7.png"  width="600" height="240"/>
~~~java
public class PersonMapperImpl implements PersonMapper {
    public PersonMapperImpl() {
    }

    public Person personDtoToPerson(PersonDto personDto) {
        if (personDto == null) {
            return null;
        } else {
            Person person = new Person();
            person.setGroup(personDto.getTeam());
            person.setName(personDto.getName());
            person.setBirth(personDto.getBirth());
            person.setEmail(personDto.getEmail());
            person.setOrders(this.orderDtoSetToOrderSet(personDto.getOrders()));
            return person;
        }
    }

    public PersonDto personToPersonDto(Person person) {
        if (person == null) {
            return null;
        } else {
            PersonDto personDto = new PersonDto();
            personDto.setTeam(person.getGroup());
            personDto.setName(person.getName());
            personDto.setEmail(person.getEmail());
            personDto.setBirth(person.getBirth());
            personDto.setOrders(this.orderSetToOrderDtoSet(person.getOrders()));
            personDto.setLocalDateTime(LocalDateTime.now());
            return personDto;
        }
    }

    public Order orderDtoToOrder(OrderDto orderDto) {
        if (orderDto == null) {
            return null;
        } else {
            Order order = new Order();
            order.setOrderNum(orderDto.getOrderId());
            order.setName(orderDto.getName());
            return order;
        }
    }

    public OrderDto orderToOrderDto(Order order) {
        if (order == null) {
            return null;
        } else {
            OrderDto orderDto = new OrderDto();
            orderDto.setOrderId(order.getOrderNum());
            orderDto.setName(order.getName());
            return orderDto;
        }
    }

    protected Set<Order> orderDtoSetToOrderSet(Set<OrderDto> set) {
        if (set == null) {
            return null;
        } else {
            Set<Order> set1 = new HashSet(Math.max((int)((float)set.size() / 0.75F) + 1, 16));
            Iterator var3 = set.iterator();

            while(var3.hasNext()) {
                OrderDto orderDto = (OrderDto)var3.next();
                set1.add(this.orderDtoToOrder(orderDto));
            }

            return set1;
        }
    }

    protected Set<OrderDto> orderSetToOrderDtoSet(Set<Order> set) {
        if (set == null) {
            return null;
        } else {
            Set<OrderDto> set1 = new HashSet(Math.max((int)((float)set.size() / 0.75F) + 1, 16));
            Iterator var3 = set.iterator();

            while(var3.hasNext()) {
                Order order = (Order)var3.next();
                set1.add(this.orderToOrderDto(order));
            }

            return set1;
        }
    }
}

~~~

<br>

### Test

~~~java
@ExtendWith(SpringExtension.class)
public class MappingTest {


    @Test
    public void entityToDto()throws Exception{
        //given
        Person person=new Person();
        person.setName("name");
        person.setEmail("email");
        person.setBirth("birth");
        person.setGroup("team");
        person.setId(1L);

        Order order=new Order();
        order.setId(1L);
        order.setName("orderName");
        order.setOrderNum("orderNum");
        order.addPerson(person);

        //when
        PersonDto personDto = PersonMapper.MAPPER.personToPersonDto(person);

        //then
        assertThat(personDto.getEmail()).isEqualTo(person.getEmail());
        assertThat(personDto.getBirth()).isEqualTo(person.getBirth());
        assertThat(personDto.getTeam()).isEqualTo(person.getGroup());
        assertThat(personDto.getLocalDateTime()).isNotNull();
        for (OrderDto orderDto : personDto.getOrders()) {
            assertThat(orderDto.getName()).isEqualTo(order.getName());
            assertThat(orderDto.getOrderId()).isEqualTo(order.getOrderNum());

        }
    }

    @Test
    public void DtoToEntity()throws Exception{
         //given
        OrderDto orderDto=new OrderDto();
        orderDto.setName("orderName");
        orderDto.setOrderId("orderNum");

        PersonDto personDto=new PersonDto();
        personDto.setBirth("birth");
        personDto.setEmail("email");
        personDto.setTeam("team");
        personDto.setName("name");
        personDto.getOrders().add(orderDto);
        //when
        Person person = PersonMapper.MAPPER.personDtoToPerson(personDto);

        //then
        assertThat(person.getEmail()).isEqualTo(personDto.getEmail());
        assertThat(person.getBirth()).isEqualTo(personDto.getBirth());
        assertThat(person.getGroup()).isEqualTo(personDto.getTeam());
        assertThat(person.getId()).isNull();
        for (Order order : person.getOrders()) {
            assertThat(order.getName()).isEqualTo(orderDto.getName());
            assertThat(order.getOrderNum()).isEqualTo(orderDto.getOrderId());
            assertThat(order.getId()).isNull();
        }
    }
}

~~~
다음과 같이 Test code 작성시 모두 success 되는 것을 확인 할수 있습니다.
<br>
###ADD

~~~java
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
public class TestDto {

    private  name;

		private List<Long>itemIds;

		private Set<PersonDto>personDtos;


}
~~~

~~~java
@Getter
@Setter
@Entity
public class Test {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToMany(mappedBy = "test")
    Set<Item> items=new HashSet<>();

		@ManyToOne
		@JoinColumn(name = "person_id")
		Person person;

}
~~~

~~~java
@Getter
@Setter
@Entity
public class Item {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

		@ManyToOne
    @JoinColumn(name = "test_id")
    Test test;

		public Item(Long itemId){
			this.id=itemId;
		}

}
~~~


예시 Entity와 Dto를 다시 생성했습니다.

~~~java
@Mapper
public interface TestMapper {

    Test MAPPER = Mappers.getMapper(TestMapper.class);  
		Person PERSON_MAPPER=Mappers.getMapper(PersonMapper.class);

    @Mappings({
						@Mapping(target = "items", expression = "java(testDto.getItemIds() == null ? null : testDto.getItemIds().stream().map(v -> new  Item(v)).collect(Collectors.toSet()))"),   
						@Mapping(target= "person" , ignore="true")  
    })
    Test testDtoToTest(TestDto testDto , @Context CycleAvoidingMappingContext context);  //1

		@Mappings({
						@Mapping(target = "items", expression = "java(test.getPerson() == null ? null : PERSON_MAPPER.personToPersonDto(test.getPerson())") //2
		})
		TestDto testToTestDto(Test test);

	  void updateUserFromUserDto(TestDto userDto, @MappingTarget Test test);


}
~~~
1.변환시 무한 참조를 방지할 수 있다??
2.다른 mapper를 이용
