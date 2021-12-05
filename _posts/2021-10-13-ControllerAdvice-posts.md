---
layout: post
title:  "ControllerAdvice를 이용한 Exception처리"
author: "jhkim593"
tags: SpringBoot

---
Spring에서 제공하는 **@ContorllerAdvice** 를 사용하면 Controller전역에 적용되는 코드를 작성 할 수있으며 추가로 패키지를 적용하면 특정 패키지 하위의 Controller에만 로직이 적용되게도 할 수 있습니다.
여기서는 예외 발생시 json형태로 결과를 반환 할 것 이기 때문에 **@RestControllerAdvice** 를 사용할 것이며 , **@RestControllerAdvice** 와 **@ExceptionHandler** 를 조합하여 예외처리를 공통코드로 분리하여  예외가 발생했을 때 표준화된 데이터 구조를 응답메시지로 처리하는 것을 다뤄보겠습니다.


<br>

### CommonResult 생성
~~~java
@Getter
@Setter
public class CommonResult {
    private boolean success;
    private String msg;
}
~~~

### ListResult 생성
~~~java
@Getter
@Setter
public class ListResult<T> extends CommonResult{
    private List<T> list;
}
~~~
### SingleResult 생성
~~~java
@Getter
@Setter
public class SingleResult<T> extends  CommonResult{
    private T data;
}
~~~

### ResponseService 생성
~~~java
@Service
public class ResponseService {

    public <T> SingleResult<T> getSingleResult(T data) {
        SingleResult<T> result = new SingleResult<>();
        result.setData(data);
        setSuccessResult(result);
        return result;
    }

    public <T> ListResult<T> getListResult(List<T> list) {
        ListResult<T> result = new ListResult<>();
        result.setList(list);
        setSuccessResult(result);
        return result;
    }

    public CommonResult getSuccessResult() {
        CommonResult result = new CommonResult();
        setSuccessResult(result);
        return result;
    }

    public CommonResult getFailResult(String msg) {
        CommonResult result = new CommonResult();
        result.setSuccess(false);
        result.setMsg(msg);
        return result;
    }

    private void setSuccessResult(CommonResult result) {
        result.setSuccess(true);
        result.setMsg("성공했습니다.");
    }
}
~~~
결과 데이터의 구조를 표준화 하는 공통 모델을 생성했습니다.

<br>

~~~java
public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(String msg, Throwable t) {
        super(msg, t);
    }

    public UserNotFoundException(String msg) {
        super(msg);
    }

    public UserNotFoundException() {
        super();
    }
}
~~~
여러 가지 예외 상항을 구분하기 위해 CustomException을 정의헸습니다. RuntimeException을 상속받아 작성했으며, 이 예외가 발생했을 때 ExceptionHandler에서 처리 후 응답 메세지를 보내게 됩니다.

<br>

### RestControllerAdvice 설정
~~~java
@RestControllerAdvice
@RequiredArgsConstructor
public class ExceptionAdvice {

   private final ResponseService responseService;

   @ExceptionHandler(Exception.class) //(1)
   @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR) //(2)
   protected CommonResult defaultException(HttpServletRequest request, Exception e) {
       return responseService.getFailResult("에러가 발생했습니다.")
   }

    @ExceptionHandler(UserNotFoundException.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    protected CommonResult userNotFoundException(HttpServletRequest request, Exception e) {
        return responseService.getFailResult("등록된 회원이 없습니다.");
    }
}
~~~

- (1)ExceptionClass는 최상위 예외 처리 객체이기 때문에 ExceptionHandler에서 걸러지지 않은 예외가 있으면 최종적으로 처리가 진행됩니다.
- (2)해당 Exception이 발생하면 Response가 출력되는 HttpStatus Code가 500이 되도록 설정했습니다.

<br>

~~~java
@Service
@RequiredArgsConstructor
@Transactional
public class UserService {
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    public User findById(Long userId){
          return userRepository.findById(userId).orElseThrow(UserNotFoundException::new);
      }
  }
~~~
userId로 user정보를 검색하는 간단한 메소드를 만들어 예외를 발생시켜 앞에서 설정한 예외처리가 잘 작동하는지 테스트 해보겠습니다.

<br>

### 테스트
~~~java
@Test
   public void exceptionHandlingTest()throws Exception{
       mockMvc.perform(MockMvcRequestBuilders.get("/user/100")
                       .contentType(MediaType.APPLICATION_JSON).
                       header(HttpHeaders.AUTHORIZATION,"Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIyIiwiaWF0IjoxNjM0MTAzMDQ2LCJleHAiOjE2MzQxMDY2NDZ9.zng21RDo_8zygaLL_BWkCRygT03RF3uZJySPZEnnqmM"))
               .andExpect(status().is5xxServerError())
               .andDo(MockMvcResultHandlers.print());
  }
~~~


~~~console
Status = 500
Error message = null
 Headers = [Content-Type:"application/json;charset=utf-8", X-Content-Type-Options:"nosniff", X-XSS-Protection:"1; mode=block", Cache-Control:"no-cache, no-store, max-age=0, must-revalidate", Pragma:"no-cache", Expires:"0", X-Frame-Options:"DENY"]
Content type = application/json;charset=utf-8
    Body = {"success":false,"message":"등록된 회원이 없습니다."}
Forwarded URL = null
Redirected URL = null
 Cookies = []
~~~

입력한 userId에 해당하는 user가 DB에 없을시 UserNotFoundException이 발생하게 되고 ExceptionHandler가 이것을 처리하여 기존에 설정한 응답 데이터가 오는 것을 확인할 수 있었습니다.
