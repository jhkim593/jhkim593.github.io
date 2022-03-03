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


### Schema 재사용
definitions 키워드는 스키마를 정의하고 $ref를 사용해 정의한 스키마를 재사용 할 수있습니다.
