---
layout: post
title: "Spring Cloud - Config Server / Client"
author: "jhkim593"
tags: Spring

---
>관련 코드는 [github](https://github.com/jhkim593/blog_code/tree/master/spring_cloud/cloud-config)를 참고해주세요

## Spring Cloud Config Server
Spring Cloud Config Server는 여러 서비스 환경 설정을 통합해서 관리할 수 있는 서버입니다.
실행 중인 어플리케이션이 Config Server에서 설정 정보를 받아와서 갱신하는 방식이기 때문에 서비스의 설정이 바뀔때마다 빌드와 배포를 따로 할 필요가 없고 모든 서비스에서 공통으로 사용되는 설정 정보를 서비스마다 일일이 설정할 필요가 없게 됩니다.

<img src="https://github.com/jhkim593/jhkim593.github.io/assets/53510936/ce7fb0fc-24f7-479f-82b6-8003e1e7bb74"  width="500" height="300"/>

**Config Server**는 설정 정보가 저장된 Repository를 바라보게되며 서비스들은 **Config Client** 로서 **Config Server**를 통해 설정 정보를 읽어오게 됩니다.

<br>

## Spring actuator + Config Server
최초 어플리케이션 실행 시점에 Config Server로 부터 설정 정보를 읽어와서 사용하게 되는데 이후에 설정 정보들이 변경 사항이 생겨도 실행 중인 어플리케이션에 바로 반영되지 않을 것입니다.<br> 설정 정보를 변경하기 위한 가장 간단한 방법은 어플리케이션을 **재기동** 하는 것인데 설정 정보가 변경되었다고 해서 매번 어플리케이션을 **재기동** 시키는 것은 좋지 않은 방법일 것입니다. <br>그래서 **Spring actuator**를 사용하게 되는데 **Spring actuator**란 어플리케이션의 상태를 모니터링하기 위한 여러 Http 엔드포인트를 제공해주는 모듈이며 여기서 `/actuator/refresh`는 어플리키에션 **재기동** 없이 설정 정보를 갱신하도록 해줍니다.

적용 후 구조는 다음과 같습니다.
<img src="https://github.com/jhkim593/jhkim593.github.io/assets/53510936/638df004-0cea-441d-9dfc-20be940bfba3"   width="500" height="300"/>

설정 정보를 업데이트 해서 저장소에 올려놓게 되고 각 서비스 마다 `/actuator/refresh`를 호출해주게되면 변경된 설정 정보가 반영되게 됩니다.

<br>

## Spring cloud Bus , monitor
하지만 만약 서비스가 한개가 아닌 백개라고 한다면 각 서비스마다 `/actuator/refresh`를 호출해야되는 불편함이 생깁니다.
이 문제를 해결하기 위해서 **Spring Cloud Bus**, **Spring Cloud monitor**를 사용할 수 있습니다.<br>
 **Spring Cloud Bus**는 메세지 브로커를 사용해 서비스들을 연결하며 한개의 서비스에만 `/actuator/refresh`호출해도 모든 서비스로 요청이 전파되게 됩니다.
 **Spring Cloud Monitor**를 Config Server에 적용하게 되면 Config Server에 /monitor 엔드포인트가 생기게 되고 해당 엔드포인트를 통해 요청을 받게되면 **Spring Cloud Bus**로 갱신 이벤트를 전파하게됩니다.

 구조는 다음과 같습니다.

<img src="https://github.com/jhkim593/jhkim593.github.io/assets/53510936/6e042c38-434e-4f48-8667-125f3179e401"   width="500" height="300"/>

설정 정보를 변경해서 저장소에 올려놓게 되면 저장소에 설정된 WebHook으로 Config Server에 `/monitor`를 호출하게되고 Spring Cloud Bus로 갱신 이벤트가 전달됩니다. 이후 Cloud Bus를 통해 연결된 모든 서비스 들에게 `/actuator/refresh`가 호출되면서 설정 정보가 모두 갱신되게 됩니다.

<br>

## 적용
Spring actuator와 Spring Cloud Bus, monitor를 적용한
ConfigServer와 두 ConfigClinet(설정 정보를 읽어오는 user, order 서비스)들을 간단하게 구현해보도록 하겠습니다.

#### 1. 저장소에 설정 파일 저장
설정 파일을 관리하기 위한 원격 저장소를 생성하고 아래와 같이 **예시 설정 파일**을 저장했습니다.
이 위치에 저장된 파일이 Config Server를 통해 각 서비스들이 읽어서 사용하게 됩니다.
<br>
<br>
**user-dev.yml**
~~~yml
user:
  var1: user1
  var2: user2
~~~
<br>

**order-dev.yml**
~~~yml
order:
  var1: order1
  var2: order2
~~~

<br>

**common-dev.yml**
~~~yml
common:
  var: common1
~~~

<br>
#### 2. Config Server 설정

build.gradle
~~~gradle
implementation 'org.springframework.cloud:spring-cloud-config-server'
implementation "org.springframework.cloud:spring-cloud-starter-bus-kafka"
implementation 'org.springframework.cloud:spring-cloud-config-monitor'
~~~
ConfigServer 등록을 위해 **spring-cloud-config-server**을 추가,
Cloud Bus 설정을 **spring-cloud-starter-bus-kafka**을 추가,
Cloud Bus로 갱신 이벤트를 전달하기 위해 **spring-cloud-config-monitor**를 추가했습니다.

<br>

~~~java
@EnableConfigServer
@SpringBootApplication
public class ConfigApplication {

	public static void main(String[] args) {
		SpringApplication.run(ConfigApplication.class, args);
	}

}
~~~

@EnableConfigServer를 명시해서 ConfigServer로 등록하게 됩니다.

<br>
**application.yml**
~~~yml
server:
  port: 8081

spring:
  application:
    name: config
  kafka:
    bootstrap-servers: //Spring cloud bus 설정을 위한 kafka 서버 정보

  cloud:
    bus:
      enabled: true

    config:
      server:
        git:
          uri:                                              // 설정 정보 저장된 저장소  url
          default-label: main                               // 브랜치
          private-key:                                      // private repository일 경우 ssh 인증을 위한 키등록
          ignore-local-ssh-settings: true
        encrypt:
          enabled: false
~~~

private repository에 경우 private key가 필요하며 private key 같은 경우 직접 노출되면 안되기때문에 `Jasypt`와 같은 라이브러리를 사용하여 암호화할 수 있습니다.

<br>
#### 3. Config Client 설정
Cloud Bus + Monitor 설정을 톨해 이벤트가 모든 서비스에 전파되는지 확인하기위해 2개의 서비스를 Config Clinet로 등록해 테스트해보도록 하겠습니다.

<br>
#### 3-1 . User 서비스 Config Client 설정
build.gradle
~~~gradle
implementation 'org.springframework.cloud:spring-cloud-config-client'
implementation "org.springframework.cloud:spring-cloud-starter-bus-kafka"
implementation 'org.springframework.boot:spring-boot-starter-actuator'
~~~

ConfigClient 등록을 위해 **spring-cloud-config-client**을 추가,
Cloud Bus 설정을 **spring-cloud-starter-bus-kafka**을 추가,
이벤트를 전달받아 설정 정보를 전달 받기 위해서 **spring-cloud-config-monitor**를 추가했습니다.

**application.yml**
~~~yml
server:
  port: 8082
spring:
  kafka:
    bootstrap-servers: 192.168.56.101:9092                       //cloud bus설정을 위해 카프카 주소 입력
  config:
    import: optional:configserver:http://127.0.0.1:8081          //config 서버 주소
  application:                                                   //1
    name: user, common                                          
  profiles:                                                     
    active: dev

  cloud:
    bus:
      enabled: true
      refresh:
        enabled: true

management:
  endpoints:
    web:
      exposure:
        include: refresh, busrefresh                            // 2
~~~
1) application.name과 profiles.active를 통해 ConfigServer가 바라보고 있는 저장소에 설정 파일을 읽어올 수 있습니다.
위 설정 파일과 같이 설정하면 원격 저장소에 {application.name}-{profile}.yml과 일치하는 설정 값을 읽어오게됩니다.
profile default는 설정 파일 이름에서 생략할 수 있습니다.

2) busrefresh 엔드포인트를 노출하기 위해 추가하였습니다.

<br>
#### 3-2 . Order 서비스 Config Client 설정

**application.yml**
~~~yml
server:
  port: 8083
spring:
  kafka:
    bootstrap-servers: 192.168.56.101:9092
  config:
    import: optional:configserver:http://127.0.0.1:8081
  application:
    name: order, common
  profiles:
    active: dev

  cloud:
    bus:
      enabled: true
      refresh:
        enabled: true

management:
  endpoints:
    web:
      exposure:
        include: refresh, busrefresh
~~~

user 서비스 설정과 동일하며 application.name만 order로 수정해서 order-dev.yml 설정 파일을 바라보도록 설정했습니다.

<br>

#### 4. 테스트
테스트를 위해 각 서비스에 설정 값 조회를 위한 코드를 추가했습니다.

#### 4-1 설정 값 조회 테스트
~~~java
@RefreshScope
@RestController
public class UserController {

    @Value("${user.var1}")
    private String var1;

    @Value("${user.var2}")
    private String var2;

    @Value("${common.var}")
    private String commonVar1;

    @GetMapping("/test")
    public ResponseEntity test() {
        String response =
                "var1 : " + var1+ " , "+
                "var2 : " + var2 + " , " +
                "common.var : "+commonVar1;
        return new ResponseEntity(response, HttpStatus.OK);
    }
}
~~~

<br>

~~~java
@RefreshScope
@RestController
public class OrderController {

    @Value("${order.var1}")
    private String var1;

    @Value("${order.var2}")
    private String var2;

    @Value("${common.var}")
    private String commonVar1;

    @GetMapping("/test")
    public ResponseEntity test() {
        String response =
                "var1 : " + var1+ " , "+
                "var2 : " + var2 + " , " +
                "common.var : "+commonVar1;
        return new ResponseEntity(response, HttpStatus.OK);
    }
}
~~~
@RefreshScope : 사용 중인 설정 값을 갱신하기 위해 설정이 필요합니다.
ConfigServer와 각 서비스들을 실행시키고 API에 요청을 전송해서 결과를 확인해 보겠습니다.

<br>
**테스트 결과는 아래와 같습니다.**
<img src="https://github.com/jhkim593/Spring-Cloud-Repository/assets/53510936/2930cb74-aad0-465f-9cbc-d37120b29290"  width="400" height="100"/>
<img src="https://github.com/jhkim593/Spring-Cloud-Repository/assets/53510936/681eabb7-34df-408e-a2c2-c4b1739d7496"  width="400" height="100"/>

각 서비스들이 Config Server를 통해 설정 파일을 정상적으로 읽은것을 확인 할 수 있었습니다.

<br>
#### 4-2 Cloud bus , monitor 확인
만약 원격 저장소에 있는 설정 파일 값을 업데이트해서 Cloud bus, monitor 동작을 테스트해보겠습니다.<br>
Config Server에 monitor 관련 의존성을 추가하면 기본적으로 /monitor 엔드포인트가 활성화됩니다.
해당 엔드포인트에 요청을 전송해서 갱신 이벤트를 전파 할 수 있습니다.

~~~curl
curl --location --request POST 'http://localhost:8081/monitor' \
--header 'Content-Type: application/json' \
--data-raw '{
    "path":["**"]
}'
~~~

body 값에 path":["\**\"] 을 넣어 전송하게되면 모든 서비스에 갱신 이벤트를 전파할 수 있습니다.
특정 서비스에만 갱신 이벤트를 전파하기 위해서는 **path" : [ "${application.name}: ??? "]** 해당 형식으로 서비스에 어플리케이션 이름을 넣어 전송하면 됩니다.

<br>
**user-dev.yml**
~~~yml
user:
  var1: user10
  var2: user20
~~~

<br>
**order-dev.yml**
~~~yml
order:
  var1: order10
  var2: order20
~~~

저장소에 설정 파일 값을 위와 같이 업데이트 후에 /monitor 엔드포인트에 요청을 전송해보겠습니다.

<br>
/monitor 요청 결과는 다음과 같습니다.

<img src="https://github.com/jhkim593/Spring-Cloud-Repository/assets/53510936/99a38557-2f6b-493c-a83f-b6f54c400a5b"  width="500" height="100"/>
<img src="https://github.com/jhkim593/Spring-Cloud-Repository/assets/53510936/b9d1f257-99fd-4a9a-8ded-e50444f6b2b6"  width="500" height="100"/>

/monitor 엔드포인트에 요청을 전송해서 어플리케이션 재기동하지 않고 설정 정보가 정상적으로 갱신된 것을 확인 할수 있습니다.
