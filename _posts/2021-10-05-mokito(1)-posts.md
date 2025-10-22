---
layout: post
title: "Mockito (1) - @Mock , @Spy , @InjectMocks"
author: "jhkim593"
tags: TEST

---
## UnitTest(단위 테스트)
단위 테스트는 응용 프로그램에서 테스트 가능한 가장 작은 소프트웨어를 실행하여 예상대로 동작하는지 확인하는 테스트이며 모든 함수와 메소드에 대한 테스트 케이스(Test case)를 작성하는 절차를 말합니다. 프로그래밍 언어마다 단위 테스트에서 사용하는 프레임워크가 다른데 **Java**는 주로 **JUnit**으로 테스트합니다.<br>
  <br>단위테스트 장점
  - 개발단계 초기에 문제를 발견하게 도와준다.
  - 기능에대한 불확실성을 감소시킬수 있다.
  - 시스템에 대한 실제문서를 제공한다.

이 글에서는 **Mockito**를 사용하여 단위테스트를 진행해보겠습니다.
<br><br>


## Mock
 주로 객체 지향 프로그래밍으로 개발한 프로그램을 테스트할 경우 테스트를 수행할 모듈과 연결되는 외부의 다른 서비스나 모듈들을 실제 사용하는 모듈을 사용하지 않고 실제의 모듈을 "흉내"내는 "가짜" 모듈을 작성하여 테스트의 효용성을 높이는 데 사용하는 객체입니다. 테스트시 DB를 읽어오지 않아 DB데이터에 의존하지 않고 ,테이블 접근을 최소화 할수 있어 테스트 시간을 줄이고 불필요한 리소스 소비를 막을 수있습니다.

## Mockito
 mock을 쉽게 만들고 mock의 행동을 정하는 stubbing, 정상적으로 작동하는지에 대한 verify 등 다양한 기능을 제공해주는 프레임워크입니다.



 테스트를 위해 UserService를 생성합니다.

~~~java
@Service
  public class UserService {
      public User getUser(){
          return new User(1L,"test");
      }
}
~~~

### @Mock

 **@Mock** 으로 만든 mock 객체는 가짜 객체이며 그 안에 메소드 호출해서 사용하려면 반드시 스터빙(stubbing)을 해야합니다.

~~~java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    UserService userService;

    @Test
    public void getUser(){
        assertThat(userService.getUser()).isNull();
    }
 }
~~~

 @Mock 을 사용하고 스터빙을 하지 않아 레퍼런스 타입은 null이 반환됩니다.
 <br><br>

~~~java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    UserService userService;

    @Test
    public void getUserWithStubbing(){
        User dummyUser=new User(2L,"test");
        when(userService.getUser()).thenReturn(dummyUser);
        User user = userService.getUser();
        assertThat(user.getId()).isEqualTo(2L);
    }
}
~~~

스터빙(stubbing)
만들어진 mock 객체의 메소드를 실행했을 때 어떤 리턴 값을 리턴할지를 정의하는 것 이며 Mockito에서는 when메소드를 사용해서 스터빙을 지원하고 있습니다.
<br><br>


### @Spy <br>
@Spy로 만든 mock객체는 진짜 객체이며 메소드 실행 시 스터빙을 하지 않으면 기존 객체의 로직을 실행한 값을 , 스터빙을 한 경우엔 스터빙 값을 리턴합니다.

~~~java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Spy
    UserService userService;

    @Test
    public void getUser(){
        User user = userService.getUser();
        assertThat(user.getId()).isEqualTo(1L);
        assertThat(user.getName()).isEqualTo("test");

    }

    @Test
    public void getUserWithStubbing(){
        User dummyUser=new User(2L,"test2");
        when(userService.getUser()).thenReturn(dummyUser);
        User user = userService.getUser();
        assertThat(user.getId()).isEqualTo(2L);
        assertThat(user.getName()).isEqualTo("test2");

    }
}
~~~

<br>
### @InjectMock
**@InjectMock** 은  **@Mock** 이나 **@Spy** 로 생성된 mock 객체를 자동으로 주입해주는 어노테이션입니다. 테스트를 위해 UserService2를 하나 더 생성하도록 하겠습니다.
~~~java
@Service
public class UserService2 {

    public User2 getUser(){
        return new User2(1L,3);
    }
}

~~~

그 후UserService와 UserService2 주입받는 UserTotalService를 생성하겠습니다.
~~~java
@Service
@RequiredArgsConstructor
public class UserTotalService {
    private final UserService userService;
    private final UserService2 userService2;

    public User getUser(){
        return userService.getUser();
    }
    public User2 getUser2(){
        return userService2.getUser2();
    }

}


~~~
<br>
~~~java
@ExtendWith(MockitoExtension.class)
class UserTotalServiceTest {

    @Mock
    UserService userService;

    @Spy
    UserService2 userService2;

    @InjectMocks
    UserTotalService userTotalService;

    @Test
    public void getUser(){

        assertThat(userTotalService.getUser()).isNull();

    }
    @Test
    public void getUser2(){

        assertThat(userTotalService.getUser2().getId()).isEqualTo(1L);
        assertThat(userTotalService.getUser2().getAge()).isEqualTo(3);
    }
}
~~~

**@InjectMocks** 를 사용해 **@Mock**, **@Spy** 로 만든 객체들이 자동으로 주입된 것을 볼 수 있습니다.





<!--
@ExtendWith(SpringExtension.class)
@WebMvcTest(controllers = TestController.class)
class TestControllerTest {

    @Autowired

    MockMvc mockMvc;

   @Test
   public void test_to_return() throws Exception {

       mockMvc.perform(get("/test"))
               .andExpect(status().isOk())
               .andExpect(content().string("test"));

   }

}
 -->



<!-- 2. Integration Test
통합 테스트는 단위테스트 보다 -->
