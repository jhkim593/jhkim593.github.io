---
layout: post
title: "Mockito (3) - verify , BDD"
author: "jhkim593"
tags: TEST

---
## verify
verify 메소드를 이용해서 스터빙한 메소드가 실행됐는지, 실행이 초과되지 않았는지 등 다양한 검증이 가능합니다.

<br>

|method|description|
|:----------:|:----------:|
|times(n)| 몇 번이 호출됐는지 검증|
|never|한 번도 호출되지 않았는지 검증|
|atLeastOnce|최소 한 번은 호출됐는지 검증|
|atLeast(n)|최소 n 번이 호출됐는지 검증|
|atMostOnece|최대 한 번이 호출됐는지 검증|
|atMost(n)|최대 n 번이 호출됐는지 검증|
|calls(n)|n번이 호출됐는지 검증 (InOrder랑 같이 사용해야 함)|
|only|해당 검증 메소드만 실행됐는지 검증|
|timeout(long mills)|n ms 이상 걸리면 Fail 그리고 바로 검증 종료|
|after(long mills)|n ms 이상 걸리는지 확인 ,timeout과 다르게 시간이 지나도 바로 검증 종료가 되지 않는다.|
|description|실패한 경우 나올 문구|

<br>
<br>
테스트에 사용할 UserService입니다.
~~~java
@Service
public class UserService {


    public User getUser() {
        return new User(1L, "test");
    }

    public int getNum() {
        return 1;
    }

    public void updateUser() {

    }
}  
~~~

<br>
~~~java
@ExtendWith(MockitoExtension.class)
class VerifyTest {

    @Mock
    UserService userService;

    @Test
    public void testVerifyTime(){

        userService.getUser();
        userService.getUser();
        verify(userService,times(2)).getUser();
    }
    @Test
    public void testVerifyNever(){

        verify(userService,never()).getUser();
    }
    @Test
    public void testAtLeastOne(){
        userService.getUser();
        verify(userService,atLeastOnce()).getUser();

    }
    @Test
    public void testAtLeast(){
        userService.getUser();
        userService.getUser();
        userService.getUser();
        verify(userService,atLeast(2)).getUser();  //최소 두번
    }
    @Test
    public void testAtMostOne(){

        userService.getUser();
        verify(userService,atMostOnce()).getUser();
    }
    @Test
    public void testAtMost(){
        userService.getUser();
        userService.getUser();
        userService.getUser();

        verify(userService,atMost(3)).getUser();
    }
    @Test
    public void testOnly(){
        userService.getUser();
        verify(userService,only()).getUser();

    }
}
~~~

<br>

다음 InOrder를 이용해 메소드 호출 순서를 검증 해보겠습니다.

verifyNoMoreInteractions(T mock) - 선언한 verify 후 해당 mock을 실행하면 fail이 발생

verifyNoInteractions(T mock) - 테스트 내에서 mock을 실행하면 fail이 발생하게 됩니다.

<br>

~~~java
@ExtendWith(MockitoExtension.class)
class VerifyTest {

    @Mock
    UserService userService;

    @Mock
    UserService2 userService2;

    @Test
    public void testInOrder(){
        userService.getUser();
        userService.getNum();
        InOrder inOrder = inOrder(userService);
        inOrder.verify(userService).getUser();
        inOrder.verify(userService).getNum();

    }

    @Test
    public void testInOrderWithCalls(){

        userService.getUser();
        userService.getUser();
        userService.getNum();

        InOrder inOrder = inOrder(userService);
        inOrder.verify(userService,calls(2)).getUser();  //메소드 실행횟수 calls 사용
        inOrder.verify(userService).getNum();

    }

    //verify 후에 userService 사용하게 되면 fail 발생
    @Test
    public void testInOrderWithVerifyNoMoreInteractions(){

        userService.getUser();


        InOrder inOrder = inOrder(userService);

        inOrder.verify(userService).getUser();

        verifyNoMoreInteractions(userService);

    }


    //userService2를 호출하면 fail발생
    @Test
    void testInOrderWithVerifyNoInteractions() {
        userService.getUser();
        userService.getNum();


        InOrder inOrder = inOrder(userService);

        inOrder.verify(userService).getUser();
        inOrder.verify(userService).getNum();

        verifyNoInteractions(userService2);
    }


}
~~~

### BDD

**테스트** 를 기준으로 하는 개발 방법론인 **TDD** 와 다르게 **BDD**는 행동을 기준으로 하는 개발 방법론입니다.
- Given : 테스트를 위한 준비 , 사용하는 변수 , 입력 값 등을 정의하거나 Mock 객체를 정의하는 구문도 Given에 포함

- When : 테스트를 실행하는 과정을 명시

- Then : 테스트 결과를 검증하는 과정이다. 예상한 값 , 실제 실행을 통해서 나온 값을 검증한다.


Mockito는 BDD 스타일로 테스트 코드를 짤 수 있게 BDDMockito 클래스를 제공하며 기존 when->given ,verify->then으로 변경해주시면 됩니다.

~~~java
@Test
   public void test(){
       //given
       given(userService.getUser()).willReturn(new User(1L,"@2"));

       //when
       userService.getUser();
       userService.getUser();

       //then
       then(userService).should(times(2)).getUser();

   }
~~~
