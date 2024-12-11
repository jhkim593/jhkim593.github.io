---
layout: post
title: "부작용과 격리된 순수 함수 만들기"
author: "jhkim593"
tags: Java
---
프로젝트 리팩토링을 진행하면서 이동욱님 블로그에서 [좋은 함수 만들기](https://jojoldu.tistory.com/697) 게시글을 읽게 되었고

프로젝트에 적용했던 과정을 정리했습니다.

<br>
## 순수 함수 , 부작용 함수

<br>
### **순수 함수란**

**동일한 입력일 경우 항상 동일한 출력을 반환**하며 부작용이 없는 함수를 의미합니다.

<br>
### 부작용 함수란

함수 내 구현이 반환값뿐만 아니라 **함수 외부 상태에 영향을 미치는 함수**를 의미합니다. 예시는 다음과 같습니다.

- 전역 변수를 수정하는 경우
- 외부 api를 호출하는 경우
- 화면이나 파일에 데이터를 쓰는 경우

<br>
순수 함수의 경우 입력이 같다면 출력이 동일하기 때문에 테스트 작성이 쉬운 반면 부작용 함수는 함수 외부에 영향을 주므로 테스트하기 어렵습니다.

또한 부작용 함수는 **전염성이 높아** 부작용 함수를 사용하는 모든 함수들이 마찬가지로 테스트하기 어려워집니다. 그래서 **순수 함수와 부작용 함수를 격리시키는 것이 중요**합니다.

이제부터 순수 함수와 부작용 함수를 격리시켜 프로젝트 코드를 리팩토링한 과정에 대해 자세히 설명하겠습니다.

<br>
## 리팩토링

리팩토링 했던 기능은 다음과 같습니다.

분석 작업이 완료되면, 사용자가 사전에 설정한 URL로 분석 성공 콜백을 전송합니다.

콜백에는 다음 정보가 포함됩니다:

- 현재 이슈 페이지 번호
- 총 이슈 수
- 결과 이슈 데이터

예를 들어 사용자가 http://test/v1를 사전에 입력했고

분석 결과로  [”A”,”B”] 총 두개 이슈가 생성됐다고하면 총 2번의 http 요청이 전송됩니다.

```json
http://test/v1
{
    "pageNo" : 1 ,
    "lastPageNo" : 2,
    "issue" : "A"
}
```

```json
http://test/v1
{
    "pageNo" : 2 ,
    "lastPageNo" : 2,
    "issue" : "B"
}
```

<br>
서비스 코드는 다음과 같습니다.

```java
@Service
@RequiredArgsConstructor
public class CallbackService {
    private final RestHttpClient httpClient;
    private final ObjectMapper objectMapper;

    public void analysisSuccessCallback(Analysis analysis) throws Exception {
        List<String> issues = analysis.getIssues();
        String type = analysis.getType();
        CallbackUrl callbackUrl = analysis.getCallbackUrl();

        int pageNo = 1;
        for (String issue : issues ) {
            //1
            CallbackBody body = getBody(issue, pageNo, issues, type);
            //2
            httpSend(body, callbackUrl);

            pageNo++
        }
    }

    private CallbackBody getBody(String issue, int pageNo, List<String> issues, String type) {
        return new CallbackBody(pageNo, issues.size(), type, issue);
    }

    private void httpSend(CallbackBody body, CallbackUrl callbackUrl) throws JsonProcessingException {
        httpClient.sendRequest(objectMapper.writeValueAsString(body), callbackUrl.getUrl(),"POST", callbackUrl.convertToheaderMap());
    }
}
```

`CallbackService` 의 `sendSuccessCallback` 메소드는 아래 기능을 수행합니다.

- `getBody` : callback에 전송 할 body 생성
- `httpSend` :  http 요청 전송

<br>
하지만 테스트를 작성하기가 까다로웠습니다.

`getBody` 메소드는 요청 body를 생성하는 **핵심 로직**으로, 해당 로직을 테스트하고자 했으나 클래스 내에서만 사용되는 **private 메소드**이기 때문에 메소드의 접근 범위를 변경해 직접 테스트 하는 것은 적절한 방법이 아니었습니다.

그래서 `sendSuccessCallback` 메소드를 테스트하는 것으로 대신하려 했지만, 이 메소드는 **외부 연동(HTTP 요청)으로 인해 부작용이 포함**되어 있어, body 생성 로직에 대한 테스트 또한 작성하기가 어려웠습니다.


<br>
body를 생성하는 `getBody` 메소드는 같은 입력에 같은 출력을 반환하는 순수 함수입니다. 따라서 부작용과 격리하기 위해 해당 로직을 새로운 클래스로 추출했습니다

```java
public class SuccessCallback {
    private final String type;
    private final CallbackUrl callbackUrl;
    private final List<String> issues;

    public SuccessCallback(String type, CallbackUrl callbackUrl, List<String> issues) {
        this.type = type;
        this.issues = issues;
        this.callbackUrl = callbackUrl;
    }

    public CallbackBody getBody(int index){
        return new CallbackBody(index+1, issues.size(), type, issues.get(index));
    }

    public CallbackUrl getCallbackUrl() {
        return callbackUrl;
    }

    public int getBodyCount() {
        return issues.size();
    }
}
```

`SuccessCallback` 클래스를 생성해 callback url , body 등 관련 정보를 모두 이 클래스에서 담당하도록 했습니다. 생성자를 사용하는 것 외 필드를 수정할 수 없도록 했으며 외부에서 사용해야 할 메소드들을 오픈했습니다.

- `getbody`: index를 파라미터로 받아 callback body를 생성
- `getCallbackUrl` : 전송할 URL 정보 조회
- `getBodyCount` : 전송해야 할 총 body Count 조회

<br>
추출된 `SuccessCallback` 클래스는 부작용이 포함되지 않은 순수 로직이기 때문에 테스트를 작성하기 매우 쉬웠습니다.

```java
class SuccessCallbackTest {
    SuccessCallback callback;

    @BeforeEach
    protected void set() {
        //CallbackUrl 설정
        CallbackUrl callbackUrl = new CallbackUrl();
        String url1 = "http://localhost:8080";
        callbackUrl.setUrl(url1);

        String headerKey1 = "key1";
        String headerValue1 = "value1";
        callbackUrl.setHeaders(Arrays.asList(new CallbackHeader(headerKey1, headerValue1)));

        //결과 설정
        List<String> issues = Arrays.asList("a", "b");

        callback = new SuccessCallback("type",callbackUrl, issues);
    }

    @Test
    public void getBodyCount(){
        //when
        int count = callback.getbodyCount();
        //then
        assertThat(count).isEqualTo(2);
    }

    @Test
    public void getCallbackUrl(){
        //when
        CallbackUrl getCallbackUrl = callback.getCallbackUrl();
        //then
        assertThat(getCallbackUrl.getUrl()).isEqualTo("http://localhost:8080");

        CallbackHeader getCallbackHeader1 = getCallbackUrl.getHeaders().get(0);
        assertThat(getCallbackHeader1.getKey()).isEqualTo("key1");
        assertThat(getCallbackHeader1.getValue()).isEqualTo("value1");
    }

    @Test
    public void getBody() throws IOException {
        //when
        CallbackBody body1 = callback.getBody(0);
        CallbackBody body2 = callback.getBody(1);

        //then
        assertThat(body1.getType()).isEqualTo("type");
        assertThat(body1.getIssue()).isEqualTo("a");
        assertThat(body1.getPageNo()).isEqualTo(1);
        assertThat(body1.getLastPageNo()).isEqualTo(2);

        assertThat(body2.getType()).isEqualTo("type");
        assertThat(body2.getIssue()).isEqualTo("b");
        assertThat(body2.getPageNo()).isEqualTo(2);
        assertThat(body2.getLastPageNo()).isEqualTo(2);
    }
}
```

Junit으로 작성한 테스트 코드입니다. 외부의 의존성이 없기 때문에 따로 mocking 하지않아도 테스트 코드를 쉽게 작성 할 수 있었습니다.

<br>
`SuccessCallback` 클래스를 적용해 최종적으로 다음과 같이 리팩토링 했습니다.

```java
@Service
@RequiredArgsConstructor
public class CallbackService {
    private final RestHttpClient httpClient;
    private final ObjectMapper objectMapper;

    public void sendSuccessCallback(Analysis analysis) throws Exception {
        List<String> issues = analysis.getIssues();
        String type = analysis.getType();
        CallbackUrl callbackUrl = analysis.getCallbackUrl();
        SuccessCallback callback = new SuccessCallback(type,callbackUrl, issues);

        httpSend(callback);
    }

    private void httpSend(SuccessCallback callback) throws JsonProcessingException {
        int bodyCount = callback.getBodyCount();
        CallbackUrl callbackUrl = callback.getCallbackUrl();
        for(int j=0; j< bodyCount; j++){
            httpClient.sendRequest(
                    objectMapper.writeValueAsString(callback.getBody(j)), callbackUrl.getUrl(), "POST", callbackUrl.convertToheaderMap());
        }
    }
}
```

`sendSuccessCallback` 메소드는 **http 요청으로 인한 부작용이 포함**되어있기 때문에 아직 테스트 작성이 어렵지만 Callback 데이터 (body , url ) 를 담당하는 `SuccessCallback` 클래스를 **부작용과 분리해 테스트 작성이 쉬운 영역을 확보**할 수 있었습니다.

`SuccessCallback` 클래스를 테스트 하는 것으로 핵심 로직이 검증되었기 때문에 부작용으로 테스트가 어려운 부분은 테스트 더블을 사용하거나 [WireMock](https://wiremock.org/docs/download-and-installation/) 같은 라이브러리로 보완하면 될 것입니다.

어플리케이션 코드를 작성하다 보면 부작용이 있는 함수는 반드시 존재하게 됩니다. 그러나 부작용과 순수 로직을 격리하고, **테스트가 쉬운 영역을 최대한 확보**하면 프로젝트의 유지보수성이 향상되고 더욱 안정적인 프로젝트로 발전할 수 있을 것이라 생각합니다.

---
## Reference
<https://jojoldu.tistory.com/697?category=1011740>
