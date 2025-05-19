---
layout: post
title: "redis lua script를 활용한 동시성 처리 로직 구현"
author: "jhkim593"
tags: redis
---

서비스의 대규모 트래픽 처리를 위해서는 여러 서버로 구성된 분산 시스템 환경을 구축하는 것이 일반적입니다. 하지만 분산환경에서는 동시성 제어가 복잡해지는 문제가 있습니다. 이번 글에서는 선착순 쿠폰 발금 시스템을 예시로 분산 시스템 환경에서 redis를 활용한 동시성 처리 방법에 대해 살펴보겠습니다.

<br>

## 기존 동시성 제어 방식
동시성 제어를 위한 방법에는 여러 가지가 있습니다.

#### **Java 단에서 처리**
`synchronized`, `java.util.concurrent` 를 사용합니다. 하지만 이 방식은 단일 서버에서만 유효하고 분산환경에서는 사용할 수 없습니다.

#### **데이터베이스 레벨의 락(Lock)**
데이터 정합성이 보장되지만 트랜잭션 처리 및 디스크 접근으로 인한 성능 저하와 데드락이 발생할 수 있습니다.

<br>

## 동시성 처리에 redis가 사용되는 이유
동시성 처리에 redis가 사용되는 이유는 대표적으로 두가지가 있습니다.

#### **싱글 스레드 기반**
여러 클라이언트 동시에 요청해도 순차적으로 처리돼서 작업의 원자성이 보장됩니다.

#### **in memory 기반**
메모리에 저장하고 처리하는 구조이기 때문에 디스크 기반 저장소에 비해 읽기 / 쓰기 속도가 빠릅니다.

<br>
추가 인프라가 필요하고 시스템 복잡성이 증가하는 단점이 있을 수 있지만 위와 같은 이유로 분산 환경에서 동시성 처리에 redis가 많이 활용되고 있습니다.

<br>

## 요구사항
이제 아래 요구 사항을 기반으로 선착순 쿠폰 발급 로직을 구현해보도록 하겠습니다. 요구사항은 다음과 같습니다.
- 선착순 쿠폰이 3000개 제한이며 그 이상 발급 될 수 없음
- 사용자당 쿠폰 한개씩만 발급 받을 수 있음

위 요구사항을 만족하기 위해 redis Set 자료형을 사용해

redis에 쿠폰 잔여 수량 및 발급 받은 사용자 ID를 저장했고 발급 요청이 왔을 때 이미 발급 받은 사용자인지 확인하고 이미 발급 받은 사용자면 추가로 쿠폰을 발급하지 않고 발급 받지 않은 사용자면 새 쿠폰을 발급하고 쿠폰 잔여 수량을 1만큼 감소합니다. 즉 사용자 요청이 왔을 때 쿠폰 발급 가능한지 확인 , 쿠폰 발급 두 작업이 이루어 져야합니다.

## 동시성 처리시 문제 상황

하지만 redis를 사용한다고 해서 동시성 문제가 해결되는 것은 아닌데 만약 두 사용자가 동시에 쿠폰 발급 요청을 했을 경우를 살펴보겠습니다.

- ( 사용자 A ) 쿠폰 잔여 수량 및 중복 발급 조회 : 잔여 수량 1
- ( 사용자 B ) 쿠폰 잔여 수량 및 중복 발급 조회 : 잔여 수량 1
- ( 사용자 A ) 쿠폰 발급 완료 : 잔여 수량 0
- ( 사용자 B ) 쿠폰 발급 완료 : 잔여 수량 -1

잔여 수량 및 중복 발급 조회 , 쿠폰 발급 명령이 순차적으로 수행됐지만 사용자 B가 업데이트 되지않은 쿠폰 데이터를 조회함으로써 잔여 수량이 -1이 된 것을 볼 수 있습니다. 해당 문제를 해결하기 위해서는 아래와 같이 잔여 수량 및 중복 발급 조회 , 쿠폰 발급 명령을 묶어서 원자적으로 수행되도록 해야합니다.

- 사용자 A
    - 쿠폰 잔여 수량 및 중복 발급 조회 : 잔여 수량 1
    - 쿠폰 발급 완료 : 잔여 수량 0
- 사용자 B
    - 쿠폰 잔여 수량 및 중복 발급 조회 : 잔여 수량 0
    - 쿠폰 발급 실패

잔여 수량 및 중복 발급 조회, 쿠폰 발급 명령을 묶어서 처리하게되면 여러 사용자가 동시에 요청해도 쿠폰 수량의 동시성이 보장할 수 있습니다. redis에서 여러 명령을 하나로 묶어서 처리하는 방법에는 대표적으로 redis transaction , lua script가 있습니다.

**redis Transaction**

여러 명령어를 원자적으로 실행 할 수 있는 방법에는 redis transaction 이 있습니다. 기본 동작은 다음과 같습니다

- MULTI 명령: 트랜잭션 시작
- 명령 큐잉: MULTI 이후의 모든 명령은 실행되지 않고 큐에 적재
- EXEC 명령: 큐에 쌓인 모든 명령을 순차적으로 실행

명령들을 큐에 적재하고 EXEC 명령 후 실행하기 때문에 Transaction 내부에서 읽기 명령( GET 등 )은 실제 데이터를 읽지 않고 큐에 쌓기만 하기 때문에 올바른 값이 반환되지 않습니다.

```
> SET key value1
OK
> MULTI
OK
> GET key
QUEUED
> SET key value2
QUEUED
> GET key
QUEUED
> EXEC
1) "value1"
2) OK
3) "value2"
```

redis에 실제로 커맨드를 날려보면 transaction내부에서는 읽기 명령이어도 전부 QUEUED를 반환하는것을 알수 있습니다. 선착순 쿠폰 시스템에서는 **transaction 내부**에서 잔여 수량 및 중복 발급 확인을 위한 조회가 필요하기 때문에 해당 방식은 적절하지 못했습니다.

**redis lua script**

lua script는 redis에서 l**ua 언어를 사용해 작성된 스크립트**를 실행하는 기능입니다.

여러 명령어를 lua 스크립트로 묶어 실행하면, **중간에 다른 클라이언트가 개입할 수 없는 원자적 실행**이 가능하기 때문에 해당 방식을 선택했으며 자세한 구현은 아래에서 다루도록 하겠습니다.

## 선착순 쿠폰 발급 로직 구현 코드

redis lua script를 사용한 쿠폰 발급 처리 로직 및 테스트 코드를 구현했습니다. RDB , MQ 등 다른 시스템과의 연동 코드는 이번글에서 다루지 않았습니다.

CouponRepository.java

```java
@Component
@RequiredArgsConstructor
public class CouponRepository {
    private final redisTemplate redisTemplate;
    private static final String couponKey = "coupon";
    private static final String couponLimit = "3000";

    public Boolean createCoupon(String userId){
        return (Boolean) redisTemplate.execute(getCountAndAddValue(), Collections.singletonList(couponKey), userId, couponLimit);
    }
    public Long size(){
        return redisTemplate.opsForSet().size(couponKey);
    }
    public Boolean delete(){
        return redisTemplate.delete(couponKey);
    }

    public redisScript<Boolean> getCountAndAddValue() {
        String script =
                "local key = KEYS[1] \n" +
                "local value = ARGV[1]\n" +
                "local limit = tonumber(ARGV[2])\n" +
                "local size = redis.call('SCARD', key)\n" +
                "\n" +
                "if size < limit then\n" +
                "    local result = redis.call('SADD', key, value)\n" +
                "    if result == 1 then\n" +
                "        return 1 \n" +
                "    else\n" +
                "        return 0\n" +
                "    end\n" +
                "else\n" +
                "    return 0\n" +
                "end\n";
        return redisScript.of(script, Boolean.class);
    }
}

```

1. 중복 발급과 수량을 모두 확인하기 위해 redis Set 자료형을 사용했습니다. 위 예시에서는 key는 coupon , 제한 쿠폰 개수는 3000개로 설정했습니다.
2. 먼저 `redis.call('SCARD', key)`를 통해 먼저 쿠폰이 제한 개수 이상 발급됐는지 확인합니다. 이 조건에 통과되지못하면 스크립트 결과로 0을 반환합니다.
3.  `redis.call('SADD', key, value)` 를 통해 Set에 value를 저장합니다. 중복되는 값이 없어 저장에 성공한다면 스크립트 결과로 1을 반환합니다. 만약 중복되는 값이 있어 저장되지 못하면 스크립트 결과로 1을 반환합니다
4. 스크립트 결과가 1이면 `getCountAndAddValue` 메소드는 true를 반환하고 0이면 false를 반환합니다.

이번 예시에서는 생략됐지만 redis에 저장되는 데이터는 휘발성이기 때문에 `createCoupon` 메소드의 리턴값에 따라 발급된 쿠폰 데이터를 영구 DB에 저장하는 로직이 필수적입니다.

CouponTest.java

```java
@SpringBootTest
public class CouponTest {
    @Autowired
    redisTemplate redisTemplate;
    @Autowired
    CouponRepository couponRepository;

    @AfterEach
    public void clear(){
        couponRepository.delete();
    }

    @Test
    public void couponTest() throws InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(100);
        CountDownLatch count = new CountDownLatch(300000);
        for (int i = 0; i < 300000; i++) {
            executorService.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        int randomNum = ThreadLocalRandom.current().nextInt(1, 30000);
                        couponRepository.createCoupon(String.valueOf(randomNum));
                    } finally {
                        count.countDown();
                    }
                }
            });
        }
        count.await();
        assertThat(couponRepository.size()).isEqualTo(3000);
    }
}
```

- `ThreadLocalRandom.current().nextInt(1, 30000)` 로 3만명의 유저를 랜덤하게 설정했습니다.
- `createCoupon` 메소드를 약 300000번 호출합니다.
- `count.await()` 를 통해 모든 처리가 완료될 때까지 대기합니다.
- `assertThat(couponRepository.size()).isEqualTo(3000)` 로 redis Set의 size를 확인해 쿠폰 제한 개수인 3천개까지만 저장됐는지 확인합니다.

테스트시 성공하는 것을 확인 할 수 있었습니다.

![image.png](attachment:5a5a3595-72ac-4c2a-b0db-250fcbf2719f:image.png)

각각의 방법에는 장단점이 있지만 현재 저희 요구에는 Scale Out 하는 것을 전제로 하였으니

여러 WAS를 병렬적 수행할 수 있게 하는 시스템이므로 Java 에서 제공하는 Locking 방법들은 사용하지 못할 것 같습니다.

또한 DB Level 에서 지원하는 Locking 기법도 하나의 DB 에서 큰 트래픽을 받아내야 하고

Locking 하며 데드락의 위험이 있으니 SPOF 의 위험으로 사용이 어렵습니다.

그렇다면 redis 진영의 Locking 을 위한 라이브러리인 redisson 이 남아있는데요.

redis 는 Single Thread 로 동작하며 하나의 요청에 Atomic한 수행을 보장하며

여러 WAS 에서 접근할 수 있고 DB 나 Application 의 책임을 분산할 수 있습니다.

쿠폰 발급 이력을 저장한다.

https://100100e.tistory.com/582

분산락

트랜잭션 새로파야 안정성높음

[https://velog.io/@jaepyo99/동시성-문제를-방지하기-위한-분산락과-Lua스크립트-적용기](https://velog.io/@jaepyo99/%EB%8F%99%EC%8B%9C%EC%84%B1-%EB%AC%B8%EC%A0%9C%EB%A5%BC-%EB%B0%A9%EC%A7%80%ED%95%98%EA%B8%B0-%EC%9C%84%ED%95%9C-%EB%B6%84%EC%82%B0%EB%9D%BD%EA%B3%BC-Lua%EC%8A%A4%ED%81%AC%EB%A6%BD%ED%8A%B8-%EC%A0%81%EC%9A%A9%EA%B8%B0)