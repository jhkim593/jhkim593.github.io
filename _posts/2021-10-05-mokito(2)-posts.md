---
layout: post
title: "Mockito (2) - Stubbing"
author: "jhkim593"
tags: SpringBoot

---

## stubbing
모의 객체 생성 및 모의 객체의 동작을 지정하는 것을 Stubbing이라고 합니다.

특정 매개변수를 받았을 때 특정 값을 반환하거나 예외를 던지도록 설정할 수 있으며,
thenReturn() 메서드나 thenThrow() 메서드를 이어 붙이는 구조를 사용하여
동일한 메서드가 여러 번 호출될 때 각각 다르게 행동하도록 할 수 있습니다.

~~~java
@Test
   public void stubbingTest(){
       User dummyUser1=new User(1L,"dummy1");
       when(userService.getUser()).thenReturn(dummyUser1).thenThrow(new RuntimeException());
       User user1 = userService.getUser();
       assertThat(user1.getId()).isEqualTo(1L);
       assertThat(user1.getName()).isEqualTo("dummy1");
       assertThatThrownBy( ()->{
           userService.getUser();
       }).isInstanceOf(RuntimeException.class);

   }
~~~

Mock 객체 메서드의 파라미터 값을 하드 코딩하고 싶지 않다면
Mockito의 Argument Matchers를 이용하면 됩니다.
~~~java
@Test
 public void stubbingArgumentMatchers(){
     when(userService.getUserWithIdAndName(anyLong(),anyString())).thenReturn("test");
     assertThat(userService.getUserWithIdAndName(1L,"test1")).isEqualTo("test");
     assertThat(userService.getUserWithIdAndName(2L,"test2")).isEqualTo("test");
 }
}
~~~

Mockito에선 when 메소드를 이용해서 스터빙을 지원하고 있으며 ,
스터빙을 할 수 있는 방법은 **OngoinStubbing** , **Stubber**를 쓰는 방법 2가지가 있습니다.

<br>

### OngoingStubbing
OngoingStubbing 메소드란 when에 넣은 메소드의 리턴 값을 정의해주는 메소드입니다.

|method|description|
|:----------:|:----------:|
|thenReturn| 스터빙한 메소드 호출 후 return할 객체 정의|
|thenAnswer|스터빙한 메소드 호출 후 custom작업을 정의|
|thenThrow|스터빙한 메소드 호출 후 Throw할 Exception정의|
|thenCallRealMethod|실제 메소드 호출|

<br>
테스트를 위해 앞서 생성한 UserService를 사용하겠습니다.

~~~java
@ExtendWith(MockitoExtension.class)
class UserTotalServiceTest {

    @Mock
    UserService userService;


    @Test
    public void testThenReturn() {
        User dummyUser = new User(10L, "test1");
        when(userService.getUser()).thenReturn(dummyUser);
        User user = userService.getUser();
        assertThat(user.getId()).isEqualTo(10L);
        assertThat(user.getName()).isEqualTo("test1");

    }
    @Test
    public void testThenAnswer() {
        when(userService.getUserWithIdAndName1(any(), any())).thenAnswer((Answer) invocation -> {
            Object[] args = invocation.getArguments();
            return new User((Long) args[0] + 3L, args[1] + "d");

        });
        User user = userService.getUserWithIdAndName1(1L, "test");
        assertThat(user.getId()).isEqualTo(4L);
        assertThat(user.getName()).isEqualTo("testd");
    }

    @Test
    public void testThenThrows() {
        when(userService.getUser()).thenThrow(new RuntimeException());
        assertThatThrownBy(() -> {
            userService.getUser();
        }).isInstanceOf(RuntimeException.class);


    }
    @Test
    public void testThenCallRealMethod(){
        when(userService.getUser()).thenCallRealMethod();
        User user = userService.getUser();
        assertThat(user.getId()).isEqualTo(1L);
        assertThat(user.getName()).isEqualTo("test");

    }
}

~~~

<br><br>

### Stubber
<!-- do while느낌 -->
Stubber 메소드는 OngoingStubbing과 다르게 when에 스터빙할 클래스를 넣고 그 후에 메소드를 호출합니다. return 타입이 void인 메소드 테스트가 가능해집니다.

|method|description|
|:----------:|:----------:|
|doReturn| 스터빙 메소드 호출 후 행동 정의|
|doThrow| 스터빙 메소드 호출 후 throw할 Exception정의|
|doAnswer|스터빙 메소드 호출 후 custom작업 정의|
|doNothing|스터빙 메소드 호출 후 행동 하지 않도록 정의|
|doCallRealMethod|실제 메소드를 호출 |

<br>

테스트를 위해 userService에 return타입이 void인 메소드를 추가했습니다.
~~~java
public void updateUser(){

   }
~~~


~~~java
@Test
public void testDoReturn(){

    User user=new User(2L,"test");
    doReturn(user).when(userService).getUser();
    assertThat(userService.getUser()).isEqualTo(user);

}

@Test
public void testDoThrow(){
    doThrow(new RuntimeException()).when(userService).updateUser();
    assertThatThrownBy(()-> userService.updateUser()).isInstanceOf(RuntimeException.class);

  }
}

~~~

return 타입이 void인 메소드 또한 테스트 할 수있는 것을 확인했습니다.
