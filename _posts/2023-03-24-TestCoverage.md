---
layout: post
title: "테스트 코드 (4) - 코드 커버리지"
author: "jhkim593"
tags: TEST

---
이번장에서는 코드 커버리지와 코드 커버리지 측정 도구인 jacoco에 대해서 알아보겠습니다.

<br>

### 코드 커버리지란?
테스트 케이스가 얼마나 충족되었는지를 나태내는 지표이며 테스트 진행 시 코드 자체가 얼마나 실행되었는지 수치를 통해 확인 가능합니다.

<br>
### 왜 사용해야 하는가?

테스트 코드는 모든 상황에 대해서 작성이 되도록 해야하는데 개발자가 직접 짜는 것이다 보니 커버 하지 못하는 케이스가 생길 수 있습니다.

이런 휴먼에러를 방지하고자 코드 커버리지를 사용해 테스트에서 놓친 부분들을 확인 후 보완할 수 있습니다.

<br>

### 코드 커버리지 측정 기준

- 라인 커버리지 : 코드 한 라인이 한번 이상 실행되면 충족
- 조건 커버리지 : 조건식 내부 조건이 true / false를 가지게 되면 충족
  - if (x > 0 && y < 0) 일때 테스트 케이스가 (x = 1 , y = 1), (x = -1 ,y = -1) 이면 x>0 , y<0 내부 조건에 모두 true false를 만족하기 때문에 해당 커버리지 충족
  - 이 경우 조건 커버리지를 만족하지만 조건식 자체는 항상 false를 반환하기 때문에 true 케이스를 검증하지는 못함
- 브랜치 커버리지 : 조건식 내부가 아닌 조건식이 true , false를 가지면 충족
  - if (x > 0 && y < 0) 일때 테스트 케이스가 (x = 1, y = -1), (x = -1, y = 1) 일때 조건식 자체가 true/false를 만족하기 때문에 해당 커버리지 충족


이 세가지 중 라인 커버리지가 대표적으로 가장 많이 사용됩니다.
라인 커버리지 같은 경우 조건문 내부 조건에 대한 모든 시나리오에 대해서 테스트를 진행했다고 보장할 수는 없지만 코드 실행이 문제 없다는것을 보장할 수있습니다.

<br>

### jacoco
자바 코드 커버리지 분석 도구로서 빌드시 설정한 커버리지 기준을 만족하는지 확인할 수 있고 , 추가로 코드 커버리지 결과를 파일 형태로 저장 가능


#### 플러그인 설정
~~~groovy
plugins {
    id 'jacoco'
}
~~~

<br>
#### jacocoTestReport task 설정
: 커버리지 결과를 리포트 형식으로 저장하기 위한 task입니다.
~~~groovy
jacocoTestReport {
    reports {
        // 원하는 리포트 형식 설정
        html.enabled true
        xml.enabled false
        csv.enabled false

        //  각 리포트 타입 마다 리포트 저장 경로를 설정
        html.destination file("$buildDir/appServerCoverageReport")
        //  xml.destination file("$buildDir/jacoco.xml")
    }
}
~~~
위와 같이 설정하면 빌드 경로 / appServerCoverageReport에 커버리지 결과 리포트가 생성됩니다.


<br>
#### jacocoTestCoverageVerification Task 설정

: 원하는 코드 커버리지 , 최소 코드 커버리지 수준을 설정할 수 있고 이를 통과 하지 못할시 task는 실패하게됩니다.

~~~groovy
jacocoTestCoverageVerification {
    violationRules {
        rule {
            element = 'METHOD'
            includes = ['com.example.jacoco.*]
            limit {
                    counter = 'BRANCH'
                    value = 'COVEREDRATIO'
                    minimum = 0.00
            }
            limit{
                    counter = 'LINE'
                    value = 'COVEREDRATIO'
                    minimum = 0.00
            }
        }
    }
}
~~~


- includes : 적용 범위

- elemets :  커버리지 체크 기준

  - BUNDLE : 패키지 번들(프로젝트 모든 파일을 합친 것)

  - CLASS : 클래스

  - GROUP : 논리적 번들 그룹

  - METHOD : 메서드

  - PACKAGE : 패키지

  - SOURCEFILE : 소스 파일

- counter : 커버리지 측정 최소 단위

  - BRANCH : 조건문 등의 분기 수

  - CLASS : 클래스 수, 내부 메서드가 한 번이라도 실행된다면 실행된 것으로 간주

  - METHOD : 메서드 수, 메서드가 한 번이라도 실행된다면 실행된 것으로 간주한다.

  - LINE : 빈 줄을 제외한 실제 코드의 라인 수, 라인이 한 번이라도 실행되면 실행된 것으로 간주

- value : 커버러지 측정 방식

  - COVEREDCOUNT : 커버된 개수

  - COVEREDRATIO : 커버된 비율, 0부터 1사이의 숫자로 1이 100%

  - MISSEDCOUNT : 커버되지 않은 개수

  - MISSEDRATIO : 커버되지 않은 비율, 0부터 1사이의 숫자로 1이 100%

  - TOTALCOUNT : 전체 개수

  - minimum : limit 통해 지정 counter값을 value에 맞게 표현 했을 때 최솟값


<br>

#### test 시 jacoco task를 실행시키기 위한 설정
jacocoTestCoverageVerification가 jacocoTestReport보다 먼저 실행되게 된다면 커비리지를 통과하지 못할시 빌드는 멈추게 되고 jacocoTestReport는 실행되지 않아 리포트는 최신화가 되지 않는 문제가 생길 수 있습니다.

그렇기 때문에 test → jacocoTestReport → jacocoTestCoverageVerification 순서로 실행 순서를 지정합니다.

~~~groovy
test {
   ...

  //test 종료시 jacocoTestReport 실행
  finalizedBy 'jacocoTestReport'
}

jacocoTestReport {
  ...

  //jacocoTestReport 실행 후 jacocoTestCoverageVerification 실행
  finalizedBy 'jacocoTestCoverageVerification'
}
~~~

<br>

#### 제외 설정
프로젝트에 적용하려고 보니 lombock `@Setter` 어노테이션으로 생성된 set메소드가 포함되어 있는 것을 볼수 있습니다.
<img src="https://user-images.githubusercontent.com/53510936/230903869-8ac56c56-cb47-4e35-a2ec-de3da88581ff.png"  width="650" height="200"/>

이 부분은 커버리지 측정이 필요없기 때문에 Root 프로젝트 경로의 lombock.config 파일을 추가해서 해결할 수 있습니다.

<br>
lombock.config
~~~config
lombok.addLombokGeneratedAnnotation = true
~~~
위와 같이 파일을 생성하게 되면 lombock 관련 코드에 generated 주석이 추가되어 jacoco에서 커버리지 측정시 자동으로 제외하게 됩니다.
