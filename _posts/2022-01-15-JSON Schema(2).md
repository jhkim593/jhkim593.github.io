---
layout: post
title: "JSON Schema를 이용한 요청 데이터 유효성 검증(2)"
author: "jhkim593"
tags: Spring

---

이번 장에서는 java코드로 `JSON Schema`를 활용한 Validator를 구현해보겠습니다.<br>
java JSON Schema Validator 라이브러리로 ever-it을 활용했으며 추가로 검증 에러에 대한 메세지를 커스텀 해보고 테스트 까지 진행 해보도록 하겠습니다.

- ever-it : [https://github.com/everit-org/json-schema](https://github.com/everit-org/json-schema)


~~~gradle
implementation group: 'com.github.erosb', name: 'everit-json-schema', version: '1.14.0'
~~~

<br>

## 1. JsonValidator class
~~~java

@Component
@RequiredArgsConstructor
public class JsonValidator {

    private final ResourceLoader resourceLoader;
    private Map<String, Schema> jsonSchemaMap = new HashMap<>();

    @PostConstruct
    private void init(){
        getResources();
    }

    private void getResources() {
        Resource[] resources;
        try {
            resources = ResourcePatternUtils.getResourcePatternResolver(resourceLoader).getResources("classpath*:/schemas/*.json");  //(1) ResourcePatternUtils를 사용해 패턴 일치하는 특정 리소스를 로드했습니다.

            for (Resource resource : resources) {
                BufferedReader br = new BufferedReader(new InputStreamReader(resource.getInputStream(), StandardCharsets.UTF_8));  
                StringBuffer stringBuffer = new StringBuffer();
                String str = null;
                while ((str = br.readLine()) != null) {
                    stringBuffer.append(str).append("\n");
                }
                int endIndex = resource.getFilename().indexOf(".schema");
                String name = resource.getFilename().substring(0, endIndex);

                JSONObject jsonObject = new JSONObject(stringBuffer.toString());

                SchemaLoader schemaLoader = SchemaLoader.builder()
                        .schemaClient(SchemaClient.classPathAwareClient())  // (2 )스키마간 참조를 위한 classPath 설정
                        .schemaJson(jsonObject)
                        .build();
                Schema schema = schemaLoader.load().build();


                jsonSchemaMap.put(name, schema);
                br.close();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }

    }
    // (3) 커스텀 메시지 추출하기 위한 메소드
    public void messageCheck(ValidationException e, Map<String,String> requiredMessageMap, Map<String,String>messageMap, List<JsonSchemaErrorDto>errorDtoList){
        if(e.getCausingExceptions().size()>0){
            e.getCausingExceptions().stream().forEach(c->{
                Map<String, String> newRequiredMessageMap = (HashMap<String, String>) c.getViolatedSchema().getUnprocessedProperties().get("requiredMessages");
                Map<String, String> newMessageMap = (HashMap<String, String>) c.getViolatedSchema().getUnprocessedProperties().get("messages");

                newMessageMap=(newMessageMap==null)? messageMap : newMessageMap;
                newRequiredMessageMap=(newRequiredMessageMap==null)?requiredMessageMap:newRequiredMessageMap;

                messageCheck(c, newRequiredMessageMap, newMessageMap ,errorDtoList);
            });
        }
        else {
            JsonSchemaErrorDto  errorDto = new JsonSchemaErrorDto();

            String pointer=e.getPointerToViolation();

            //keyword required 일 때
            if(e.getKeyword().equals("required")) {
                int startIndex = e.getMessage().indexOf("[");
                int endIndex = e.getMessage().indexOf("]");
                String index = e.getMessage().substring(startIndex+1, endIndex);
                pointer+="/"+index;

                HashMap<String, String> newRequiredMessageMap = (HashMap<String, String>) e.getViolatedSchema().getUnprocessedProperties().get("requiredMessages");

                if(newRequiredMessageMap!=null) {
                    errorDto.setMessage(newRequiredMessageMap.get(index));
                }
                else{
                    if(requiredMessageMap!=null){
                        errorDto.setMessage(requiredMessageMap.get(index));
                    }
                }
            }

            //keyword required 아닐 때
            else {
                HashMap<String, String> newMessageMap = (HashMap<String, String>) e.getViolatedSchema().getUnprocessedProperties().get("messages");
                if(newMessageMap!=null){
                    errorDto.setMessage(newMessageMap.get(e.getKeyword()));
                }
                else{
                    if(messageMap!=null) {
                        errorDto.setMessage(messageMap.get(e.getKeyword()));
                    }
                }
            }

            if(errorDto.getMessage()==null){
                errorDto.setMessage("invalid");
            }
            errorDto.setPointer(pointer);
            errorDto.setKeyword(e.getKeyword());
            errorDtoList.add(errorDto);
        }
    }
    public List<JsonSchemaErrorDto> valid(String schemaName, String jsonStr) {

        try {
            if (jsonSchemaMap.containsKey(schemaName)) {
                if (jsonStr.charAt(0) == '[') {
                    JSONArray jsonArray = new JSONArray(jsonStr);
                    jsonSchemaMap.get(schemaName).validate(jsonArray);
                } else {
                    JSONObject jsonObject = new JSONObject(jsonStr);
                    jsonSchemaMap.get(schemaName).validate(jsonObject);
                }

            }
        } catch (ValidationException e) {
            ArrayList<JsonSchemaErrorDto> errorDtoList = new ArrayList<>();

            Map<String, String> requiredMessageMap = (HashMap<String, String>) e.getViolatedSchema().getUnprocessedProperties().get("requiredMessages");
            Map<String, String> messageMap = (HashMap<String, String>) e.getViolatedSchema().getUnprocessedProperties().get("messages");
            messageCheck(e,requiredMessageMap,messageMap,errorDtoList);
            return errorDtoList;
        }

        return null;
    }

}

~~~
- (1) : ResourcePatternUtils를 사용해 패턴 일치하는 특정 리소스를 로드했습니다.
- (2) : schemaClient(SchemaClient.classPathAwareClient())를 스키마간 참조를 위해 classPath설정을 했습니다.
- (3) : messageCheck 재귀함수를 이용해서 Schema 파일에 작성한 커스텀 메세지를 추출 했습니다.


<br>


## 2. JsonSchemaErrorDto

~~~java

@Getter
@Setter
public class JsonSchemaErrorDto {
    private String message;
    private String keyword;
    private String pointer;
}
~~~
- message : Schema파일에 작성한 커스텀 메세지 설정을 위한 변수
- keyword : type , maxLength 등 검증 오류 키워드 설정을 위한 변수
- pointer : 검증 오류 위치 정보를 위한 포인터 변수

<br>

## 3. Controller

~~~java
@RestController
@RequiredArgsConstructor
public class TestController {

    private final JsonValidator jsonValidator;


    @RequestMapping(value = "/json/test", method = RequestMethod.POST)
    public ResponseEntity jsonTest(@RequestBody String body , @RequestParam String schemaName)  {
        List<JsonSchemaErrorDto> conditionTest = jsonValidator.valid(schemaName, body);
        if(conditionTest==null){
            return new ResponseEntity(HttpStatus.OK);
        }
        return new ResponseEntity(conditionTest,HttpStatus.BAD_REQUEST);
    }
}
~~~
jsonTest 컨트롤러에서는 RequestParam을 이용해 스키마 파일을 입력받아 유효성 검증을 수행 할 것이고 검증에러가 있을 때 상태코드 400 , 없을 때 200 을 응답하도록 설정 했습니다.

<br>

## Test
Schema 파일을 작성해서 실제 요청 데이터 유효성 검증을 테스트 해보겠습니다.

#### 1. if -then 조건 테스트

- conditionTest.Schema.json 작성
~~~json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "id": {
      "type": "integer",
      "maxLength": 24,
      "minLength": 2,
      "messages": {
        "type": "id type is invalid",
        "maxLength": "id max Length is invalid",
        "minLength": "id min Length is invalid"
      }
    },
    "data": {
      "type": ["array"],
      "items": {
        "type": "object",
        "properties": {
          "person": {
            "type": "object",
            "messages": {
              "type": "person type is invalid"
            },
            "properties": {
              "personId": {
                "type": "number",
                "messages": {
                  "type": "personId type is invalid"
                }
              }
            }
          },
          "value": {
            "anyOf": [
              {"type": "string"},
              {"type": "null"}
            ],
            "maxLength": 5,
            "messages": {
              "type": "value type is invalid",
              "maxLength": "value maxLength is invalid"
            }
          }
        },
        "if": {
          "properties": {
            "value": {"type": "null"}
          }
        },
        "then": {
          "properties": {
            "person": {
              "required": ["personId"],
              "requiredMessages": {
                "personId": "personId required"
              }
            }
          },
          "required": ["person"],
          "requiredMessages": {
            "person": "person required"
          }
        },
        "required": [
          "value"
        ]
      },
      "maxItems": 10
    }
  },

  "required": ["id"]
}
~~~
array형식 data를 받으며 data 속성 value가 null 일때 person , personId required 설정을 추가했습니다.

<br>

- body
~~~json
{
    "id":1,
    "data":[
        {
            "value": "test",
            "person":{}
        },
        {
            "value":null,
            "person":{}
        },
        {
            "value":"aaaaaa",
            "person" :{}
        }
     ]    
}
~~~

<br>

- 결과
<img src="https://user-images.githubusercontent.com/53510936/156911476-8a99b902-1e7e-4337-b1d5-71000a21c9fb.png"  width="700" height="300"/>
- #/data/1/person/personId : value type이 null이기 때문에 유효성 검증 통과하지 못함
- #/data/2/value : value maxLength 5 이기 때문이기 때문에 유효성 검증 통과하지 못함

<br>

#### 2. definitions 테스트

- definitionsTest.shcema.test 작성
~~~json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "definitions": {
    "data": {
      "type": "object",
      "properties": {
        "person": {
          "type": "object",
          "messages": {
            "type": "person type is invalid"
          },
          "properties": {
            "personId": {
              "type": "number",
              "messages": {
                "type": "personId type is invalid"
              }
            }
          }
        },
        "value": {
          "anyOf": [
            {"type": "string"},
            {"type": "null"}
          ],
          "maxLength": 5,
          "messages": {
            "type": "value type is invalid",
            "maxLength": "value maxLength is invalid"
          }
        }
      },
      "if": {
        "properties": {
          "value": {"type": "null"}
        }
      },
      "then": {
        "properties": {
          "person": {
            "required": ["personId"],
            "requiredMessages": {
              "personId": "personId required"
            }
          }
        },
        "required": ["person"],
        "requiredMessages": {
          "person": "person required"
        }
      },
      "required": [
        "value"
      ]
    }
  },
  "type": "object",
  "properties": {
    "id": {
      "type": "integer",
      "maxLength": 24,
      "minLength": 2,
      "messages": {
        "type": "id type is invalid",
        "maxLength": "id max Length is invalid",
        "minLength": "id min Length is invalid"
      }
    },
    "datalist": {
      "type": ["array"],
      "items": {
        "$ref": "#/definitions/data"
      },
      "maxItems": 10
    }
  },

  "required": ["id"]
}
~~~
definitions 키워드를 이용해서 정의한 스키마를 참조했습니다.

- body
~~~json
{
    "id":1,
    "datalist":[
        {
            "value": "test",
            "person":{}
        },
        {
            "value":null,
            "person":{}
        },
        {
            "value":"aaaaaa",
            "person" :{}
        }
     ]    
}
~~~

- 결과
<img src="https://user-images.githubusercontent.com/53510936/156911476-8a99b902-1e7e-4337-b1d5-71000a21c9fb.png"  width="700" height="300"/>
`$ref` 키워드로 정의된 스키마를 참조 했기 때문에 결과는 위와 결과는 동일 합니다.

#### 3. classPath / equal test

ever-it 라이이브러리 사용시 다른 스키마를 참조하기 위해 classPath 설정을 추가 했습니다.


- classPathTest.schema.json작성

~~~json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "test": {
      "type": "integer",
      "maxLength": 24,
      "minLength": 2,
      "messages": {
        "type": "test type is invalid",
        "maxLength": "test max Length is invalid",
        "minLength": "test min Length is invalid"
      }
    },
    "data": {
      "$ref": "conditionTest.schema.json"
    }
  },
    "required": ["test"]
}
~~~
`$ref` 키워드를 사용해 conditionTest shcema를 참조하도록 설정 했습니다.

- body
~~~json
{
    "test":1,
    "data":{
        "id":2,
        "data":[
            {
               "value": "test",
               "person":{}
            },
            {
                "value":null,
                "person":{}
            }
        ]

    }
}
~~~
<br>

- 결과
<img src="https://user-images.githubusercontent.com/53510936/157264606-685c235d-9b55-44c0-81d6-a1961b08331b.png"  width="800" height="300"/>
위와 같은 결과가 나오며 다른 스키마를 참조해 유효성 검증이 동작 하는 것을 확인했습니다.

---
<br>

# Reference

<br>

관련 github링크 : [https://github.com/jhkim593/JSON-Schema-ever-it](https://github.com/jhkim593/JSON-Schema-ever-it)
