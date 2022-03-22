---
layout: post
title: "Quartz Job Scheduler (1) "
author: "jhkim593"
tags: Spring

---

## Quartz
Quartz는 Terracotta 라는 회사에 의해 개발된 Job Scheduling 라이브러리입니다. 완전히 자바로 개발되어 어느 자바 프로그램에서도 쉽게 통합해서 개발할 수 있습니다. 또한 Quartz는 수십에서 수천 개의 작업도 실행 가능하며 간단한 interval 형식이나 Cron 표현식으로 복잡한 스케줄링도 지원하며 Spring 프레임워크와 함께 사용될 수 있습니다.


<img src="https://user-images.githubusercontent.com/53510936/159417953-9838344a-cfc6-45a8-9b82-cb069a5a329d.png"  width="800" height="700"/>

## Quartz 기본 구성
- Job : 스케줄링할 실제 작업을 가지는 객체 ,실제 수행해야하는 작업을 execute 메소드에서 구현합니다. Job의 Trigger가 발생하면 스케줄러는  JobExecutionContext 객체를 넘기고 execute 메소드를 호출하며, 여기서 JobExecutionContext는 Scheduler, Trigger, JobDetail 등을 포함하여 Job에 대한 정보를 제공합니다.
- JobDetail : Job의 정보를 구성하는 객체 , Trigger가 Job을 수행 시에 사용합니다.
- Trigger : Job이 언제 시행시킬지 조건을 담고 있습니다.
- JobData : Job에서 사용할 데이터를 전달하는 객체 ,  JobDetail 생성시 세팅해주면 됩니다.
- Scheduler : JobDetail 과 Trigger 정보를 이용해서 Job을 시스템에 등록하고 , Trigger가 동작하면 지정된 Job을 실행시키는 객체
- SchedulerFactory : Scheduler 인스턴스를 생성하는 역할을 하는 객체
- JobStore : 스케줄러에 등록된 Job의 정보와 실행이력이 저장되는 공간, 기본적으로 RAM에 저장되어 JVM 메모리공간에서 관리되지만, 원한다면 다른 RDB에서 관리할 수 있다.
- Listener Scheduler의 이벤트를 받으며 Job 실행 전 후로 이벤트를 받는 JobListener와 Trigger 이벤트를 받는 TriggerListener 가 있습니다.
