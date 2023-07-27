---
layout: post
title: "Spring Cloud (1) "
author: "jhkim593"
tags: Spring

---

## Spring Cloud
Spring Cloud는 마이크로 서비스 개발 , 배포 운영에 필요한 아키텍처를 쉽게 구성할 수 있도록 지원하는 Spring Boot기반의 Framework이다.
Spring Cloud는 Spring Boot와 함께 사용하게 되며 MSA를 위한 환경 설정 , 서비스 검색 , 라우팅 등 분산 시스템을 빠르게 설정할 수 있다.

Spring Cloud가 제공하는 다양한 기능 몇가지를 간단하게 알아보고 ,
자세한 내용은 다음 챕터에서 다룰 예정이다.

### Spring Cloud API Gateway
만약 Client side에서 각 서비스들을 연결하는 엔드포인트를 가지고 있다고 한다면 엔드포인트가 추가 , 수정 될 떄마다 클라이언트도 다시 업데이트 해줘야하는 불편함이 생긴다. 다음 문제를 해결하기 위해 API Gateway를 두고 마이크로 서비스 요청정보를 일괄적으로 처리한다.

<img src="https://user-images.githubusercontent.com/53510936/202893107-52c4e046-1dda-465c-94af-57df6bed7f36.png"  width="800" height="400"/>

<br>

### Spring Cloud Config server
Spring Cloud Config server는 분산된 환경에서 여러 서비스 환경 설정을 독립적으로 관리할 수 있는 기능을 제공합니다.
Config server를 사용하게 되면 서비스의 환경 설정을 따로 배포 하지 않아도 되고 , 모든 서비스에 공통된 환경 설정을 일일이 배포할 필요가 없습니다.
즉 실행 중에 설정값 변경이 필요해지면, 설정 서버만 변경하고 애플리케이션은 갱신하도록 해주면 된다.

<br>

<img src="https://user-images.githubusercontent.com/53510936/202890438-add9eedc-f574-4fb8-90ac-7d0066114d6b.png"  width="800" height="400"/>

위 이미지는 Spring Cloud Config의 기본 구조이며 동작 순서는 다음과 같다.

- 클라이언트는 서버로 설정값을 요청하면 서버는 설정 파일이 위치한 Git 저장소에 접근한다.
- 서버는 Git 저장소로부터 최신 설정값을 받고 클라이언트는 다시 서버를 통해 최신 설정값을 받는다. 만일, 사용자가 설정 파일을 변경하고 Git 저장소를 업데이트했다면 애플리케이션(클라이언트)의 actuator/refresh 엔드 포인트를 호출하여 설정값을 변경하도록 한다.
- 여기서의 설정값 갱신을 위한 엔드 포인트 호출은 각각의 클라이언트를 호출해야 하며 호출된 클라이언트만 반영된다.


### Service Discovery
보통의 경우 한 서비스에서 다른 서비스를 호출할 때 IP와 Port 정보를 이용해 특정 서비스를 식별해 요청을 보낸다. 하지만 클라우드 환경에서 IP와 Port 정보가 Auto-scaling 등으로 인해 동적으로 변경된다. 이런 상황에서 서비스를 식별하는 다른 방법이 필요한데 Service Discovery가 특정 서비스를 호출하고자할 때 위치와 상태를 확인해 식별 가능하도록 기능을 제공한다.

Service Discovery를 기능을 구현하는 방식에는 크게 두가지 방식이 있다.

<br>

1. Client Side Discovery
<img src="https://user-images.githubusercontent.com/53510936/202892202-9b0c8ce9-d18f-4487-9c0b-d70f3ca425ad.png"  width="800" height="500"/>
이 방식은  Client Service가 Service Registry에 서비스의 위치를 찾아서 호출하는 방식이다. Service 인스턴스의 네트워크 위치는 생성될 때 Service Registry에 등록되고 , service instance가 종료될 때 service registry에서 삭제된다.


2. Server Side Discovery
<img src="https://user-images.githubusercontent.com/53510936/202892513-41563acc-fef7-4c8e-bef7-cfa3af6ecbb5.png"  width="800" height="500"/>
이 방식은 요청을 로드 밸런서를 통해서 Service로 전달하는 방식이다. 로드 밸런서는 Service registry로 query를 날리고 각각 요청을 가용할 수 있는 서비스 인스턴스로 라우팅한다. client-side discovery처럼 서비스 인스턴스가 service registry에 생성되고 삭제된다.

Spring Cloud Netflix Eureka가 Service Discovery의 역할을 제공하며 자세한 내용은 다음 챕터에 다룰 예정이다.
