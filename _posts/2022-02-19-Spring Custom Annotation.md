---
layout: post
title: "Spring Custom Annotation"
author: "jhkim593"
tags: SpringBoot

---

## Annotation
Annotation은 Java5부터 새롭게 추가된 문법요소로, 사전적 의미는 주석이라는 의미를 가지고 있습니다. 자바에서 Annotation을 소스코드에 추가하면 단순 주석의 기능을 하는 것이 아니라 특별한 기능을 사용할 수 있습니다. 예를들어 Spring Framework는 해당 클래스가 어떤 역할인지 정하기도 하고, Bean을 주입하기도 하며, 자동으로 getter나 setter를 생성하기도 합니다.

여기서는 이미 만들어진 어노테이션 외에 직접 커스텀해서 어노테이션을 만들어 사용해보도록 하겠습니다.

<br>


## 구현
~~~java
@Target({ElementType.METHOD}) // 1
@Retention(RetentionPolicy.RUNTIME) // 2
public @interface Test {  //3
}
~~~


#### 1. @Target
어노테이션이 생성될 위치를 지정하는 어노테이션입니다.

<br>

|name|description|
|:----------:|:----------:|
|PACKAGE|패키지 선언 시|
|TYPE|타입 (클래스 , 인터페이스 , enum)선언시|
|CONSTRUCTOR|생성자 선언시|
|FIELD|enum 상수를 포함한 멤버변수 선언시|
|METHOD|메소드 선언시|
|ANNOTATION_TYPE|어노테이션 타입 선언시|
|LOCAL_VARIABLE|지역변수 선언시|
|PARAMETER|파라미터 선언시|
|TYPE_PARAMETER|파라미터 타입 선언시|

<br>


#### 2. @Retention
어노테이션이 언제까지 유효할지 지정하는 어노테이션입니다.

<br>

|name|description|
|:----------:|:----------:|
|RUNTIME|컴파일 이후에도 참조가능|
|CLASS|클래스를 참조할 때 까지 유효|
|SOURCE|컴파일 이후 어노테이션 정보 소멸|

<br>

#### 3. @interface
@interface 추가해 어노테이션을 생성할 수있습니다.


<br>

## Example

이 페이지에서는 커스텀 어노테이션을 생성해 컨트롤러 메소드에 붙여서 인터셉터 단에서 특정 검증로직을 수행할 수있도록 해보겠습니다.

### Annotation 생성

~~~java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface JsonValidatorUtil {
     String schema() default "";
     Checker[] checker(); //1
}
~~~
1. JsonValidatorUtil에서 Checker어노테이션을 생성 후 사용가능

<br>

~~~java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Checker {
     String[] check() default {};
     String name() default "";
}
~~~

<br>

### 컨트롤러 메소드에 어노테이션 적용

~~~java
@JsonValidatorUtil(schema = "userPost", checker = {
            @Checker(name ="userChecker")
    })
    @RequestMapping(value = "/api/v1/user", method = RequestMethod.POST)
    public ResponseEntity userPost(@RequestBody UserDto userDto){
			userService.post(userDto);
			return new ResponseEntity(newGadgetDto, HttpStatus.OK);
		}
~~~
컨트롤러 메소드위에 생성한 JsonValidatorUtil 어노테이션을 적용해 userPost 스키마 파일을 사용하도록 했으며 , userChecker를 통해 특정로직을 인터셉터에서 수행하도록 했습니다.

<br>

### 인터셉터 설정

~~~java

@Component("jsonValidateInterceptor")
@RequiredArgsConstructor
public class JsonValidateInterceptor extends HandlerInterceptorAdapter {

    @Autowired
    @Qualifier("jsonValidator")
    private JsonValidator jsonValidator;

    private Map<String, JsonCustomChecker> checkerMap =new HashMap<>();

    private final JsonCustomChecker userChecker;

    @PostConstruct
    public void init() {
        checkerMap.put("userChecker",userPasswordEqualChecker);
    }

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

        // json 요청의 경우만 validation 처리를 한다.
        if(request.getContentType() == null || !request.getContentType().contains("application/json")){
            return super.preHandle(request, response, handler);
        }
        ErrorResponseDto errorResponseDto = new ErrorResponseDto();

        // request body를 가져옴
        String requestBody = getRequestBody(request);

        ValidResultDto validResultDto=null;
        JsonValidatorUtil annotation = getAnnotation((HandlerMethod) handler, JsonValidatorUtil.class);
        if(annotation!=null) {
					  //어노테이션에 schema값을 이용
            validResultDto = jsonValidator.valid(annotation.schema(), requestBody);
            errorResponseDto.setErrors(validResultDto.getMsgList());
            JsonCustomChecker jsonCustomChecker=null;
						//어노테이션 checker값 이용
            for (Checker check : annotation.checker()) {
                if ((jsonCustomChecker = checkerMap.get(check.name())) != null) {
                    List<ErrorDto> errorDtoList = jsonCustomChecker.check(annotation.checker()[0], requestBody);
                    if (errorDtoList != null) {
                        errorDtoList.stream().forEach(c -> errorResponseDto.addErrorDto(c));
                    }
                }
            }
        }
            response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
            errorResponseDto.setStatus(HttpStatus.BAD_REQUEST);
            String resultStr = objectMapper.writeValueAsString(errorResponseDto);
            response.setCharacterEncoding("UTF-8");
            PrintWriter out = response.getWriter();
            out.print(resultStr);
            out.flush();
            out.close();
            return false;

        return super.preHandle(request, response, handler);
    }

    private <A extends Annotation> A getAnnotation(HandlerMethod handlerMethod, Class<A> annotationType) {
        return Optional.ofNullable(handlerMethod.getMethodAnnotation(annotationType))
                .orElse(handlerMethod.getBeanType().getAnnotation(annotationType));
    }
}
~~~
어노테이션에 설정된 checker name을 이용해서 인터셉터에서 해당 체커로 검증 로직을 수행할 수 있습니다.

<br>

## 주의
어노테이션을 사용하면 코드가 간결해지기 때문에, 자칫하면 커스텀 어노테이션을 남발할 수 있습니다. 주의 사항을 일부 발췌해왔습니다.

<br>

어노테이션의 의도는 숨어있기 때문에 내부적으로 어떤 동작을 하게 되는지 명확하지 않다면 로직 플로우를 이해하기 어렵게 되고, 코드정리가 덜 되어 현재 사용되지 않고 있는 어노테이션들이 있더라도 쉽사리 누군가가 손을 대기 부담스러워하는 경우도 왕왕 봐왔습니다. 하물며 ‘커스텀’ 어노테이션은 그 부담을 가중시킵니다. 무분별한 어노테이션 추가가 당장의 작업 속도를 끌어올릴 순 있지만, 긴 관점에서 시의적절한 것인지를 공감할 수 있어야 합니다.

<br>

구성원간의 이해와 공감대가 선행돼야 합니다. 또한 커스텀 어노테이션은 플로우에 대한 컨텍스트를 담고 있기 때문에 용도와 목적에 맞게 작성하는 것이 중요합니다. 다른 말로, 이것 저것 할 수 있는 다기능으로 만들게 되면 해석이 어려워진다는 뜻입니다.

간결한 코드를 위해 어노테이션을 무작성 생성해 사용하게 되었을 때 생기는 문제도 크기 때문에 어노테이션 커스텀이 꼭 필요한 부분을 확인 후 적용하는 것이 중요할 것같습니다.
