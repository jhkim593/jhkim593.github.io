---
layout: post
title: "JSON Schema를 이용한 요청 데이터 유효성 검증"
author: "jhkim593"
tags: Spring

---

## JSON Schema
Key와 Value의 쌍으로 구성된 JSON 데이터가 정해진 규칙에 따라 구성되어 있는지 유효성 검사를 할 수 있는 방법이며 JSON 데이터의 형식을 기술한 문서입니다.

### JSON Schema 검증 키워드
JSON Schema는 동일하게 JSON 형식으로 되어있으며 유효성 검증을 위한 키워드가 있습니다..
- type: 필드의 유효한 데이터 타입을 명시
- required: 반드시 포함되어야 하는 속성을 배열 형태로 표현
- properties: 객체(object) 타입인 경우 속성을 표기
- minLength: 문자열의 최소 길이
- maxLength: 문자열의 최대 길이
- minItems: 배열 요소의 최소 개수
- maxItems: 배열 요소의 최대 개수
- pattern: 정규 표현식과 일치한 문자열

### JSON Schema 데이터 타입
- number	모든 숫자 형태
- integer	정수 형태
- string	문자열/텍스트 형태로 표현
- array	연속된 요소들
- object	키/값 형태로 구성된 문자열
- boolean	문자열의 최소 길이	true 또는 false
- null	값이 없는 경우	null

### array 요소 검사
items 키워드를 사용해서 배열 요소에 대한 유효성 검증을 수행 할 수 있습니다.


~~~json
{
    "type": "array",
    "items": {
        "type": "integer"
    }
}
~~~
위의 예제에서 배열 요소가 모두 정수인 배열이나 배열 요소가 하나도 없는 빈 배열은 검증을 통과합니다.
하지만 배열 요소로 정수 외의 데이터를 가지는 배열은 검증을 통과하지 못합니다.
<br>

~~~json
{
    "type": "array",
    "items": [
        {
            "type": "integer"
        },
        {
            "type": "string",
            "maxLength": 10
        }
    ],
    "additionalItems": false
}
~~~
요소들에 대해 각각 검증 형식을 지정 할 수 있습니다. 첫번째 요소 정수형 이며 두번째 요소는 문자열 타입 , 길이 10을 넘지 않아야 합니다. 마지막에 설정한 additionalItems의 값이 false인 경우 정의한 요소 외에 다른 요소는 허용하지 않습니다.

### array 중복값 검증
uniqueItems 키워드를 사용하여 해당 배열에 저장된 배열 요소에 대한 중복 값 허용 여부를 설정 할 수 있으며, uniqueItems 값이 true일때, 배열 요소의 값에 중복 값을 허용하지 않습니다.

~~~json
{
    "type": "array",
    "uniqueItems": true
}
~~~

<br>

### array 타입 길이 검증
~~~json
{
    "type": "array",
    "minItems": 5,
    "maxItems": 10
}
~~~
배열 요소 최소 , 최대 개수를 설정 할 수 있습니다.


### Schema 재사용
definitions 키워드는 스키마를 정의하고 $ref를 사용해 정의한 스키마를 재사용 할 수있습니다.

~~~json
{
   "type":"object",
   "properties":{
      "contents":{
         "type":"array",
         "items":[{ "$ref":"#/definitions/content" }]
      },
      "datas":{
         "type":"array",
         "items":[{ "$ref":"#/definitions/data" }]
      }
   },
   "definitions":{
      "content":{
         "type":"object",
         "properties":{
            "id":{ "type":"string" },
            "type":{ "type":"string"}
         }
      },
      "data":{
         "type":"object",
         "properties":{
            "id":{ "type":"string" },
            "type":{ "type":"string", "enum":[ "photo1", "photo2" ]},
            "name":{ "type":"string" }
         },
         "required":[ "name" ]
      }
   }
}
~~~

### if - then - else

if , then , else 를 이용해 조건에 따라 유효성 검증 방식을 설정 할 수있습니다.

- if 조건 충족시 `then`을 실행
- if 조건 충족되지 않으면 `else` 실행

~~~json
{
  "type": "object",
  "properties": {
    "street_address": {
      "type": "string"
    },
    "country": {
      "default": "United States of America",
      "enum": ["United States of America", "Canada"]
    }
  },
  "if": {
    "properties": {
      "country": { "const": "United States of America" },
      "street_address":{"const": "BBAA2"}
     }
  },
  "then": {
    "properties": { "postal_code": { "pattern": "[0-9]{5}(-[0-9]{4})?" } }
  },
  "else": {
    "properties": { "postal_code": { "pattern": "[A-Z][0-9][A-Z] [0-9][A-Z][0-9]" } }
  }
}
~~~
if에 ,를 구분자로 여러개 조건을 and로 설정 할 수 있습니다.

<br>

### not
not을 사용하면 JSON Schema를 통해 검증되지 않은 데이터만 찾을 수 있습니다.
~~~json
{
    "not": {
      "type": "string"
    }
}
~~~
이 형식을 사용하면 모든 문자열은 유효성 검증을 통과 할 수 없습니다.
