---
layout: post
title: "Spring Cloud (2) - Eureka , API Gateway "
author: "jhkim593"
tags: Spring

---

이번장에서는 Service Discovery 역할을 하는 Netflix Eureka , Spring Cloud Gateway를 이용 MSA 구성을 진행할 것이다.

<br>

### Eureka 서버 프로젝트

<br>

##### build.gradle
~~~gradle
dependencies {
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-server'
}
~~~
Eureka 서버 의존성을 추가 했으며 Application class에 '@EnableEurekaServer'를 추가해 Eureka 서버임을 명시했다.

<br>

~~~java
@SpringBootApplication
@EnableEurekaServer
public class DiscoveryserviceApplication {

    public static void main(String[] args) {
        SpringApplication.run(DiscoveryserviceApplication.class, args);
    }

}
~~~

<br>

##### application.yml

~~~yml
server:
  port: 8761

spring:
  application:
    name: discoveryservice

## eureka clinet 설정
eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
~~~
server port 8761로 설정했고 , 각 마이크로 서비스에 고유 값 설정을 위해 spring.application.name 값을 설정했다.
다음 eureka client 설정을 해줬는데 eureka 라이브러리가 추가된채로 서버가 구동되면 기본적으로 eureka client 로서 어딘가 등록하는 작업을 하게되고 아래 설정들은 기본값이 true이기 때문에 현재 자신의 정보를 자신에게 등록하는 현상이 된다. 이것은 의미없는 작업이므로 false로 설정 해주었다.

<br>

### FIRST-SERVICE 프로젝트

<br>

##### build.gradle
~~~gralde
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
}
~~~
Spring Web , eureka client 의존성을 추가했다.

<br>

##### application.yml

~~~yml
server:
  port: 0

spring:
#  main:
#    web-application-type: reactive
  application:
    name: first-service

eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:8761/eureka
  instance:
    instance-id: ${spring.application.name}:${spring.application.instance_id:${random.value}}
~~~
- 매번 새로운 랜덤 포트를 부여하기 위해 server.port를 0으로 설정했다.
- eureka.client  : eureka 서버에 등록하기 위해 eureka 서버 url 설정
- instance-id : server port를 위와같이 0으로 설정하게 하고 같은 프로젝트를 여러개 띄우게 되면 포트번호가 0으로 같기 때문에 Eureka 서버에 하나의 인스턴스만 표시되는 현상이 발생한다. 이것을 해결하기 위해 instance-id를 변경하도록 하기위한 설정이다.

<br>

##### Controller

~~~java
@RestController
@Slf4j
public class FirstServiceController {

    Environment env;

    @Autowired
    public FirstServiceController(Environment env) {
        this.env = env;
    }

    @GetMapping("/check")
    public String check(HttpServletRequest request) {
        log.info("Server port = {}", request.getServerPort());
        return String.format("Hi, there. This is a message from First Service on PORT %s"
                , env.getProperty("local.server.port"));
    }
}
~~~
요청이 정상적으로 오는지 확인하기 위한 간단한 컨트롤러를 작성했다.

<br>

Eureka 서버 의존성을 추가 했으며 Application class에 '@EnableDiscoveryClient'를 추가해 Eureka 서버에 등록되는 서비스임을 명시했다.

~~~java
@SpringBootApplication
@EnableEurekaClient
public class FirstserviceApplication {

    public static void main(String[] args) {
        SpringApplication.run(FirstserviceApplication.class, args);
    }

}
~~~


### API Gateway 프로젝트

<br>

##### build.gradle

~~~gradle
dependencies {
        implementation 'org.springframework.cloud:spring-cloud-starter-gateway'
        implementation 'org.springframework.cloud:spring-cloud-starter-netflix-eureka-client'
}
~~~
build.gradle에 Spring Cloud Gateway 와 eureka clinet 의존성 설정을 추가했다.

<br>

##### application.yml
~~~yml
server:
  port: 8000

eureka:
  client:
    fetch-registry: true
    register-with-eureka: true
    service-url:
      defaultZone: http://127.0.0.1:8761/eureka


spring:
  application:
    name: apigateway-service
  cloud:
    gateway:
      default-filters:
        - name: GlobalFilter
          args:
            baseMessage: Spring Cloud Gateway Global Filter
            preLogger: true
            postLogger: true
      routes:
        - id: first-service
          uri: lb://FIRST-SERVICE
          predicates:
            - Path=/first-service/**
          filters:
            - RewritePath=/first-service/(?<segment>.*), /$\{segment}
            - name: LoggingFilter
              args:
                baseMessage: Hi
                preLogger: true
                postLogger: true
~~~

- default-filters : GatewayService에서 해당 서비스까지 요청을 전달하기 이전에 거칠 기본 필터를 설정할 수 있다.
- predicates : API Gateway로 요청이 들어왔을때 요청 url 조건을 설정한다.
- uri :
  - predicates조건에 매칭되는 url을 어디로 라우팅할 지 명시한다.
  - lb://는 eureka 서버에 등록된 서비스를 나타내며 application.name으로 설정할 수 있다.
  - lb는 load balancer의 약자로 해당 서비스가가 여러 개 띄워져 있다면 트래픽을 분산한다.
- filters
    - RewritePath : predicates가 /first-service/** 이므로 실제 USER-SERVICE로 요청시 localhost:${first-service.port}/first-service/** 형태로 요청된다. 따라서 앞에 /first-service 부분을 제거해서 요청하기 위한 rewrite 하기위한 설정이다.
    - AbstractGatewayFilterFactory를 상속한 Filter를 직접 구현해 적용할 수도 있다. 필터에 대한 내용은 바로 아래 적어두었다.

<br>

### Filter 추가

<br>

##### GlobalFilter
~~~java
@Component
@Slf4j
public class GlobalFilter extends AbstractGatewayFilterFactory<GlobalFilter.Config> {

    public GlobalFilter() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        return (exchange, chain) -> {
            ServerHttpRequest request = exchange.getRequest();
            ServerHttpResponse response = exchange.getResponse();
            log.info("Global filter baseMessage -> {}", config.getBaseMessage());

            if (config.isPreLogger()) {
                log.info("Global Filter Start:request id -> {}", request.getId());
            }

            //Custom Post Filter
            return chain.filter(exchange).then(Mono.fromRunnable(() -> {
                if (config.isPostLogger()) {
                    log.info("Global Filter End: response code -> {}", response.getStatusCode());
                }
            }));
        };
    }

    @Data
    public static class Config {
        private String baseMessage;
        private boolean preLogger;
        private boolean postLogger;
    }
}
~~~  
API Gateway application.yml 설정에 default-filters로 등록한 GlobalFilter로 모든 요청은 해당 필터를 거치게 된다.
- Config class
  - application.yml args에 입력한 값들을 주입받는 class이다.
  - 생성자에서 해당 클래스를 super의 인자로 넣어주면 apply 메서드의 인자로 들어오는 config값으로 동일하게 사용할 수 있다.
- apply 메소드
  - exchange에서 request와 response를 가져올 수 있다.
  - eureka 라이브러리가 추가된 경우 내장 서버로 톰캣이 아닌 비동기 방식 netty로 서버가 올라간다.
  - 따라서 webFlux를 사용하기 때문에 HttpServletRequest , HttpServletResponse 사용 할수 없고 ServerHttpRequest , ServerHttpResponse를 사용한다.

<br>

##### LoggingFilter

~~~java
@Component
@Slf4j
public class LoggingFilter extends AbstractGatewayFilterFactory<LoggingFilter.Config> {

    public LoggingFilter() {
        super(Config.class);
    }

    @Override
    public GatewayFilter apply(Config config) {
        GatewayFilter filter = new OrderedGatewayFilter((exchange, chain) -> {
            ServerHttpRequest request = exchange.getRequest();
            ServerHttpResponse response = exchange.getResponse();

            log.info("Logging Filter baseMessage: {}", config.getBaseMessage());
            if (config.isPreLogger()) {
                log.info("Logging PRE Filter: request id -> {}", request.getId());
            }
            return chain.filter(exchange).then(Mono.fromRunnable(() -> {
                if (config.isPostLogger()) {
                    log.info("Logging POST Filter: response code -> {}", response.getStatusCode());
                }
            }));
        }, Ordered.LOWEST_PRECEDENCE);
        return filter;
    }

    @Data
    public static class Config {
        private String baseMessage;
        private boolean preLogger;
        private boolean postLogger;
    }
}
~~~
- Ordered.LOWEST_PRECEDENCE : 적용된 필터중 가장 나중 적용

<br>

### TEST

Eureka 서버 , Firtst-Service , API Gateway를 모두 실행시켜 설정이 제대로 되었는지 테스트 해보았다.
http://localhost:8000/first-service/check로 요청을 보내게되면

<br>

<img src="https://user-images.githubusercontent.com/53510936/202906906-ff6fa729-11c9-4e7b-bc77-588e2a62bf1d.png"  width="800" height="50"/>
firtst-service에 정상적으로 요청이 들어오는 것을 확인할 수있었고,

<img src="https://user-images.githubusercontent.com/53510936/202906988-6991fddf-37fb-4318-972f-47d79672404e.png"  width="800" height="100"/>

API Gateway에서 설정한 필터가 올바르게 적용된 것을 확인할 수있었다.
