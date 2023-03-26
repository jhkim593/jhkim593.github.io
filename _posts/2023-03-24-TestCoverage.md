---
layout: post
title: "테스트 코드 (4) - 코드 커버리지"
author: "jhkim593"
tags: TEST

---
테스트 케이스가 얼마나 충족되었는지를 나태내는 지표이며 테스트 진행 시 코드 자체가 얼마나 실행되었는지 수치를 통해 확인 가능하다.



왜 사용해야 하는가?

테스트 코드는 모든 상황에 대해서 작성되어야 하는데 테스트 코드로 작성되지 않아 커버 하지 못하는 케이스가 생기는 등 휴먼에러가 발생할수 있기 때문에 이 것을 방지하고자 코드 커버리지를 사용한다.

코드 커버리지를 사용해 테스트에서 놓친 부분들을 확인 후 보완할 수 있다.



코드 커버리지 측정 기준

라인 커버리지 : 코드 한 라인이 한번 이상 실행되면 충족
조건 커버리지 : 조건식 내부 조건이 true / false를 가지게 되면 충족
if (x > 0 && y < 0) 일때 테스트케이스 x = 1, y = 1, x = -1, y = -1 이면 x ,y 모두 내부 조건에 true false를 만족하기 때문에 해당 커버리지 충족
브랜치 커버리지 : 조건식이 true , false를 가지면 충족
if (x > 0 && y < 0) x = 1, y = -1, x = -1, y = 1 일때 조건식 자체가 true/false를 만족하기 때문에 해당 커버리지 충족


이 세가지 중 라인 커버리지가 대표적으로 가장 많이 사용됨
라인 커버리지 같은 경우 조건문 내부 조건에 대한 모든 시나리오에 대해서 테스트를 진행했다고 보장할 수는 없지만 코드 실행이 문제 없다는것을 보장할 수있음



jacoco
자바 코드 커버리지 분석 도구로서  

빌드시 설정한 커버리지 기준을 만족하는지 확인할 수 있어 적용함
추가로 코드 커버리지 결과를 파일 형태로 저장 가능


플러그인 설정
~~~groovy
plugins {
    id 'jacoco'
}
~~~

jacocoTestCoverageVerification Task 설정

: 원하는 코드 커버리지 , 최소 코드 커버리지 수준을 설정할 수 있고 이를 통과 하지 못할시 task는 실패하게됨

~~~groovy
jacocoTestCoverageVerification {
    violationRules {
        rule {
            enable = true
            element = 'CLASS'
            includes = ['com.sparrow.sep.im.backend.service.*']  

            limit {
                counter = 'LINE'
                value = 'COVEREDRATIO'
                minimum = 0.30
            }

            // excludes = []
        }

        rule {
            ...
        }
    }
}

test {
    useJUnitPlatform()
    //테스트 다음 jacoco task 실행시키면됨
    finalizedBy 'jacocoTestCoverageVerification'
}
~~~


includes : 적용 범위

elemets :  커버리지 체크 기준

  BUNDLE : 패키지 번들(프로젝트 모든 파일을 합친 것)

  CLASS : 클래스

  GROUP : 논리적 번들 그룹

  METHOD : 메서드

  PACKAGE : 패키지

  SOURCEFILE : 소스 파일

counter : 커버리지 측정 최소 단위

  BRANCH : 조건문 등의 분기 수

  CLASS : 클래스 수, 내부 메서드가 한 번이라도 실행된다면 실행된 것으로 간주

  METHOD : 메서드 수, 메서드가 한 번이라도 실행된다면 실행된 것으로 간주한다.

  LINE : 빈 줄을 제외한 실제 코드의 라인 수, 라인이 한 번이라도 실행되면 실행된 것으로 간주

value : 커버러지 측정 방식

  COVEREDCOUNT : 커버된 개수

  COVEREDRATIO : 커버된 비율, 0부터 1사이의 숫자로 1이 100%

  MISSEDCOUNT : 커버되지 않은 개수

  MISSEDRATIO : 커버되지 않은 비율, 0부터 1사이의 숫자로 1이 100%

  TOTALCOUNT : 전체 개수

  minimum : limit 통해 지정 counter값을 value에 맞게 표현 했을 때 최솟값
