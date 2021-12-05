---
layout: post
title:  "Postman - tests"
author: "jhkim593"

tags: SpringBoot

---

## Tests

~~~json
{
    "test": "hi",
    "test2":"hi2"
}
~~~
body에 데이터를 넣어 API를 테스트 해보겠습니다.
<br><br>

### Dto
~~~java
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
public class TestDto {
    private String test;
    private String test2;
}

~~~

### Controller

~~~java
@RestController
public class TestController {
@GetMapping("/test")
   public ResponseEntity apiTest(@RequestBody TestDto testDto){
       return new ResponseEntity(new TestDto(testDto.getTest(), testDto.getTest2()), HttpStatus.OK);
   }
}
~~~

<br>

~~~javascript
pm.test("Status code is 200", function () {
    pm.response.to.have.status(200);                            //status code 확인
    pm.expect(pm.environment.get("IP")).to.equal("localhost")   //환경변수값 일치 확인
});
pm.test("Response must have the test property", function () {
    var jsonData = pm.response.json();
    pm.expect(jsonData.test).to.eql("hi");                    
    pm.expect(jsonData).to.have.keys('test','test2');             //json값 확인
});
pm.test("Body matches string", function(){ pm.expect(pm.response.text()).to.include("hi2"); });  // responseBody에 문자열 포함 확인
pm.test("Content-Type is present", function(){ pm.response.to.have.header("Content-Type"); });   //content-type 헤더 포함 확인
pm.test("Response time is less than 200ms",function(){ pm.expect(pm.response.responseTime).to.be.below(200); }); //응답시간 확인
~~~
Postman tests에 테스트 스크립트를 작성합니다.

그 외에도

~~~javascript
pm.test("Content-Type is present", function(){ pm.response.to.have.header("Content-Type"); });
~~~
ContentType이 헤더에 포함되어 있는지

<br>

~~~javascript
pm.test("Status code name has String",function(){ pm.response.to.have.status("Created"); });
~~~
status code 이름이 문자열이며 일치하는지를 검증할 수있다.
